
!!! 注意
    此页面的内容是使用大型语言模型(Claude 3)创建的,并基于英文版本。如有差异,以英文版本为准。

# Kubernetes 控制平面

Kubernetes 控制平面由 Kubernetes API 服务器、Kubernetes 控制器管理器、调度程序和 Kubernetes 正常运行所需的其他组件组成。这些组件的可扩展性限制因集群中运行的内容而有所不同,但对扩展性影响最大的领域包括 Kubernetes 版本、利用率和单个节点的扩展。

## 使用 EKS 1.24 或更高版本

EKS 1.24 引入了一些变更,并将容器运行时切换为 [containerd](https://containerd.io/) 而不是 docker。Containerd 通过将容器运行时功能限制为与 Kubernetes 需求更加一致,从而帮助集群扩展,提高单个节点的性能。Containerd 在所有受支持的 EKS 版本中都可用,如果您希望在 1.24 之前的版本中切换到 containerd,请使用 [`--container-runtime` 引导标志](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html#containerd-bootstrap)。

## 限制工作负载和节点突发

!!! 注意
    为了避免达到控制平面的 API 限制,您应该限制集群规模增加的突发性(例如,从 1000 个节点增加到 1100 个节点,或从 4000 个 pod 增加到 4500 个 pod)。

EKS 控制平面将随着集群的增长而自动扩展,但扩展速度有限制。当您首次创建 EKS 集群时,控制平面可能无法立即扩展到数百个节点或数千个 pod。要了解 EKS 如何进行扩展改进,请参阅[此博客文章](https://aws.amazon.com/blogs/containers/amazon-eks-control-plane-auto-scaling-enhancements-improve-speed-by-4x/)。

大型应用程序的扩展需要基础设施能够适应并完全准备就绪(例如,预热负载均衡器)。为了控制扩展速度,请确保您是根据应用程序的正确指标进行扩展。CPU 和内存扩展可能无法准确预测应用程序的约束,在 Kubernetes Horizontal Pod Autoscaler (HPA) 中使用自定义指标(例如每秒请求数)可能是更好的扩展选择。

要使用自定义指标,请参见 [Kubernetes 文档](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/#autoscaling-on-multiple-metrics-and-custom-metrics)中的示例。如果您有更高级的扩展需求,或需要根据外部源(例如 AWS SQS 队列)进行扩展,请使用 [KEDA](https://keda.sh) 进行基于事件的工作负载扩展。

## 安全地缩减节点和 pod

### 替换长期运行的实例

定期更换节点可以通过避免配置漂移和仅在长时间运行后才会出现的问题(例如内存泄漏)来保持集群健康。自动替换将为您提供节点升级和安全修补的良好流程和实践。如果集群中的每个节点都定期替换,那么维护持续维护的单独流程所需的工作量就会减少。

使用 Karpenter 的[生存时间(TTL)](https://aws.github.io/aws-eks-best-practices/karpenter/#use-timers-ttl-to-automatically-delete-nodes-from-the-cluster)设置在指定的时间后替换实例。自管理节点组可以使用 `max-instance-lifetime` 设置自动循环节点。托管节点组目前没有这个功能,但您可以在 [GitHub](https://github.com/aws/containers-roadmap/issues/1190) 上跟踪请求。

### 删除利用不足的节点

您可以使用 Kubernetes Cluster Autoscaler 中的 `--scale-down-utilization-threshold` 设置或在 Karpenter 中使用 `ttlSecondsAfterEmpty` 配置器设置,在节点上没有正在运行的工作负载时删除节点。

### 使用 Pod 中断预算和安全节点关闭

从 Kubernetes 集群中删除 Pod 和节点需要控制器对多个资源(例如 EndpointSlices)进行更新。频繁或过快地执行此操作可能会导致 API 服务器节流和应用程序中断,因为更改会传播到控制器。[Pod 中断预算](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)是在节点被删除或重新调度时保护工作负载可用性的最佳实践。

## 在运行 Kubectl 时使用客户端缓存

inefficiently 使用 kubectl 命令可能会给 Kubernetes API 服务器增加额外的负载。您应该避免运行重复使用 kubectl 的脚本或自动化(例如在 for 循环中)或运行没有本地缓存的命令。

`kubectl` 有一个客户端缓存,它缓存来自集群的发现信息,以减少所需的 API 调用次数。缓存默认启用,每 10 分钟刷新一次。

如果您从容器中运行 kubectl 或没有客户端缓存,您可能会遇到 API 节流问题。建议通过挂载 `--cache-dir` 来保留集群缓存,以避免进行不必要的 API 调用。

## 禁用 kubectl 压缩

禁用 kubectl 在您的 kubeconfig 文件中的压缩可以减少 API 和客户端 CPU 的使用。默认情况下，服务器会压缩发送到客户端的数据以优化网络带宽。这会给每个请求的客户端和服务器增加 CPU 负载，禁用压缩可以在您有足够带宽的情况下减少开销和延迟。要禁用压缩，您可以使用 `--disable-compression=true` 标志或在您的 kubeconfig 文件中设置 `disable-compression: true`。
```
apiVersion: v1
clusters:
- cluster:
    server: serverURL
    disable-compression: true
  name: cluster
```

## 分片集群自动缩放器

[Kubernetes 集群自动缩放器](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/proposals/scalability_tests.md)已经过测试,可以扩展到 1000 个节点。在拥有超过 1000 个节点的大型集群中,建议以分片模式运行多个集群自动缩放器实例。每个集群自动缩放器实例都被配置为缩放一组节点组。以下示例显示了 2 个集群自动缩放配置,它们被配置为各自缩放 4 个节点组。

ClusterAutoscaler-1
```
autoscalingGroups:
- name: eks-core-node-grp-20220823190924690000000011-80c1660e-030d-476d-cb0d-d04d585a8fcb
  maxSize: 50
  minSize: 2
- name: eks-data_m1-20220824130553925600000011-5ec167fa-ca93-8ca4-53a5-003e1ed8d306
  maxSize: 450
  minSize: 2
- name: eks-data_m2-20220824130733258600000015-aac167fb-8bf7-429d-d032-e195af4e25f5
  maxSize: 450
  minSize: 2
- name: eks-data_m3-20220824130553914900000003-18c167fa-ca7f-23c9-0fea-f9edefbda002
  maxSize: 450
  minSize: 2
```

集群自动扩缩容器-2
```
autoscalingGroups:
- name: eks-data_m4-2022082413055392550000000f-5ec167fa-ca86-6b83-ae9d-1e07ade3e7c4
  maxSize: 450
  minSize: 2
- name: eks-data_m5-20220824130744542100000017-02c167fb-a1f7-3d9e-a583-43b4975c050c
  maxSize: 450
  minSize: 2
- name: eks-data_m6-2022082413055392430000000d-9cc167fa-ca94-132a-04ad-e43166cef41f
  maxSize: 450
  minSize: 2
- name: eks-data_m7-20220824130553921000000009-96c167fa-ca91-d767-0427-91c879ddf5af
  maxSize: 450
  minSize: 2
```


## API 优先级和公平性

![](../images/APF.jpg)

### 概述

<iframe width="560" height="315" src="https://www.youtube.com/embed/YnPPHBawhE0" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

为了在请求量增加期间保护自身免受过载,API 服务器限制了它在给定时间内可以有的未完成请求数量。一旦超过这个限制,API 服务器将开始拒绝请求,并向客户端返回 429 HTTP 响应代码"请求过多"。服务器丢弃请求并让客户端稍后重试要比没有服务器端请求数量限制并导致控制平面过载,从而导致性能下降或不可用性更好。

Kubernetes 用来配置这些未完成请求如何在不同请求类型之间分配的机制称为 [API 优先级和公平性](https://kubernetes.io/docs/concepts/cluster-administration/flow-control/)。API 服务器通过将 `--max-requests-inflight` 和 `--max-mutating-requests-inflight` 标志指定的值相加来配置它可以接受的未完成请求总数。EKS 使用这些标志的默认值 400 和 200 请求,允许在给定时间内调度 600 个请求。但是,随着它根据增加的利用率和工作负载变化来扩展控制平面,它相应地将未完成请求配额增加到最高 2000(可能会更改)。APF 指定了这些未完成请求配额如何在不同请求类型之间进一步细分。请注意,EKS 控制平面是高度可用的,每个集群至少有 2 个注册的 API 服务器。这意味着您的集群可以处理的未完成请求总数是每个 kube-apiserver 设置的未完成请求配额的两倍(或者如果进一步水平扩展,则更高)。这相当于最大 EKS 集群每秒处理数千个请求。

Kubernetes 有两种对象,称为 PriorityLevelConfigurations 和 FlowSchemas,用于配置如何在不同类型的请求之间划分总请求数。这些对象由 API Server 自动维护,EKS 使用给定 Kubernetes 次版本的默认配置。PriorityLevelConfigurations 表示总允许请求数的一部分。例如,workload-high PriorityLevelConfiguration 被分配了 600 个请求中的 98 个。分配给所有 PriorityLevelConfigurations 的请求总和将等于 600(或略高于 600,因为 API Server 会将给定级别的小数部分向上舍入)。要检查集群中的 PriorityLevelConfigurations 以及分配给每个 PriorityLevelConfiguration 的请求数,可以运行以下命令。这些是 EKS 1.24 上的默认值:
```
$ kubectl get --raw /metrics | grep apiserver_flowcontrol_request_concurrency_limit
apiserver_flowcontrol_request_concurrency_limit{priority_level="catch-all"} 13
apiserver_flowcontrol_request_concurrency_limit{priority_level="global-default"} 49
apiserver_flowcontrol_request_concurrency_limit{priority_level="leader-election"} 25
apiserver_flowcontrol_request_concurrency_limit{priority_level="node-high"} 98
apiserver_flowcontrol_request_concurrency_limit{priority_level="system"} 74
apiserver_flowcontrol_request_concurrency_limit{priority_level="workload-high"} 98
apiserver_flowcontrol_request_concurrency_limit{priority_level="workload-low"} 245
```

第二种对象是 FlowSchemas。具有给定属性集的 API 服务器请求被归类到同一个 FlowSchema 下。这些属性包括经过身份验证的用户或请求的属性,例如 API 组、命名空间或资源。FlowSchema 还指定了此类请求应映射到的 PriorityLevelConfiguration。这两个对象共同说明,"我希望这种类型的请求计入这个份额的正在进行的请求。"当请求到达 API 服务器时,它将检查每个 FlowSchema,直到找到一个与所有必需属性匹配的 FlowSchema。如果多个 FlowSchema 与请求匹配,API 服务器将选择具有最小匹配优先级的 FlowSchema,该优先级在对象中指定为属性。

可以使用以下命令查看 FlowSchemas 到 PriorityLevelConfigurations 的映射:
```
$ kubectl get flowschemas
NAME                           PRIORITYLEVEL     MATCHINGPRECEDENCE   DISTINGUISHERMETHOD   AGE     MISSINGPL
exempt                         exempt            1                    <none>                7h19m   False
eks-exempt                     exempt            2                    <none>                7h19m   False
probes                         exempt            2                    <none>                7h19m   False
system-leader-election         leader-election   100                  ByUser                7h19m   False
endpoint-controller            workload-high     150                  ByUser                7h19m   False
workload-leader-election       leader-election   200                  ByUser                7h19m   False
system-node-high               node-high         400                  ByUser                7h19m   False
system-nodes                   system            500                  ByUser                7h19m   False
kube-controller-manager        workload-high     800                  ByNamespace           7h19m   False
kube-scheduler                 workload-high     800                  ByNamespace           7h19m   False
kube-system-service-accounts   workload-high     900                  ByNamespace           7h19m   False
eks-workload-high              workload-high     1000                 ByUser                7h14m   False
service-accounts               workload-low      9000                 ByUser                7h19m   False
global-default                 global-default    9900                 ByUser                7h19m   False
catch-all                      catch-all         10000                ByUser                7h19m   False
```

优先级级别配置(PriorityLevelConfigurations)可以有队列(Queue)、拒绝(Reject)或豁免(Exempt)三种类型。对于队列(Queue)和拒绝(Reject)类型,会对该优先级别的最大并发请求数进行限制,但行为会有所不同。例如,workload-high优先级级别配置使用队列(Queue)类型,为controller-manager、endpoint-controller、scheduler、EKS相关控制器以及在kube-system命名空间中运行的Pod提供了98个可用请求。由于使用了队列(Queue)类型,API服务器将尝试将请求保留在内存中,并希望在这些请求超时之前,并发请求数降到98以下。如果某个请求在队列中超时,或者已经有太多请求排队,API服务器别无选择,只能丢弃该请求并向客户端返回429响应。请注意,排队可能会防止请求收到429响应,但代价是请求的端到端延迟会增加。

现在考虑映射到catch-all优先级级别配置(PriorityLevelConfiguration)的catch-all FlowSchema,该配置使用拒绝(Reject)类型。如果客户端达到13个并发请求的限制,API服务器将不会使用排队机制,而是立即丢弃请求并返回429响应代码。最后,映射到类型为豁免(Exempt)的优先级级别配置(PriorityLevelConfiguration)的请求永远不会收到429响应,并且会立即被调度。这种类型用于高优先级请求,如健康检查(healthz)请求或来自system:masters组的请求。

### 监控APF和丢弃的请求

要确认是否有任何请求由于APF而被丢弃,可以监控API服务器的`apiserver_flowcontrol_rejected_requests_total`指标,以检查受影响的FlowSchema和PriorityLevelConfigurations。例如,该指标显示100个来自service-accounts FlowSchema的请求由于在workload-low队列中超时而被丢弃:
```
% kubectl get --raw /metrics | grep apiserver_flowcontrol_rejected_requests_total
apiserver_flowcontrol_rejected_requests_total{flow_schema="service-accounts",priority_level="workload-low",reason="time-out"} 100
```

要检查给定的 PriorityLevelConfiguration 与收到 429 错误或由于排队而导致的延迟增加之间的接近程度，您可以比较并发限制和正在使用的并发之间的差异。在此示例中，我们有 100 个请求的缓冲区。
```
% kubectl get --raw /metrics | grep 'apiserver_flowcontrol_request_concurrency_limit.*workload-low'
apiserver_flowcontrol_request_concurrency_limit{priority_level="workload-low"} 245

% kubectl get --raw /metrics | grep 'apiserver_flowcontrol_request_concurrency_in_use.*workload-low'
apiserver_flowcontrol_request_concurrency_in_use{flow_schema="service-accounts",priority_level="workload-low"} 145
```

要检查给定的 PriorityLevelConfiguration 是否正在排队但并非必然丢弃请求，可以参考 `apiserver_flowcontrol_current_inqueue_requests` 指标:
```
% kubectl get --raw /metrics | grep 'apiserver_flowcontrol_current_inqueue_requests.*workload-low'
apiserver_flowcontrol_current_inqueue_requests{flow_schema="service-accounts",priority_level="workload-low"} 10
```


其他有用的 Prometheus 指标包括:

- apiserver_flowcontrol_dispatched_requests_total
- apiserver_flowcontrol_request_execution_seconds
- apiserver_flowcontrol_request_wait_duration_seconds

有关 APF 指标的完整列表,请参见上游文档[APF 指标](https://kubernetes.io/docs/concepts/cluster-administration/flow-control/#observability)。

### 防止丢弃请求

#### 通过更改您的工作负载来防止 429 错误

当 APF 由于给定的 PriorityLevelConfiguration 超过其允许的最大并发请求数而丢弃请求时,受影响的 FlowSchemas 中的客户端可以减少在给定时间内执行的请求数量。这可以通过减少在出现 429 错误的时期内发出的总请求数来实现。请注意,长时间运行的请求(如昂贵的列表调用)尤其有问题,因为它们在整个执行时间内都被计为一个并发请求。减少这些昂贵请求的数量或优化这些列表调用的延迟(例如,通过减少每个请求获取的对象数量或切换到使用 watch 请求)可以帮助减少给定工作负载所需的总并发性。

#### 通过更改 APF 设置来防止 429 错误

!!! Warning
    只有在您知道自己在做什么的情况下,才应该更改默认的 APF 设置。配置错误的 APF 设置可能会导致 API 服务器请求被丢弃,并造成严重的工作负载中断。

防止丢弃请求的另一种方法是更改在 EKS 集群上安装的默认 FlowSchemas 或 PriorityLevelConfigurations。EKS 会安装给定 Kubernetes 次版本的上游默认设置。除非在这些对象上设置了以下注释为 false,否则 API 服务器将自动将这些对象恢复为默认设置:
```
  metadata:
    annotations:
      apf.kubernetes.io/autoupdate-spec: "false"
```


在高层次上,可以修改 APF 设置来实现以下目标:

- 为您关心的请求分配更多的 inflight 容量。
- 隔离非必要或昂贵的请求,以免占用其他请求类型的容量。

这可以通过更改默认的 FlowSchemas 和 PriorityLevelConfigurations,或者创建这些类型的新对象来实现。运营商可以增加相关 PriorityLevelConfigurations 对象的 assuredConcurrencyShares 值,以增加分配给它们的 inflight 请求份额。此外,如果应用程序能够处理请求排队带来的额外延迟,也可以增加在给定时间内可以排队的请求数量。

另一种方法是创建特定于客户工作负载的新 FlowSchema 和 PriorityLevelConfigurations 对象。请注意,增加现有 PriorityLevelConfigurations 或新 PriorityLevelConfigurations 的 assuredConcurrencyShares 将导致其他 bucket 可以处理的请求数量减少,因为总体限制仍为每个 API 服务器 600 个 inflight 请求。

在对 APF 默认值进行更改时,应在非生产集群上监控这些指标,以确保更改设置不会导致意外的 429 错误:

1. 应监控所有 FlowSchemas 的 `apiserver_flowcontrol_rejected_requests_total` 指标,以确保没有 bucket 开始丢弃请求。
2. 应比较 `apiserver_flowcontrol_request_concurrency_limit` 和 `apiserver_flowcontrol_request_concurrency_in_use` 的值,以确保该优先级别的并发使用量不会超过限制。

定义新的 FlowSchema 和 PriorityLevelConfiguration 的一个常见用例是隔离。假设我们想将来自 pod 的长时间运行的列表事件调用隔离到自己的请求份额。这将防止使用现有 service-accounts FlowSchema 的重要请求受到 429 错误的影响,并被请求容量饿死。请记住,总的 inflight 请求数量是有限的,但是这个示例显示了如何修改 APF 设置来更好地划分给定工作负载的请求容量:

隔离列表事件请求的示例 FlowSchema 对象:
```
apiVersion: flowcontrol.apiserver.k8s.io/v1beta1
kind: FlowSchema
metadata:
  name: list-events-default-service-accounts
spec:
  distinguisherMethod:
    type: ByUser
  matchingPrecedence: 8000
  priorityLevelConfiguration:
    name: catch-all
  rules:
  - resourceRules:
    - apiGroups:
      - '*'
      namespaces:
      - default
      resources:
      - events
      verbs:
      - list
    subjects:
    - kind: ServiceAccount
      serviceAccount:
        name: default
        namespace: default
```


- 此 FlowSchema 捕获了默认命名空间中由服务帐户发起的所有列表事件调用。
- 匹配优先级 8000 低于现有 service-accounts FlowSchema 使用的值 9000，因此这些列表事件调用将匹配 list-events-default-service-accounts 而不是 service-accounts。
- 我们正在使用通用 PriorityLevelConfiguration 来隔离这些请求。此存储桶仅允许这些长时间运行的列表事件调用使用 13 个并发请求。当尝试发出超过 13 个并发请求时，Pods 将立即开始收到 429 错误。

## 在 API 服务器中检索资源

从 API 服务器获取信息是任何规模集群的预期行为。随着集群中资源数量的增加，请求频率和数据量可能会迅速成为控制平面的瓶颈,并导致 API 延迟和缓慢。根据延迟的严重程度,如果不小心,可能会导致意外的停机。

了解您正在请求的内容以及请求频率是避免这类问题的第一步。以下是基于扩展最佳实践来限制查询量的指南。本节中的建议按从已知最佳扩展开始的顺序提供。

### 使用共享 Informer

在构建与 Kubernetes API 集成的控制器和自动化时,您通常需要从 Kubernetes 资源获取信息。如果您定期轮询这些资源,可能会对 API 服务器造成重大负载。

使用 client-go 库中的 [informer](https://pkg.go.dev/k8s.io/client-go/informers) 将让您获得基于事件而不是轮询变更的好处。Informer 进一步通过使用事件和变更的共享缓存来减少负载,因此多个监视相同资源的控制器不会增加额外的负载。

控制器应避免轮询没有标签和字段选择器的集群范围资源,特别是在大型集群中。每个未过滤的轮询都需要从 etcd 通过 API 服务器发送大量不必要的数据,然后由客户端进行过滤。通过基于标签和命名空间进行过滤,可以减少 API 服务器需要执行的工作量以及发送到客户端的数据量。

### 优化 Kubernetes API 使用

使用自定义控制器或自动化调用 Kubernetes API 时,重要的是您只限制调用所需的资源。如果没有限制,您可能会对 API 服务器和 etcd 造成不必要的负载。

建议您尽可能使用 watch 参数。没有参数时,默认行为是列出对象。要使用 watch 而不是列表,您可以将 `?watch=true` 附加到 API 请求的末尾。例如,要获取默认命名空间中的所有 Pods 并进行监视,可以使用:
```
/api/v1/namespaces/default/pods?watch=true
```


如果您正在列出对象,您应该限制所列对象的范围和返回的数据量。您可以通过在请求中添加 `limit=500` 参数来限制返回的数据。`fieldSelector` 参数和 `/namespace/` 路径可以帮助确保您的列表范围尽可能窄。例如,要仅列出默认命名空间中正在运行的 Pod,可以使用以下 API 路径和参数。
```
/api/v1/namespaces/default/pods?fieldSelector=status.phase=Running&limit=500
```

或使用以下命令列出所有正在运行的 pod:

    
```
/api/v1/pods?fieldSelector=status.phase=Running&limit=500
```

另一个限制观看调用或列出对象的选项是使用 `resourceVersions`，您可以在 Kubernetes 文档中阅读相关内容。没有 `resourceVersion` 参数，您将收到最新可用版本，这需要 etcd 法定读取，这是数据库最昂贵和最慢的读取。resourceVersion 取决于您尝试查询的资源，可以在 `metadata.resourseVersion` 字段中找到。在使用观察调用而不仅仅是列表调用的情况下，也建议使用这种方法。

还有一个特殊的 `resourceVersion=0` 可用，它将从 API 服务器缓存返回结果。这可以减少 etcd 负载，但不支持分页。
```
/api/v1/namespaces/default/pods?resourceVersion=0
```


建议使用 watch 并将 resourceVersion 设置为从前一个列表或 watch 接收的最新已知值。这在 client-go 中会自动处理。但如果您在其他语言中使用 k8s 客户端,建议您再次检查。
```
/api/v1/namespaces/default/pods?watch=true&resourceVersion=362812295
```

如果您在不带任何参数的情况下调用 API，这将是对 API 服务器和 etcd 最资源密集的操作。此调用将获取所有命名空间中的所有 Pods，而不进行分页或限制范围，并需要从 etcd 进行法定读取。
```
/api/v1/pods
```

