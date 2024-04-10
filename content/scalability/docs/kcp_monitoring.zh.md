---
日期: 2023-09-22
作者:
  - Shane Corbett
---
# 控制平面监控

## API 服务器
在查看我们的 API 服务器时,我们需要记住它的一个功能是限制入站请求,以防止控制平面过载。在 API 服务器级别看起来可能是瓶颈的情况,实际上可能是在保护它免受更严重的问题。我们需要权衡增加通过系统的请求量的利弊。要确定是否应该增加 API 服务器值,以下是我们需要注意的一些事项:

1. 通过系统的请求延迟是多少?
2. 这种延迟是 API 服务器本身造成的,还是"下游"的 etcd 造成的?
3. API 服务器队列深度是否是导致这种延迟的因素?
4. API 优先级和公平性 (APF) 队列是否为我们想要的 API 调用模式设置正确?

## 问题出在哪里?
首先,我们可以使用 API 延迟指标来了解 API 服务器为请求提供服务的时间。让我们使用下面的 PromQL 和 Grafana 热图来显示这些数据。

```
max(increase(apiserver_request_duration_seconds_bucket{subresource!="status",subresource!="token",subresource!="scale",subresource!="/healthz",subresource!="binding",subresource!="proxy",verb!="WATCH"}[$__rate_interval])) by (le)
```

!!! tip
    有关如何使用本文中使用的 API 仪表板监控 API 服务器的深入报告,请参见以下[博客](https://aws.amazon.com/blogs/containers/troubleshooting-amazon-eks-api-servers-with-prometheus/)

![API 请求持续时间热图](../images/api-request-duration.png)

这些请求都在一秒钟以内,这表明控制平面正在及时处理请求。但如果情况并非如此呢?

我们在上面使用的 API 请求持续时间格式是热图。热图格式的好处是它默认告诉我们 API 的超时值(60 秒)。但我们真正需要知道的是在达到超时阈值之前,这个值应该引起多大的关注。作为一个粗略的可接受阈值指南,我们可以使用上游 Kubernetes SLO,可以在[这里](https://github.com/kubernetes/community/blob/master/sig-scalability/slos/slos.md#steady-state-slisslos)找到。

!!! tip
    注意这个语句中的 max 函数?当使用聚合多个服务器(默认情况下 EKS 上有两个 API 服务器)的指标时,不应该将这些服务器平均在一起。

### 不对称的流量模式
如果一个 API 服务器 [pod] 负载很轻,而另一个负载很重呢?如果我们将这两个数字平均在一起,我们可能会误解正在发生的事情。例如,这里我们有三个 API 服务器,但所有负载都在其中一个 API 服务器上。作为一个规则,任何有多个服务器的东西,如 etcd 和 API 服务器,在研究扩展和性能问题时都应该被分开。

![未完成请求总数](../images/inflight-requests.png)

随着向 API 优先级和公平性的转变,系统上的总请求数只是检查 API 服务器是否过载的一个因素。由于系统现在基于一系列队列工作,我们必须查看这些队列是否已满,以及该队列的流量是否被丢弃。

让我们使用以下查询来查看这些队列:

```
max without(instance)(apiserver_flowcontrol_request_concurrency_limit{})
```

!!! note
    有关 API A&F 如何工作的更多信息,请参见以下[最佳实践指南](https://aws.github.io/aws-eks-best-practices/scalability/docs/control-plane/#api-priority-and-fairness)

这里我们看到集群默认有七个不同的优先级组

![共享并发](../images/shared-concurrency.png)

接下来,我们想看看该优先级组被使用的百分比,这样我们就可以了解某个优先级级别是否饱和。降低工作负载低级别的请求可能是可取的,但在领导选举级别的下降就不是了。

API 优先级和公平性 (APF) 系统有许多复杂的选项,其中一些选项可能会产生意想不到的后果。我们在现场看到的一个常见问题是增加队列深度到开始添加不必要延迟的程度。我们可以使用 `apiserver_flowcontrol_current_inqueue_request` 指标来监控这个问题。我们可以使用 `apiserver_flowcontrol_rejected_requests_total` 来检查丢弃。如果任何存储桶超过其并发性,这些指标将是非零值。

![使用中的请求](../images/requests-in-use.png)

增加队列深度可能会使 API 服务器成为显著的延迟源,应该谨慎进行。我们建议对创建的队列数量保持谨慎。例如,EKS 系统上的共享数量是 600,如果我们创建太多队列,这可能会减少重要队列(如领导选举队列或系统队列)中的共享,这些队列需要吞吐量。创建太多额外的队列可能会使这些队列的大小调整更加困难。

为了关注一个简单有影响的 APF 变更,我们只需从利用不足的存储桶中获取共享,并增加使用率达到最大的存储桶的大小。通过在这些存储桶之间智能地重新分配共享,您可以减少丢弃的可能性。

更多信息,请访问 EKS 最佳实践指南中的[API 优先级和公平性设置](https://aws.github.io/aws-eks-best-practices/scalability/docs/control-plane/#api-priority-and-fairness)。

### API 与 etcd 延迟
我们如何使用 API 服务器的指标/日志来确定是 API 服务器存在问题,还是 API 服务器上游/下游存在问题,或者两者兼有。为了更好地理解这一点,让我们看看 API 服务器和 etcd 之间的关系,以及很容易错误地对系统进行故障排查。

在下面的图表中,我们看到 API 服务器延迟,但我们也看到大部分延迟与 etcd 服务器相关,因为图表中显示大部分延迟都在 etcd 级别。如果在 20 秒的 API 服务器延迟期间有 15 秒的 etcd 延迟,那么大部分延迟实际上都在 etcd 级别。

通过查看整个流程,我们发现不应该仅关注 API 服务器,还应该寻找表明 etcd 处于压力状态的信号(即缓慢的应用计数器增加)。能够仅通过一瞥就快速转移到正确的问题领域,这就是仪表板强大的地方。

!!! tip
    本节中的仪表板可以在 https://github.com/RiskyAdventure/Troubleshooting-Dashboards/blob/main/api-troubleshooter.json 找到

![ETCD 压力](../images/etcd-duress.png)

### 控制平面与客户端问题
在这个图表中,我们正在寻找在该时间段内完成时间最长的 API 调用。在这种情况下,我们看到一个自定义资源 (CRD) 调用一个 APPLY 函数是在 05:40 时间段内延迟最严重的调用。

![最慢的请求](../images/slowest-requests.png)

有了这些数据,我们可以使用 Ad-Hoc PromQL 或 CloudWatch Insights 查询来拉取该时间段内的审核日志中的 LIST 请求,以查看这可能是哪个应用程序。

### 使用 CloudWatch 找到源头
指标最好用于找到我们想要查看的问题领域,并缩小时间范围和问题搜索参数。一旦我们有了这些数据,我们就想转向日志以获得更详细的时间和错误。为此,我们将使用 [CloudWatch Logs Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html) 将日志转换为指标。

例如,要调查上述问题,我们将使用以下 CloudWatch Logs Insights 查询来拉取 userAgent 和 requestURI,以确定是哪个应用程序导致了这种延迟。

!!! tip
    需要使用适当的 Count,以免拉取正常的 List/Resync 行为上的 Watch。

```
fields *@timestamp*, *@message*
| filter *@logStream* like "kube-apiserver-audit"
| filter ispresent(requestURI)
| filter verb = "list"
| parse requestReceivedTimestamp /\d+-\d+-(?<StartDay>\d+)T(?<StartHour>\d+):(?<StartMinute>\d+):(?<StartSec>\d+).(?<StartMsec>\d+)Z/
| parse stageTimestamp /\d+-\d+-(?<EndDay>\d+)T(?<EndHour>\d+):(?<EndMinute>\d+):(?<EndSec>\d+).(?<EndMsec>\d+)Z/
| fields (StartHour * 3600 + StartMinute * 60 + StartSec + StartMsec / 1000000) as StartTime, (EndHour * 3600 + EndMinute * 60 + EndSec + EndMsec / 1000000) as EndTime, (EndTime - StartTime) as DeltaTime
| stats avg(DeltaTime) as AverageDeltaTime, count(*) as CountTime by requestURI, userAgent
| filter CountTime >=50
| sort AverageDeltaTime desc
```

使用这个查询,我们发现有两个不同的代理在运行大量高延迟的列表操作。Splunk 和 CloudWatch 代理。有了这些数据,我们可以决定删除、更新或用另一个项目替换这个控制器。

![查询结果](../images/query-results.png)

!!! tip
    有关此主题的更多详细信息,请参见以下[博客](https://aws.amazon.com/blogs/containers/troubleshooting-amazon-eks-api-servers-with-prometheus/)

## 调度器
由于 EKS 控制平面实例在单独的 AWS 帐户中运行,我们无法抓取这些组件的指标(API 服务器除外)。但是,由于我们可以访问这些组件的审核日志,我们可以将这些日志转换为指标,以查看是否有任何子系统导致了扩展瓶颈。让我们使用 CloudWatch Logs Insights 来查看调度器队列中有多少未调度的 pod。

### 调度器日志中未调度的 pod
如果我们能够直接在自管理的 Kubernetes (如 Kops)上抓取调度器指标,我们将使用以下 PromQL 来了解调度器积压。

```
max without(instance)(scheduler_pending_pods)
```

由于我们无法在 EKS 中访问上述指标,我们将使用以下 CloudWatch Logs Insights 查询来查看在特定时间段内无法调度的积压。然后我们可以深入研究峰值时间段内的消息,以了解瓶颈的性质。例如,节点启动速度不够快,或调度器本身的速率限制器。

```
fields timestamp, pod, err, *@message*
| filter *@logStream* like "scheduler"
| filter *@message* like "Unable to schedule pod"
| parse *@message*  /^.(?<date>\d{4})\s+(?<timestamp>\d+:\d+:\d+\.\d+)\s+\S*\s+\S+\]\s\"(.*?)\"\s+pod=(?<pod>\"(.*?)\")\s+err=(?<err>\"(.*?)\")/
| count(*) as count by pod, err
| sort count desc
```

这里我们看到调度器报告 pod 未部署,因为 PVC 存储不可用。

![CloudWatch Logs 查询](../images/cwl-query.png)

!!! note
    必须打开控制平面的审核日志记录才能启用此功能。限制日志保留时间也是一个最佳实践,以免随时间推移不必要地增加成本。下面是使用 EKSCTL 工具打开所有日志记录功能的示例。

```yaml
cloudWatch:
  clusterLogging:
    enableTypes: ["*"]
    logRetentionInDays: 10
```

## Kube 控制器管理器
Kube 控制器管理器(像所有其他控制器一样)有限制它一次可以执行多少操作。让我们通过查看 KOPS 配置来查看一些这些标志是什么。

```yaml
  kubeControllerManager:
    concurrentEndpointSyncs: 5
    concurrentReplicasetSyncs: 5
    concurrentNamespaceSyncs: 10
    concurrentServiceaccountTokenSyncs: 5
    concurrentServiceSyncs: 5
    concurrentResourceQuotaSyncs: 5
    concurrentGcSyncs: 20
    kubeAPIBurst: 20
    kubeAPIQPS: "30"
```

这些控制器在集群高变化时期有队列会填满。在这种情况下,我们看到 replicaset 设置控制器在其队列中有大量积压。

![队列](../images/queues.png)

我们有两种不同的方法来解决这种情况。如果运行自管理,我们可以简单地增加并发 goroutine,但这会对 etcd 产生影响,因为 KCM 会处理更多数据。另一个选择是减少部署上的 `.spec.revisionHistoryLimit` 来减少可回滚的 replicaset 对象数量,从而减轻这个控制器的压力。

```yaml
spec:
  revisionHistoryLimit: 2
```

其他 Kubernetes 功能也可以进行调整或关闭,以减轻高变化率系统的压力。例如,如果应用程序中的 pod 不需要直接与 k8s API 通信,那么关闭将 secret 投射到这些 pod 中就会减少对 ServiceaccountTokenSyncs 的负载。如果可能,这是解决此类问题的更可取的方式。

```yaml
kind: Pod
spec:
  automountServiceAccountToken: false
```

在我们无法访问指标的系统中,我们再次可以查看日志来检测争用。如果我们想查看每个控制器或总体级别上正在处理的 API 请求数量,我们将使用以下 CloudWatch Logs Insights 查询。

### KCM 处理的总量

```
# 查询计算来自 kube-controller-manager 的 API qps,按控制器类型拆分。
# 如果任何特定控制器的值接近 20/秒,它很可能会遇到客户端 API 限流。
fields @timestamp, @logStream, @message
| filter @logStream like /kube-apiserver-audit/
| filter userAgent like /kube-controller-manager/
# 排除与租约相关的调用(不计入 kcm qps)
| filter requestURI not like "apis/coordination.k8s.io/v1/namespaces/kube-system/leases/kube-controller-manager"
# 排除 API 发现调用(不计入 kcm qps)
| filter requestURI not like "?timeout=32s"
# 排除 watch 调用(不计入 kcm qps)
| filter verb != "watch"
# 如果您想获取来自特定控制器的 API 调用计数,请取消注释下面的适当行:
# | filter user.username like "system:serviceaccount:kube-system:job-controller"
# | filter user.username like "system:serviceaccount:kube-system:cronjob-controller"
# | filter user.username like "system:serviceaccount:kube-system:deployment-controller"
# | filter user.username like "system:serviceaccount:kube-system:replicaset-controller"
# | filter user.username like "system:serviceaccount:kube-system:horizontal-pod-autoscaler"
# | filter user.username like "system:serviceaccount:kube-system:persistent-volume-binder"
# | filter user.username like "system:serviceaccount:kube-system:endpointslice-controller"
# | filter user.username like "system:serviceaccount:kube-system:endpoint-controller"
# | filter user.username like "system:serviceaccount:kube-system:generic-garbage-controller"
| stats count(*) as count by user.username
| sort count desc
```

这里的关键是在进入详细的故障排查阶段之前,要查看路径中的每一个步骤(API、调度器、KCM、etcd)。在生产环境中,您通常会发现需要调整 Kubernetes 的多个部分,才能使系统以最佳性能运行。很容易无意中对症下药(例如节点超时),而忽略了更大的瓶颈。

## ETCD
etcd 使用内存映射文件高效地存储键值对。有一个保护机制来设置这个内存空间的大小,通常设置为 2、4 和 8GB 的限制。数据库中的对象越少,etcd 在对象更新时需要清理旧版本的工作就越少。这个清理旧版本对象的过程称为压缩。经过一系列压缩操作后,会有一个后续的过程来恢复可用空间,称为碎片整理,这个过程会在达到一定阈值或固定时间间隔时发生。

我们可以做一些用户相关的操作来限制 Kubernetes 中的对象数量,从而减少压缩和碎片整理过程的影响。例如,Helm 保留了很高的 `revisionHistoryLimit`。这使得可以保留更多的旧对象(如 ReplicaSets)来进行回滚。将历史限制设置为 2,我们可以将对象(如 ReplicaSets)从十个减少到两个,这反过来会减轻系统的负载。

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  revisionHistoryLimit: 2
```

从监控的角度来看,如果系统延迟峰值以固定的模式出现,相隔几个小时,检查这个碎片整理过程是否是源头可能会很有帮助。我们可以使用 CloudWatch Logs 来查看这一点。

如果您想查看碎片整理的开始/结束时间,请使用以下查询:

```
fields *@timestamp*, *@message*
| filter *@logStream* like /etcd-manager/
| filter *@message* like /defraging|defraged/
| sort *@timestamp* asc
```

![碎片整理查询](../images/defrag.png)