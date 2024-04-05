!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

# 运行高可用性应用程序

您的客户希望您的应用程序始终可用,包括在您进行更改时,尤其是在流量激增期间。可扩展和有弹性的架构可以使您的应用程序和服务在不中断的情况下运行,从而保持用户满意。可扩展的基础设施会根据业务需求进行扩大和缩小。消除单点故障是提高应用程序可用性和使其具有弹性的关键步骤。

使用Kubernetes,您可以操作您的应用程序并以高可用性和弹性的方式运行它们。其声明式管理确保一旦您设置了应用程序,Kubernetes将不断尝试[将当前状态与所需状态匹配](https://kubernetes.io/docs/concepts/architecture/controller/#desired-vs-current)。

## 建议

### 避免运行单个Pod

如果您的整个应用程序在单个Pod中运行,那么如果该Pod被终止,您的应用程序将无法使用。不要使用单个pod部署应用程序,而是创建[Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)。如果Deployment创建的Pod失败或被终止,Deployment[控制器](https://kubernetes.io/docs/concepts/architecture/controller/)将启动一个新的Pod,以确保始终运行指定数量的副本Pod。

### 运行多个副本

使用Deployment运行多个副本Pod有助于以高可用性的方式运行应用程序。如果一个副本失败,其余副本仍将正常运行,尽管容量会有所降低,直到Kubernetes创建另一个Pod来弥补损失。此外,您可以使用[Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)根据工作负载需求自动扩展副本。

### 在节点上调度副本

如果所有副本都在同一个节点上运行,并且该节点变得不可用,那么运行多个副本就没有太大用处。考虑使用pod反亲和性或pod拓扑传播约束将Deployment的副本分散在多个工作节点上。

您还可以通过在多个可用性区域(AZ)中运行应用程序来进一步提高典型应用程序的可靠性。

#### 使用Pod反亲和性规则

下面的清单告诉Kubernetes调度程序*首选*将pod放在不同的节点和AZ上。它不要求不同的节点或AZ,因为如果这样做,一旦每个AZ都有一个pod运行,Kubernetes就无法再调度任何pod。如果您的应用程序只需要三个副本,您可以对`topologyKey: topology.kubernetes.io/zone`使用`requiredDuringSchedulingIgnoredDuringExecution`,Kubernetes调度程序将不会在同一AZ中调度两个pod。
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spread-host-az
  labels:
    app: web-server
spec:
  replicas: 4
  selector:
    matchLabels:
      app: web-server
  template:
    metadata:
      labels:
        app: web-server
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - web-server
              topologyKey: topology.kubernetes.io/zone
            weight: 100
          - podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - web-server
              topologyKey: kubernetes.io/hostname 
            weight: 99
      containers:
      - name: web-app
        image: nginx:1.16-alpine
```

使用 Pod 拓扑传播约束

与 pod 反亲和性规则类似,pod 拓扑传播约束允许您在不同的故障(或拓扑)域(如主机或 AZ)中分散您的应用程序。当您试图通过在不同的拓扑域中拥有多个副本来确保容错性和可用性时,这种方法效果很好。另一方面,pod 反亲和性规则很容易产生一个结果,即在一个拓扑域中只有一个副本,因为彼此具有反亲和性的 pod 会产生排斥效果。在这种情况下,在专用节点上只有一个副本对于容错性来说并不理想,也不是资源的良好利用。使用拓扑传播约束,您可以更好地控制调度器应该尝试在拓扑域之间应用的传播或分布。以下是这种方法中一些重要的属性:

1. `maxSkew` 用于控制或确定拓扑域之间不平衡的最大程度。例如,如果一个应用程序有 10 个副本并部署在 3 个 AZ 中,您无法实现均匀的传播,但您可以影响分布的不平衡程度。在这种情况下,`maxSkew` 可以是 1 到 10 之间的任何值。值为 1 意味着您最终可能会得到类似 `4,3,3`、`3,4,3` 或 `3,3,4` 的传播。相比之下,值为 10 意味着您最终可能会得到类似 `10,0,0`、`0,10,0` 或 `0,0,10` 的传播。

2. `topologyKey` 是节点标签之一的键,定义了应该用于 pod 分布的拓扑域类型。例如,区域传播将具有以下键值对:
```
topologyKey: "topology.kubernetes.io/zone"
```

3. `whenUnsatisfiable`属性用于确定当无法满足所需约束时,您希望调度程序如何响应。
4. `labelSelector`用于查找匹配的 pod,以便调度程序在决定将 pod 放置在哪里时,可以了解它们并根据您指定的约束进行安排。

除了上述这些,您还可以在 [Kubernetes 文档](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)中进一步了解其他字段。

![Pod 拓扑分散约束跨 3 个 AZ](./images/pod-topology-spread-constraints.jpg)
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spread-host-az
  labels:
    app: web-server
spec:
  replicas: 10
  selector:
    matchLabels:
      app: web-server
  template:
    metadata:
      labels:
        app: web-server
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: "topology.kubernetes.io/zone"
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: express-test
      containers:
      - name: web-app
        image: nginx:1.16-alpine
```


### 运行 Kubernetes 指标服务器

安装 Kubernetes [指标服务器](https://github.com/kubernetes-sigs/metrics-server)以帮助扩展您的应用程序。Kubernetes 自动缩放器附加组件如 [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) 和 [VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) 需要跟踪应用程序的指标才能对其进行扩展。指标服务器收集可用于做出扩展决策的资源指标。这些指标是从 kubelet 收集的,并以 [Metrics API 格式](https://github.com/kubernetes/metrics)提供。

指标服务器不保留任何数据,也不是监控解决方案。它的目的是向其他系统公开 CPU 和内存使用指标。如果您想随时间跟踪应用程序的状态,您需要一个监控工具,如 Prometheus 或 Amazon CloudWatch。

按照 [EKS 文档](https://docs.aws.amazon.com/eks/latest/userguide/metrics-server.html)在您的 EKS 集群中安装指标服务器。

## 水平 Pod 自动缩放器 (HPA)

HPA 可以根据需求自动扩展您的应用程序,并帮助您在高峰流量期间避免影响您的客户。它作为 Kubernetes 中的一个控制循环实现,定期从提供资源指标的 API 查询指标。

HPA 可以从以下 API 检索指标:
1. `metrics.k8s.io`(也称为资源指标 API) - 提供 pod 的 CPU 和内存使用情况
2. `custom.metrics.k8s.io` - 提供来自其他指标收集器(如 Prometheus)的指标;这些指标是__内部__于您的 Kubernetes 集群。
3. `external.metrics.k8s.io` - 提供__外部__于您的 Kubernetes 集群的指标(例如,SQS 队列深度、ELB 延迟)。

您必须使用这三个 API 中的一个来提供用于扩展应用程序的指标。

### 基于自定义或外部指标扩展应用程序

您可以使用自定义或外部指标来根据 CPU 或内存利用率以外的指标扩展应用程序。[自定义指标](https://github.com/kubernetes-sigs/custom-metrics-apiserver) API 服务器提供 `custom-metrics.k8s.io` API,HPA 可以使用它来自动扩展应用程序。

您可以使用 [Prometheus Adapter for Kubernetes Metrics APIs](https://github.com/directxman12/k8s-prometheus-adapter) 从 Prometheus 收集指标,并将其与 HPA 一起使用。在这种情况下,Prometheus 适配器将以 [Metrics API 格式](https://github.com/kubernetes/metrics/blob/master/pkg/apis/metrics/types.go)公开 Prometheus 指标。

部署 Prometheus 适配器后,您可以使用 kubectl 查询自定义指标。
`kubectl get —raw /apis/custom.metrics.k8s.io/v1beta1/`

外部指标顾名思义,为水平 Pod 自动缩放器提供了使用集群外指标来扩缩部署的能力。例如,在批处理工作负载中,通常会根据 SQS 队列中正在进行的作业数量来扩缩副本数量。

要使用 CloudWatch 指标自动扩缩部署,例如[根据 SQS 队列深度扩缩批处理处理器应用程序](https://github.com/awslabs/k8s-cloudwatch-adapter/blob/master/samples/sqs/README.md),您可以使用 [`k8s-cloudwatch-adapter`](https://github.com/awslabs/k8s-cloudwatch-adapter)。`k8s-cloudwatch-adapter` 是一个社区项目,不由 AWS 维护。

## 垂直 Pod 自动缩放器 (VPA)

VPA 自动调整 Pod 的 CPU 和内存预留,以帮助您"合理调整"应用程序的大小。对于需要垂直扩缩 - 通过增加资源分配来完成 - 的应用程序,您可以使用 [VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) 自动扩缩 Pod 副本或提供扩缩建议。

如果 VPA 需要扩缩您的应用程序,您的应用程序可能会暂时不可用,因为 VPA 的当前实现不会对 Pod 进行就地调整;相反,它将重新创建需要扩缩的 Pod。

[EKS 文档](https://docs.aws.amazon.com/eks/latest/userguide/vertical-pod-autoscaler.html)包含设置 VPA 的演练。

[Fairwinds Goldilocks](https://github.com/FairwindsOps/goldilocks/) 项目提供了一个仪表板,用于可视化 VPA 对 CPU 和内存请求以及限制的建议。它的 VPA 更新模式允许您根据 VPA 建议自动扩缩 Pod。

## 更新应用程序

现代应用程序需要高度稳定和可用性的快速创新。Kubernetes 为您提供了工具,可以持续更新应用程序,而不会中断您的客户。

让我们看看一些最佳实践,使您能够快速部署更改,而不牺牲可用性。

### 有一种机制来执行回滚

有一个撤消按钮可以避免灾难。在更新生产集群之前,在单独的较低环境(测试或开发环境)中测试部署是一种最佳实践。使用 CI/CD 管道可以帮助您自动化和测试部署。使用持续部署管道,如果升级出现故障,您可以快速恢复到旧版本。

您可以使用部署来更新正在运行的应用程序。这通常通过更新容器镜像来完成。您可以使用 `kubectl` 像这样更新部署:
```bash
kubectl --record deployment.apps/nginx-deployment set image nginx-deployment nginx=nginx:1.16.1
```


在更新需要重新创建 Pods 的 Deployment 时，Deployment 将执行[滚动更新](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/)。换句话说，Kubernetes 只会更新 Deployment 中正在运行的 Pods 的一部分，而不是一次更新所有 Pods。您可以通过 `RollingUpdateStrategy` 属性控制 Kubernetes 执行滚动更新的方式。

在执行 Deployment 的*滚动更新*时，您可以使用 [`Max Unavailable`](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#max-unavailable) 属性指定在更新过程中最多可以有多少个 Pods 处于不可用状态。Deployment 的 `Max Surge` 属性允许您设置可以创建的超出期望 Pods 数量的最大 Pods 数。

考虑调整 `max unavailable` 以确保滚动更新不会中断您的客户。例如，Kubernetes 默认设置 `max unavailable` 为 25%，这意味着如果您有 100 个 Pods，在滚动更新期间可能只有 75 个 Pods 处于活跃状态。如果您的应用程序需要至少 80 个 Pods，这种滚动更新可能会造成中断。相反，您可以将 `max unavailable` 设置为 20%，以确保在整个滚动更新过程中至少有 80 个功能性 Pods。

### 使用蓝/绿部署

变更本质上是有风险的,但无法撤销的变更可能会造成灾难性后果。允许您通过*回滚*有效地回到过去的变更程序可以使增强和实验更加安全。蓝/绿部署为您提供了一种快速撤回变更的方法,如果出现问题。在这种部署策略中,您会为新版本创建一个环境。这个环境与正在更新的应用程序的当前版本完全相同。一旦新环境配置完成,流量就会被路由到新环境。如果新版本产生了预期的结果而没有产生错误,就可以终止旧环境。否则,流量将恢复到旧版本。

您可以在 Kubernetes 中通过创建一个与现有版本的 Deployment 完全相同的新 Deployment 来执行蓝/绿部署。一旦您验证了新 Deployment 中的 Pods 正在运行且没有错误,您就可以通过更改路由流量到您应用程序 Pods 的 Service 的 `selector` 规范来开始将流量发送到新 Deployment。

许多持续集成工具,如 [Flux](https://fluxcd.io)、[Jenkins](https://www.jenkins.io) 和 [Spinnaker](https://spinnaker.io)，都允许您自动执行蓝/绿部署。AWS Containers Blog 包含了一个使用 AWS Load Balancer Controller 的演练: [使用 AWS Load Balancer Controller 进行蓝/绿部署、金丝雀部署和 A/B 测试](https://aws.amazon.com/blogs/containers/using-aws-load-balancer-controller-for-blue-green-deployment-canary-deployment-and-a-b-testing/)

### 使用金丝雀部署

金丝雀部署是蓝/绿部署的一种变体,可以大大降低变更的风险。在这种部署策略中,您会创建一个新的拥有较少 Pod 的 Deployment 与您的旧 Deployment 并行,并将少量流量转移到新的 Deployment。如果指标表明新版本的性能与现有版本相同或更好,您可以逐步增加流量到新的 Deployment 并扩大其规模,直到所有流量都转移到新的 Deployment。如果出现问题,您可以将所有流量路由到旧的 Deployment 并停止向新的 Deployment 发送流量。

尽管 Kubernetes 没有提供原生的执行金丝雀部署的方式,但您可以使用诸如 [Flagger](https://github.com/weaveworks/flagger) 与 [Istio](https://docs.flagger.app/tutorials/istio-progressive-delivery) 或 [App Mesh](https://docs.flagger.app/install/flagger-install-on-eks-appmesh) 等工具。

## 健康检查和自我修复

没有任何软件是无bug的,但 Kubernetes 可以帮助您最小化软件故障的影响。过去,如果应用程序崩溃,需要有人手动重新启动应用程序来修复问题。Kubernetes 使您能够检测 Pod 中的软件故障,并自动用新的副本替换它们。通过 Kubernetes,您可以监控应用程序的健康状况,并自动替换不健康的实例。

Kubernetes 支持三种类型的[健康检查](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/):

1. 存活探针
2. 启动探针(Kubernetes 1.16+版本支持)
3. 就绪探针

[Kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)是 Kubernetes 代理,负责运行上述所有检查。Kubelet 可以通过三种方式检查 Pod 的健康状况:Kubelet 可以在 Pod 容器内运行 shell 命令,向其容器发送 HTTP GET 请求,或在指定端口上打开 TCP 套接字。

如果选择基于 `exec` 的探针,它在容器内运行 shell 脚本,请确保 shell 命令在 `timeoutSeconds` 值到期之前退出。否则,您的节点将出现 `<defunct>` 进程,导致节点故障。

## 建议
### 使用存活探针删除不健康的 Pod

应用程序启动时间较长时,可以使用启动探针(Startup Probe)来延迟存活探针(Liveness Probe)和就绪探针(Readiness Probe)的执行。例如,一个需要从数据库中加载缓存的Java应用程序可能需要长达两分钟的时间才能完全启动。在应用程序完全启动之前,任何存活探针或就绪探针都可能会失败。配置启动探针将允许Java应用程序在执行存活探针或就绪探针之前达到"健康"状态。

在启动探针成功之前,所有其他探针都将被禁用。您可以定义Kubernetes应该等待应用程序启动的最长时间。如果在最大配置时间后,Pod仍然无法通过启动探针,它将被终止,并创建一个新的Pod。

启动探针类似于存活探针 - 如果它们失败,Pod将被重新创建。正如Ricardo A.在他的文章[Fantastic Probes And How To Configure Them](https://medium.com/swlh/fantastic-probes-and-how-to-configure-them-fef7e030bd2f)中所解释的,当应用程序的启动时间不可预测时,应该使用启动探针。如果您知道您的应用程序需要10秒钟才能启动,您应该使用带有`initialDelaySeconds`的存活探针/就绪探针。

当 Liveness 探针检测到应用程序中的故障并通过终止 Pod（因此重新启动应用程序）来解决这些故障时,Readiness 探针检测应用程序可能暂时无法使用的情况。在这些情况下,应用程序可能暂时无响应;但是,预计在此操作完成后,应用程序将再次健康。

例如,在磁盘 I/O 操作密集的情况下,应用程序可能暂时无法处理请求。在这里,终止应用程序的 Pod 并不是一个补救措施;同时,发送到 Pod 的其他请求也可能失败。

您可以使用 Readiness 探针来检测应用程序的临时不可用性,并在其再次可用之前停止向其 Pod 发送请求。*与 Liveness 探针不同,Liveness 探针的失败会导致 Pod 被重新创建,而 Readiness 探针的失败意味着 Pod 将不会从 Kubernetes 服务接收任何流量*。当 Readiness 探针成功时,Pod 将恢复接收来自服务的流量。

与 Liveness 探针一样,请避免配置依赖于 Pod 外部资源(如数据库)的 Readiness 探针。以下是一个配置不当的 Readiness 探针可能导致应用程序无法正常运行的场景 - 如果 Pod 的 Readiness 探针在应用程序的数据库无法访问时失败,其他 Pod 副本也会同时失败,因为它们共享相同的健康检查标准。以这种方式设置探针将确保在数据库不可用时,Pod 的 Readiness 探针将失败,Kubernetes 将停止向所有 Pod 发送流量。

使用 Readiness 探针的一个副作用是,它们可能会增加更新部署所需的时间。新的副本将不会收到流量,除非 Readiness 探针成功;在此之前,旧的副本将继续接收流量。

---

## 处理中断

Pod 有有限的生命周期 - 即使您有长期运行的 Pod,在 Pod 终止时也应确保它们正确终止。根据您的升级策略,Kubernetes 集群升级可能需要您创建新的工作节点,这需要所有 Pod 在较新的节点上重新创建。适当的终止处理和 Pod 中断预算可以帮助您在从旧节点删除 Pod 并在较新节点上重新创建时避免服务中断。
升级工作节点的首选方式是创建新的工作节点并终止旧的工作节点。在终止工作节点之前,您应该`排空`它。当工作节点被排空时,其所有 pod 都会被*安全地*驱逐。"安全"是这里的关键词;当 pod 上的容器被驱逐时,它们不会简单地收到 `SIGKILL` 信号。相反,会向每个 pod 中容器的主进程(PID 1)发送 `SIGTERM` 信号。发送 `SIGTERM` 信号后,Kubernetes 会给进程一些时间(宽限期),然后再发送 `SIGKILL` 信号。默认宽限期为 30 秒;您可以使用 kubectl 中的 `grace-period` 标志或在 Podspec 中声明 `terminationGracePeriodSeconds` 来覆盖默认值。

`kubectl delete pod <pod 名称> —grace-period=<秒数>`

在容器的主进程不是 PID 1 的情况下很常见。以下是一个基于 Python 的示例容器:
```
$ kubectl exec python-app -it ps
 PID USER TIME COMMAND
 1   root 0:00 {script.sh} /bin/sh ./script.sh
 5   root 0:00 python app.py
```

在此示例中，shell 脚本接收到 `SIGTERM`，主进程（在本例中是 Python 应用程序）没有收到 `SIGTERM` 信号。当 Pod 被终止时，Python 应用程序将被突然杀死。这可以通过将容器的 [`ENTRYPOINT`](https://docs.docker.com/engine/reference/builder/#entrypoint) 更改为启动 Python 应用程序来解决。或者，您可以使用像 [dumb-init](https://github.com/Yelp/dumb-init) 这样的工具来确保您的应用程序可以处理信号。

您还可以使用 [Container hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/#container-hooks) 在容器启动或停止时执行脚本或 HTTP 请求。`PreStop` 钩子操作在容器收到 `SIGTERM` 信号之前运行，并且必须在发送此信号之前完成。`terminationGracePeriodSeconds` 值从 `PreStop` 钩子操作开始执行时开始计算，而不是从发送 `SIGTERM` 信号时开始计算。

## 建议

### 使用 Pod 中断预算保护关键工作负载

Pod 中断预算或 PDB 可以暂时阻止驱逐过程，如果应用程序的副本数量低于声明的阈值。一旦可用副本数量超过阈值，驱逐过程将继续。您可以使用 PDB 来声明 `minAvailable` 和 `maxUnavailable` 副本数量。例如，如果您希望至少有三个应用程序副本可用，您可以创建一个 PDB。
```
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: my-svc-pdb
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: my-svc
```


服务网格

上述 PDB 策略告诉 Kubernetes 在至少有三个副本可用之前暂停驱逐进程。节点排空过程尊重 `PodDisruptionBudgets`。在 EKS 托管节点组升级期间, [节点会在 15 分钟超时后被排空](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-update-behavior.html)。如果更新未强制执行(在 EKS 控制台中称为滚动更新),则更新失败。如果强制执行更新,则会删除 pod。

对于自管理节点,您还可以使用 [AWS Node Termination Handler](https://github.com/aws/aws-node-termination-handler) 等工具,该工具确保 Kubernetes 控制平面适当响应可能导致 EC2 实例无法使用的事件,例如 [EC2 维护](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-instances-status-check_sched.html)事件和 [EC2 Spot 中断](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-interruptions.html)。它使用 Kubernetes API 将节点隔离,以确保不会调度新的 Pod,然后对其进行排空,终止任何正在运行的 Pod。

您可以使用 Pod 反亲和性来调度部署的 Pod 在不同节点上,从而避免节点升级期间与 PDB 相关的延迟。

### 实践混沌工程
> 混沌工程是在分布式系统上进行实验的学科,目的是建立对系统在生产环境中承受动荡条件的能力的信心。

在他的博客中,Dominik Tornow 解释说,[Kubernetes 是一个声明式系统](https://medium.com/@dominik.tornow/the-mechanics-of-kubernetes-ac8112eaa302),其中"*用户向系统提供系统所需状态的表示。系统然后考虑当前状态和所需状态,以确定从当前状态过渡到所需状态的一系列命令。*"这意味着 Kubernetes 始终存储*所需状态*,如果系统偏离,Kubernetes 将采取行动来恢复状态。例如,如果工作节点不可用,Kubernetes 将重新调度 Pod 到另一个工作节点。同样,如果一个 `replica` 崩溃,[部署控制器](https://kubernetes.io/docs/concepts/architecture/controller/#design)将创建一个新的 `replica`。通过这种方式,Kubernetes 控制器会自动修复故障。

混沌工程工具,如 [Gremlin](https://www.gremlin.com),可帮助您测试 Kubernetes 集群的弹性,并识别单点故障。在集群(及其他地方)引入人为混沌的工具可以发现系统性弱点,提供识别瓶颈和错误配置的机会,并在受控环境中纠正问题。混沌工程哲学主张有目的地破坏事物,并对基础设施进行压力测试,以最大限度地减少意外停机。

您可以使用服务网格来提高应用程序的弹性。服务网格可以实现服务间通信,并增加微服务网络的可观察性。大多数服务网格产品都是通过在每个服务旁运行一个小型网络代理来实现的,该代理会拦截和检查应用程序的网络流量。您可以在不修改应用程序的情况下将其置于网格中。利用服务代理的内置功能,您可以生成网络统计数据、创建访问日志,并为分布式跟踪添加HTTP头。

服务网格可以通过自动重试请求、超时、断路器和限流等功能,帮助您使微服务更加有弹性。

如果您运营多个集群,您可以使用服务网格来实现跨集群的服务间通信。

### 服务网格
+ [AWS App Mesh](https://aws.amazon.com/app-mesh/)
+ [Istio](https://istio.io)
+ [LinkerD](http://linkerd.io)
+ [Consul](https://www.consul.io)

---

## 可观察性

可观察性是一个包括监控、日志和跟踪的总称。基于微服务的应用程序本质上是分布式的。与监控单一系统就足够的单体应用程序不同,在分布式应用程序架构中,您需要监控每个组件的性能。您可以使用集群级监控、日志和分布式跟踪系统来识别集群中的问题,以便在它们影响您的客户之前解决。

Kubernetes内置的故障排查和监控工具有限。metrics-server收集资源指标并将其存储在内存中,但不会持久化。您可以使用kubectl查看Pod的日志,但Kubernetes不会自动保留日志。分布式跟踪的实现要么在应用程序代码级别,要么使用服务网格。

Kubernetes的可扩展性在这里发挥作用。Kubernetes允许您引入首选的集中式监控、日志和跟踪解决方案。

## 建议

### 监控您的应用程序

现代应用程序需要监控的指标数量不断增加。如果您有一种自动跟踪应用程序的方法,就可以专注于解决客户的挑战。像[Prometheus](https://prometheus.io)或[CloudWatch Container Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights.html)这样的集群范围监控工具可以监控您的集群和工作负载,并在出现问题时或更好的是在问题发生之前向您发出信号。

监控工具允许您创建警报,供您的运营团队订阅。考虑制定规则,以在可能导致停机或影响应用程序性能的事件发生时触发警报。

如果您不确定应该监控哪些指标,可以从以下方法中获取灵感:

- [RED方法](https://www.weave.works/blog/a-practical-guide-from-instrumenting-code-to-specifying-alerts-with-the-red-method)。代表请求、错误和持续时间。
- [USE方法](http://www.brendangregg.com/usemethod.html)。代表利用率、饱和度和错误。

Sysdig的文章[Kubernetes最佳报警实践](https://sysdig.com/blog/alerting-kubernetes/)包含了一个全面的组件列表,这些组件可能会影响应用程序的可用性。

### 使用Prometheus客户端库公开应用程序指标

除了监控应用程序的状态和聚合标准指标外,您还可以使用[Prometheus客户端库](https://prometheus.io/docs/instrumenting/clientlibs/)公开特定于应用程序的自定义指标,以提高应用程序的可观察性。

### 使用集中式日志工具收集和持久化日志

EKS中的日志记录分为两类:控制平面日志和应用程序日志。EKS控制平面日志直接从控制平面提供审核和诊断日志到您帐户中的CloudWatch日志。应用程序日志是在集群内运行的Pod产生的日志。应用程序日志包括运行业务逻辑应用程序和Kubernetes系统组件(如CoreDNS、集群自动缩放器、Prometheus等)的Pod产生的日志。

[EKS提供五种类型的控制平面日志](https://docs.aws.amazon.com/eks/latest/userguide/control-plane-logs.html):

1. Kubernetes API服务器组件日志
2. 审核
3. 认证器
4. 控制器管理器
5. 调度器

控制器管理器和调度器日志可帮助诊断控制平面问题,如瓶颈和错误。默认情况下,EKS控制平面日志不会发送到CloudWatch日志。您可以启用控制平面日志记录,并选择要为帐户中的每个集群捕获的EKS控制平面日志类型。

收集应用程序日志需要在集群中安装日志聚合工具,如[Fluent Bit](http://fluentbit.io)、[Fluentd](https://www.fluentd.org)或[CloudWatch容器洞察](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/deploy-container-insights-EKS.html)。

Kubernetes日志聚合工具以DaemonSet的形式运行,并从节点中抓取容器日志。然后将应用程序日志发送到集中式目标进行存储。例如,CloudWatch容器洞察可以使用Fluent Bit或Fluentd收集日志并将其发送到CloudWatch日志进行存储。Fluent Bit和Fluentd支持许多流行的日志分析系统,如Elasticsearch和InfluxDB,使您能够通过修改Fluent bit或Fluentd的日志配置来更改日志的存储后端。

### 使用分布式跟踪系统识别瓶颈

一个典型的现代应用程序的组件分布在网络上,其可靠性取决于构成应用程序的每个组件的正常运行。您可以使用分布式跟踪解决方案来了解请求的流动方式以及系统之间的通信方式。跟踪可以向您显示应用程序网络中存在的瓶颈,并防止可能导致级联故障的问题。

您有两种选择来实现应用程序中的跟踪:您可以使用共享库在代码级别实现分布式跟踪,也可以使用服务网格。

在代码级别实现跟踪可能会有不利影响。在这种方法中,您必须对代码进行更改。如果您有多语言应用程序,这将更加复杂。您还需要负责维护另一个库,跨您的服务。

[LinkerD](http://linkerd.io)、[Istio](http://istio.io)和[AWS App Mesh](https://aws.amazon.com/app-mesh/)等服务网格可用于在应用程序中实现分布式跟踪,而无需对应用程序代码进行大量更改。您可以使用服务网格来标准化指标生成、日志记录和跟踪。

[AWS X-Ray](https://aws.amazon.com/xray/)、[Jaeger](https://www.jaegertracing.io)等跟踪工具支持共享库和服务网格实现。

考虑使用支持两种实现(共享库和服务网格)的跟踪工具,如[AWS X-Ray](https://aws.amazon.com/xray/)或[Jaeger](https://www.jaegertracing.io),这样如果您后来采用服务网格,就不必切换工具。
