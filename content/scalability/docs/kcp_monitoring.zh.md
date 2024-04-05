!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。
[控制平面监控]

## API 服务器
在查看我们的 API 服务器时,我们需要记住它的一个功能是限制入站请求,以防止控制平面过载。在 API 服务器级别看起来可能是瓶颈,但实际上可能是在保护它免受更严重的问题。我们需要权衡增加通过系统的请求量的利弊。要确定是否应该增加 API 服务器的值,以下是我们需要注意的一些小样本:

1. 通过系统的请求延迟是多少?
2. 这种延迟是 API 服务器本身还是"下游"的某些东西,如 etcd?
3. API 服务器队列深度是否是导致这种延迟的因素?
4. API 优先级和公平性 (APF) 队列是否为我们想要的 API 调用模式设置正确?

## 问题出在哪里?
首先,我们可以使用 API 延迟指标来洞察 API 服务器为请求提供服务需要多长时间。让我们使用下面的 PromQL 和 Grafana 热图来显示这些数据。
```
max(increase(apiserver_request_duration_seconds_bucket{subresource!="status",subresource!="token",subresource!="scale",subresource!="/healthz",subresource!="binding",subresource!="proxy",verb!="WATCH"}[$__rate_interval])) by (le)
```

[简体中文翻译:]

!!! tip
    有关如何使用本文中使用的 API 仪表板监控 API 服务器的深入写作,请参见以下[博客](https://aws.amazon.com/blogs/containers/troubleshooting-amazon-eks-api-servers-with-prometheus/)

![API 请求持续时间热图](../images/api-request-duration.png)

这些请求都在一秒钟以内,这表明控制平面能够及时处理请求。但如果情况并非如此呢?

我们在上述 API 请求持续时间中使用的格式是热图。热图格式的好处是它默认告诉我们 API 的超时值(60 秒)。但我们真正需要知道的是,在达到超时阈值之前,应该关注的阈值是多少。作为可接受阈值的粗略指南,我们可以使用上游 Kubernetes SLO,可以在[此处](https://github.com/kubernetes/community/blob/master/sig-scalability/slos/slos.md#steady-state-slisslos)找到。

!!! tip
    注意这个语句中的 max 函数?当使用聚合多个服务器(默认情况下 EKS 上有两个 API 服务器)的指标时,不应将它们平均在一起。

### 非对称流量模式
如果一个 API 服务器 [pod] 负载较轻,而另一个负载较重呢?如果我们将这两个数字平均,可能会误解正在发生的情况。例如,这里我们有三个 API 服务器,但所有负载都在其中一个 API 服务器上。作为一般规则,任何具有多个服务器(如 etcd 和 API 服务器)的系统在研究扩展和性能问题时都应该分开。

![总的飞行请求](../images/inflight-requests.png)

随着向 API 优先级和公平性的转变,系统上的总请求数只是检查 API 服务器是否过载的一个因素。由于系统现在基于一系列队列工作,我们必须查看这些队列是否已满,以及该队列的流量是否被丢弃。

让我们使用以下查询来查看这些队列:
```
max without(instance)(apiserver_flowcontrol_request_concurrency_limit{})
```


# 简化中文翻译

!!! note
    有关 API A&F 如何工作的更多信息,请参见以下[最佳实践指南](https://aws.github.io/aws-eks-best-practices/scalability/docs/control-plane/#api-priority-and-fairness)

在这里,我们看到集群默认提供的七个不同的优先级组

![共享并发](../images/shared-concurrency.png)

接下来,我们想看看该优先级组使用的百分比,以便我们了解某个特定优先级级别是否饱和。降低工作负载低级别的请求可能是可取的,但在领导选举级别的下降则不应如此。

API 优先级和公平性 (APF) 系统有许多复杂的选项,其中一些选项可能会产生意想不到的后果。我们在现场看到的一个常见问题是,增加队列深度到开始添加不必要延迟的程度。我们可以使用 `apiserver_flowcontrol_current_inqueue_request` 指标来监控这个问题。我们可以使用 `apiserver_flowcontrol_rejected_requests_total` 检查下降情况。如果任何存储桶超过其并发性,这些指标将为非零值。

![使用中的请求](../images/requests-in-use.png)

增加队列深度可能会使 API 服务器成为延迟的重要来源,应该谨慎进行。我们建议慎重创建队列的数量。例如,EKS 系统上的共享数量为 600,如果我们创建太多队列,这可能会减少重要队列(如领导选举队列或系统队列)中的共享,从而降低吞吐量。创建太多额外的队列可能会使这些队列的正确大小调整变得更加困难。

为了关注一个简单有影响的 APF 变更,我们只需从利用不足的存储桶中获取共享,并增加达到最大使用率的存储桶的大小。通过智能地在这些存储桶之间重新分配共享,您可以减少下降的可能性。

更多信息,请访问 EKS 最佳实践指南中的[API 优先级和公平性设置](https://aws.github.io/aws-eks-best-practices/scalability/docs/control-plane/#api-priority-and-fairness)。

### API 与 etcd 延迟

我们如何使用 API 服务器的指标/日志来确定是 API 服务器存在问题,还是 API 服务器上游/下游存在问题,亦或两者兼有。为了更好地理解这一点,让我们看看 API 服务器和 etcd 之间的关系,以及如何轻松地对错误系统进行故障排查。

在下面的图表中,我们看到 API 服务器延迟,但我们也看到大部分延迟与 etcd 服务器相关,因为图表中显示的大部分延迟都在 etcd 级别。如果在 20 秒的 API 服务器延迟期间存在 15 秒的 etcd 延迟,那么大部分延迟实际上都在 etcd 级别。
[通过查看整个流程,我们发现专注于 API 服务器并不明智,还需要关注表明 etcd 处于压力之下的信号(即缓慢增加的应用计数器)。能够仅通过一瞥就快速定位到正确的问题领域,这就是仪表盘的强大之处。

!!! 提示
    本节中的仪表盘可以在 https://github.com/RiskyAdventure/Troubleshooting-Dashboards/blob/main/api-troubleshooter.json 找到

![ETCD 压力](../images/etcd-duress.png)

### 控制平面与客户端问题
在这个图表中,我们正在寻找在该时间段内完成时间最长的 API 调用。在这种情况下,我们发现一个自定义资源 (CRD) 正在调用一个在 05:40 时间段内延迟最严重的 APPLY 函数。

![最慢的请求](../images/slowest-requests.png)

有了这些数据,我们可以使用临时的 PromQL 或 CloudWatch Insights 查询来从审核日志中提取该时间段内的 LIST 请求,以确定可能是哪个应用程序导致了这个问题。

### 使用 CloudWatch 找到源头
指标最好用于找到我们想要关注的问题领域,并缩小时间范围和问题搜索参数。一旦我们有了这些数据,我们就想转向日志以获取更详细的时间和错误信息。为此,我们将使用 [CloudWatch Logs Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html) 将日志转换为指标。

例如,为了调查上述问题,我们将使用以下 CloudWatch Logs Insights 查询来提取 userAgent 和 requestURI,以确定是哪个应用程序导致了这种延迟。

!!! 提示
    需要使用适当的计数,以免拉取正常的 List/Resync 行为的 Watch。]
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

使用此查询,我们发现有两个不同的代理正在执行大量的高延迟列表操作。Splunk 和 CloudWatch 代理。有了这些数据,我们可以决定删除、更新或用另一个项目替换这个控制器。

![查询结果](../images/query-results.png)

!!! 提示
    有关此主题的更多详细信息,请参见以下[博客](https://aws.amazon.com/blogs/containers/troubleshooting-amazon-eks-api-servers-with-prometheus/)

## 调度器
由于 EKS 控制平面实例在单独的 AWS 帐户中运行,我们无法直接采集这些组件的指标(API 服务器除外)。但是,由于我们可以访问这些组件的审核日志,我们可以将这些日志转换为指标,以查看是否有任何子系统造成了扩展瓶颈。让我们使用 CloudWatch Logs Insights 来查看调度器队列中有多少未调度的 pod。

### 调度器日志中的未调度 pod
如果我们能够直接采集自管理 Kubernetes(如 Kops)上的调度器指标,我们将使用以下 PromQL 来了解调度器积压情况。
```
max without(instance)(scheduler_pending_pods)
```

由于我们无法在 EKS 中访问上述指标,我们将使用以下 CloudWatch Logs Insights 查询来查看在特定时间段内无法调度的 pod 数量,从而了解积压情况。然后我们可以进一步深入分析峰值时段的消息,以了解瓶颈的性质,例如节点启动速度不够快,或调度程序本身的速率限制器。
```
fields timestamp, pod, err, *@message*
| filter *@logStream* like "scheduler"
| filter *@message* like "Unable to schedule pod"
| parse *@message*  /^.(?<date>\d{4})\s+(?<timestamp>\d+:\d+:\d+\.\d+)\s+\S*\s+\S+\]\s\"(.*?)\"\s+pod=(?<pod>\"(.*?)\")\s+err=(?<err>\"(.*?)\")/
| count(*) as count by pod, err
| sort count desc
```

这里我们看到调度程序的错误,说 pod 没有部署,因为存储 PVC 不可用。

![CloudWatch Logs query](../images/cwl-query.png)

!!! 注意
    必须打开控制平面的审核日志记录,才能启用此功能。限制日志保留时间也是一种最佳实践,以免随时间推移不必要地增加成本。下面是使用 EKSCTL 工具打开所有日志功能的示例。
```yaml
cloudWatch:
  clusterLogging:
    enableTypes: ["*"]
    logRetentionInDays: 10
```

## Kube 控制器管理器
Kube 控制器管理器与所有其他控制器一样,对于一次可以执行的操作数量有限制。让我们通过查看一个 KOPS 配置来了解这些参数是什么。
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

这些控制器在集群高流失期间会积累队列。在这种情况下,我们看到复制集控制器在其队列中有大量积压。

![Queues](../images/queues.png)

我们有两种不同的方法来解决这种情况。如果是自主管理,我们可以简单地增加并发 goroutine 的数量,但这将对 etcd 产生影响,因为在 KCM 中会处理更多数据。另一个选择是减少部署中使用 `.spec.revisionHistoryLimit` 的复制集对象数量,从而减少可回滚的复制集对象数量,进而减轻这个控制器的压力。
```yaml
spec:
  revisionHistoryLimit: 2
```

其他 Kubernetes 功能可以调整或关闭以减轻高流失率系统的压力。例如,如果我们 pod 中的应用程序不需要直接与 k8s API 通信,那么关闭将秘密投射到这些 pod 中就会减少 ServiceaccountTokenSyncs 的负载。如果可能,这是解决此类问题的更理想的方式。
```yaml
kind: Pod
spec:
  automountServiceAccountToken: false
```

在无法访问指标的系统中,我们可以再次查看日志来检测争用情况。如果我们想查看每个控制器或总体级别上正在处理的请求数量,我们将使用以下 CloudWatch Logs Insights 查询。

### KCM 处理的总量

![any text]
```
# Query to count API qps coming from kube-controller-manager, split by controller type.
# If you're seeing values close to 20/sec for any particular controller, it's most likely seeing client-side API throttling.
fields @timestamp, @logStream, @message
| filter @logStream like /kube-apiserver-audit/
| filter userAgent like /kube-controller-manager/
# Exclude lease-related calls (not counted under kcm qps)
| filter requestURI not like "apis/coordination.k8s.io/v1/namespaces/kube-system/leases/kube-controller-manager"
# Exclude API discovery calls (not counted under kcm qps)
| filter requestURI not like "?timeout=32s"
# Exclude watch calls (not counted under kcm qps)
| filter verb != "watch"
# If you want to get counts of API calls coming from a specific controller, uncomment the appropriate line below:
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

这里的关键要点是,在研究可扩展性问题时,在进入详细的故障排查阶段之前,要先看看路径上的每一个步骤(API、调度程序、KCM、etcd)。在生产环境中,您通常会发现需要对Kubernetes的多个部分进行调整,才能使系统发挥最佳性能。很容易无意中排查只是一个更大瓶颈的症状(例如节点超时)。

## ETCD
etcd使用内存映射文件高效地存储键值对。有一个保护机制,可以设置这个内存空间的大小,通常设置为2GB、4GB和8GB的限制。数据库中的对象越少,etcd在对象更新和清理旧版本时需要做的清理工作就越少。这个清理旧版本对象的过程称为压缩。经过多次压缩操作后,会有一个后续的过程来恢复可用空间,这个过程称为碎片整理,它会在达到一定阈值或按固定时间表执行。

我们可以做一些与用户相关的操作来限制Kubernetes中对象的数量,从而减少压缩和碎片整理过程的影响。例如,Helm保留了很高的`revisionHistoryLimit`。这使得可以保留较旧的对象(如ReplicaSets)以便进行回滚。通过将历史记录限制设置为2,我们可以将对象(如ReplicaSets)的数量从十个减少到两个,从而减轻系统的负载。
```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  revisionHistoryLimit: 2
```

从监控的角度来看,如果系统延迟峰值以固定模式出现,并且间隔数小时,检查这个碎片整理过程是否是导致的源头可能会很有帮助。我们可以通过使用CloudWatch日志来查看这一点。

如果您想查看碎片整理的开始/结束时间,可以使用以下查询:
```
fields *@timestamp*, *@message*
| filter *@logStream* like /etcd-manager/
| filter *@message* like /defraging|defraged/
| sort *@timestamp* asc
```

![Defrag 查询](../images/defrag.png)
