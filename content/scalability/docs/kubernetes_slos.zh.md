
# Kubernetes 上游 SLO

Amazon EKS 运行与上游 Kubernetes 版本相同的代码,并确保 EKS 集群在 Kubernetes 社区定义的 SLO 范围内运行。Kubernetes[可扩展性特殊兴趣小组(SIG)](https://github.com/kubernetes/community/tree/master/sig-scalability)定义了可扩展性目标,并通过 SLI 和 SLO 调查性能瓶颈。

SLI 是我们衡量系统的方式,如可用于确定系统"运行良好"程度的指标或度量,例如请求延迟或计数。SLO 定义了系统"运行良好"时的预期值,例如请求延迟保持在 3 秒以内。Kubernetes SLO 和 SLI 侧重于 Kubernetes 组件的性能,与关注 EKS 集群端点可用性的 Amazon EKS 服务 SLA 完全独立。

Kubernetes 有许多功能允许用户使用自定义插件或驱动程序扩展系统,如 CSI 驱动程序、准入 Webhook 和自动缩放器。这些扩展可能会以不同的方式严重影响 Kubernetes 集群的性能,例如,如果 Webhook 目标不可用,具有 `failurePolicy=Ignore` 的准入 Webhook 可能会增加 K8s API 请求的延迟。Kubernetes 可扩展性 SIG 使用["你承诺,我们承诺"框架](https://github.com/kubernetes/community/blob/master/sig-scalability/slos/slos.md#how-we-define-scalability)定义可扩展性:

> 如果你承诺:
>     - 正确配置你的集群
>     - 合理使用可扩展性功能
>     - 将集群负载保持在[推荐限制](https://github.com/kubernetes/community/blob/master/sig-scalability/configs-and-limits/thresholds.md)内
>
> 那么我们承诺你的集群可以扩展,即:
>     - 所有 SLO 都得到满足。

## Kubernetes SLO

Kubernetes SLO 并未考虑所有可能影响集群的插件和外部限制,如工作节点扩展或准入 Webhook。这些 SLO 侧重于[Kubernetes 组件](https://kubernetes.io/docs/concepts/overview/components/),并确保 Kubernetes 操作和资源在预期范围内运行。SLO 有助于 Kubernetes 开发人员确保对 Kubernetes 代码的更改不会降低整个系统的性能。

[Kubernetes 可扩展性 SIG 定义了以下官方 SLO/SLI](https://github.com/kubernetes/community/blob/master/sig-scalability/slos/slos.md)。Amazon EKS 团队定期对 EKS 集群运行可扩展性测试,以监控随着更改和新版本发布而出现的性能降低情况。

|目标|定义|SLO|
|---|---|---|

|API请求延迟(变更)	|对单个对象进行变更API调用的处理延迟,以过去5分钟的第99百分位数衡量	|在默认的Kubernetes安装中,对于每个(资源,动词)对,不包括虚拟和聚合资源以及自定义资源定义,每个集群每天第99百分位数 <= 1秒	|
|API请求延迟(只读)	|处理非流式只读API调用的延迟,对于每个(资源,范围)对以过去5分钟的第99百分位数衡量	|在默认的Kubernetes安装中,对于每个(资源,范围)对,不包括虚拟和聚合资源以及自定义资源定义,(a) 如果`scope=resource`,第99百分位数每集群每天 <= 1秒 (b) 否则(如果`scope=namespace`或`scope=cluster`),第99百分位数每集群每天 <= 30秒	|
|Pod启动延迟	|可调度的无状态Pod的启动延迟,不包括拉取镜像和运行初始化容器的时间,从Pod创建时间戳到所有容器报告为已启动并通过监视观察到的时间,以过去5分钟的第99百分位数衡量	|在默认的Kubernetes安装中,每个集群每天第99百分位数 <= 5秒	|

### API请求延迟

`kube-apiserver`默认将`--request-timeout`设置为`1m0s`,这意味着请求最多可以运行一分钟(60秒)才会超时并被取消。定义的SLO延迟指标根据请求的类型(变更或只读)进行细分:

#### 变更

Kubernetes中的变更请求会修改资源,如创建、删除或更新。这些请求比较昂贵,因为这些更改必须先写入[etcd后端](https://kubernetes.io/docs/concepts/overview/components/#etcd)才能返回更新后的对象。[Etcd](https://etcd.io/)是一个分布式键值存储,用于存储Kubernetes集群的所有数据。

这种延迟以过去5分钟的第99百分位数来衡量Kubernetes资源的(资源,动词)对,例如测量创建Pod请求和更新Node请求的延迟。请求延迟必须 <= 1秒才能满足SLO。

#### 只读

只读请求会检索单个资源(如获取Pod X)或集合(如"从命名空间X获取所有Pod")。`kube-apiserver`维护了一个对象缓存,所以请求的资源可能来自缓存,也可能需要先从etcd中检索。
这些延迟也是以过去5分钟的第99百分位数来衡量,但只读请求可以有不同的范围。SLO定义了两个不同的目标:

* 对于针对*单个*资源的请求(即`kubectl get pod -n mynamespace my-controller-xxx`),请求延迟应保持在 <= 1秒。
* 对于针对命名空间或集群中多个资源的请求(例如`kubectl get pods -A`),延迟应保持在 <= 30秒

简化中文版本:

SLO 对不同请求范围有不同的目标值,因为针对 Kubernetes 资源列表的请求需要在 SLO 内返回所有对象的详细信息。在大型集群或大量资源集合中,这可能导致响应大小较大,需要一些时间才能返回。例如,在运行数万个 Pod 的集群中,每个 Pod 在 JSON 编码时大约为 1 KiB,返回集群中的所有 Pod 将包含 10MB 或更多。Kubernetes 客户端可以通过[使用 APIListChunking 分块检索大型资源集合](https://kubernetes.io/docs/reference/using-api/api-concepts/#retrieving-large-results-sets-in-chunks)来减小响应大小。

### Pod 启动延迟

这个 SLO 主要关注从 Pod 创建到容器实际开始执行的时间。为了测量这一点,计算从记录在 Pod 上的创建时间戳到[监视该 Pod](https://kubernetes.io/docs/reference/using-api/api-concepts/#efficient-detection-of-changes)报告容器已启动的时间差(不包括容器镜像拉取和 init 容器执行的时间)。为了满足 SLO,每个集群日的 Pod 启动延迟的 99 百分位必须保持在 <=5 秒以内。

请注意,这个 SLO 假设工作节点已经处于就绪状态,以便调度 Pod。这个 SLO 不考虑镜像拉取或 init 容器执行,并且也将测试限制在不使用持久存储插件的"无状态 Pod"上。

## Kubernetes SLI 指标

Kubernetes 还通过向 Kubernetes 组件添加 [Prometheus 指标](https://prometheus.io/docs/concepts/data_model/)来改善对 SLI 的可观察性,以随时间跟踪这些 SLI。使用 [Prometheus 查询语言 (PromQL)](https://prometheus.io/docs/prometheus/latest/querying/basics/)，我们可以构建查询来在 Prometheus 或 Grafana 仪表板中显示 SLI 性能随时间的变化,以下是上述 SLO 的一些示例。

### API 服务器请求延迟

|指标	|定义	|
|---	|---	|
|apiserver_request_sli_duration_seconds	| 每个动词、组、版本、资源、子资源、范围和组件的响应延迟分布(不包括 webhook 持续时间和优先级和公平性队列等待时间)秒数。	|
|apiserver_request_duration_seconds	| 每个动词、干运行值、组、版本、资源、子资源、范围和组件的响应延迟分布秒数。	|

*注意: `apiserver_request_sli_duration_seconds` 指标从 Kubernetes 1.27 开始可用。*
您可以使用这些指标来调查 API 服务器响应时间,并确定 Kubernetes 组件或其他插件/组件中是否存在瓶颈。以下查询基于[社区 SLO 仪表板](https://github.com/kubernetes/perf-tests/tree/master/clusterloader2/pkg/prometheus/manifests/dashboards)。

**API 请求延迟 SLI (可变)** - 此时间*不包括*Webhook 执行或队列等待时间。
`histogram_quantile(0.99, sum(rate(apiserver_request_sli_duration_seconds_bucket{verb=~"CREATE|DELETE|PATCH|POST|PUT", subresource!~"proxy|attach|log|exec|portforward"}[5m])) by (resource, subresource, verb, scope, le)) > 0`

**API 请求延迟总计 (可变)** - 这是请求在 API 服务器上总共花费的时间,这个时间可能比 SLI 时间长,因为它包括 Webhook 执行和 API 优先级和公平性等待时间。
`histogram_quantile(0.99, sum(rate(apiserver_request_duration_seconds_bucket{verb=~"CREATE|DELETE|PATCH|POST|PUT", subresource!~"proxy|attach|log|exec|portforward"}[5m])) by (resource, subresource, verb, scope, le)) > 0`

在这些查询中,我们排除了不会立即返回的流 API 请求,例如 `kubectl port-forward` 或 `kubectl exec` 请求(`subresource!~"proxy|attach|log|exec|portforward"`)。我们只过滤修改对象的 Kubernetes 动词(`verb=~"CREATE|DELETE|PATCH|POST|PUT"`)。然后我们计算过去 5 分钟内该延迟的第 99 百分位。

我们可以使用类似的查询来获取只读 API 请求,我们只需修改过滤的动词,包括只读操作 `LIST` 和 `GET`。根据请求的范围,即获取单个资源还是列出多个资源,也有不同的 SLO 阈值。

**API 请求延迟 SLI (只读)** - 此时间*不包括*Webhook 执行或队列等待时间。
对于单个资源(scope=resource, threshold=1s)
`histogram_quantile(0.99, sum(rate(apiserver_request_sli_duration_seconds_bucket{verb=~"GET", scope=~"resource"}[5m])) by (resource, subresource, verb, scope, le))`

对于同一命名空间中的资源集合(scope=namespace, threshold=5s)
`histogram_quantile(0.99, sum(rate(apiserver_request_sli_duration_seconds_bucket{verb=~"LIST", scope=~"namespace"}[5m])) by (resource, subresource, verb, scope, le))`

对于整个集群范围内的资源集合(scope=cluster, threshold=30s)
`histogram_quantile(0.99, sum(rate(apiserver_request_sli_duration_seconds_bucket{verb=~"LIST", scope=~"cluster"}[5m])) by (resource, subresource, verb, scope, le))`

**API 请求延迟总计 (只读)** - 这是请求在 API 服务器上总共花费的时间,这个时间可能比 SLI 时间长,因为它包括 Webhook 执行和等待时间。
对于单个资源(scope=resource, threshold=1s)

`histogram_quantile(0.99, sum(rate(apiserver_request_duration_seconds_bucket{verb=~"GET", scope=~"resource"}[5m])) by (resource, subresource, verb, scope, le))`

对于同一命名空间中的一组资源(scope=namespace, threshold=5s)  
`histogram_quantile(0.99, sum(rate(apiserver_request_duration_seconds_bucket{verb=~"LIST", scope=~"namespace"}[5m])) by (resource, subresource, verb, scope, le))`

对于整个集群范围内的一组资源(scope=cluster, threshold=30s)  
`histogram_quantile(0.99, sum(rate(apiserver_request_duration_seconds_bucket{verb=~"LIST", scope=~"cluster"}[5m])) by (resource, subresource, verb, scope, le))`

SLI指标提供了关于Kubernetes组件性能的洞见,因为它们排除了请求在API优先级和公平性队列中等待、通过准入Webhook或其他Kubernetes扩展处理的时间。总体指标提供了更全面的视图,因为它反映了您的应用程序等待从API服务器获得响应的时间。比较这些指标可以提供关于请求处理延迟引入位置的洞见。

### Pod启动延迟

|指标	| 定义	|
|---	|---	|
|kubelet_pod_start_sli_duration_seconds	|启动Pod所需的时间(以秒为单位),不包括拉取镜像和运行Init容器的时间,从Pod创建时间戳到所有容器报告为已启动并通过监视观察到的时间	|
|kubelet_pod_start_duration_seconds	|从Kubelet第一次看到Pod到Pod开始运行所需的时间(以秒为单位)。这不包括调度Pod或扩展工作节点容量的时间。	|

*注意:  `kubelet_pod_start_sli_duration_seconds`从Kubernetes 1.27开始可用。*

与上述查询类似,您可以使用这些指标来了解节点扩展、镜像拉取和Init容器延迟Pod启动与Kubelet操作之间的差异。

**Pod启动延迟SLI -** 这是从Pod创建到应用程序容器报告为运行所需的时间。这包括工作节点容量可用和Pod调度所需的时间,但不包括拉取镜像或运行Init容器所需的时间。  
`histogram_quantile(0.99, sum(rate(kubelet_pod_start_sli_duration_seconds_bucket[5m])) by (le))`

**Pod启动延迟总计 -** 这是Kubelet第一次启动Pod所需的时间。这是从Kubelet通过WATCH接收Pod开始计算的,不包括工作节点扩展或调度的时间。这包括拉取镜像和运行Init容器所需的时间。  
`histogram_quantile(0.99, sum(rate(kubelet_pod_start_duration_seconds_bucket[5m])) by (le))`

## 您集群上的SLO

如果您正在从 EKS 集群中的 Kubernetes 资源收集 Prometheus 指标,您可以深入了解 Kubernetes 控制平面组件的性能。

[perf-tests 仓库](https://github.com/kubernetes/perf-tests/)包含 Grafana 仪表板,用于显示集群在测试期间的延迟和关键性能指标。perf-tests 配置利用了[kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack),这是一个开源项目,配置好可以收集 Kubernetes 指标,但您也可以[使用 Amazon Managed Prometheus 和 Amazon Managed Grafana](https://aws-observability.github.io/terraform-aws-observability-accelerator/eks/)。

如果您正在使用 `kube-prometheus-stack` 或类似的 Prometheus 解决方案,您可以安装相同的仪表板来实时观察集群的 SLO。

1. 您首先需要使用 `kubectl apply -f prometheus-rules.yaml` 安装仪表板中使用的 Prometheus 规则。您可以在此处下载规则副本:https://github.com/kubernetes/perf-tests/blob/master/clusterloader2/pkg/prometheus/manifests/prometheus-rules.yaml
    1. 请确保文件中的命名空间与您的环境匹配
    2. 验证标签是否与 `kube-prometheus-stack` 的 `prometheus.prometheusSpec.ruleSelector` helm 值匹配
2. 然后您可以在 Grafana 中安装仪表板。JSON 仪表板和生成它们的 Python 脚本可在此处获得:https://github.com/kubernetes/perf-tests/tree/master/clusterloader2/pkg/prometheus/manifests/dashboards
    1. [the `slo.json` dashboard](https://github.com/kubernetes/perf-tests/blob/master/clusterloader2/pkg/prometheus/manifests/dashboards/slo.json) 显示了集群在 Kubernetes SLO 方面的性能

请注意,SLO 侧重于集群中 Kubernetes 组件的性能,但您还可以查看其他指标,这些指标提供了不同的视角或洞见。Kubernetes 社区项目如 [Kube-state-metrics](https://github.com/kubernetes/kube-state-metrics/tree/main) 可以帮助您快速分析集群趋势。来自 Kubernetes 社区的大多数常见插件和驱动程序也会发出 Prometheus 指标,允许您调查自动扩缩器或自定义调度器等内容。

[Observability Best Practices Guide](https://aws-observability.github.io/observability-best-practices/guides/containers/oss/eks/best-practices-metrics-collection/#control-plane-metrics) 提供了其他 Kubernetes 指标的示例,您可以使用这些指标获得更深入的洞见。
