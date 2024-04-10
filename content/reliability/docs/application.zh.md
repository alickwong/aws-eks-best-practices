# 运行高可用性应用程序

您的客户期望您的应用程序始终可用,包括在您进行更改时以及在流量激增期间。可扩展和有弹性的架构可以确保您的应用程序和服务在不中断的情况下运行,从而保持用户满意度。可扩展的基础设施根据业务需求进行扩展和收缩。消除单点故障是提高应用程序可用性和弹性的关键步骤。

使用 Kubernetes,您可以以高可用和有弹性的方式运行您的应用程序。其声明式管理确保一旦您设置了应用程序,Kubernetes 将不断尝试[将当前状态与所需状态匹配](https://kubernetes.io/docs/concepts/architecture/controller/#desired-vs-current)。

## 建议

### 避免运行单个 Pod

如果您的整个应用程序在单个 Pod 中运行,那么如果该 Pod 被终止,您的应用程序将无法使用。不要使用单个 Pod 部署应用程序,而是创建[部署](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)。如果由部署创建的 Pod 失败或被终止,部署[控制器](https://kubernetes.io/docs/concepts/architecture/controller/)将启动一个新的 Pod 以确保指定数量的副本 Pod 始终在运行。

### 运行多个副本

使用部署运行应用程序的多个副本 Pod 可以以高可用的方式运行它。如果一个副本失败,其余副本仍将正常运行,尽管容量会有所下降,直到 Kubernetes 创建另一个 Pod 来弥补损失。此外,您可以使用[水平 Pod 自动缩放器](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)根据工作负载需求自动扩展副本。

### 在节点上调度副本

如果所有副本都在同一个节点上运行,并且该节点变得不可用,运行多个副本就不会很有用。考虑使用 pod 反亲和性或 pod 拓扑传播约束将部署的副本分散在多个工作节点上。

您还可以通过在多个可用性区域(AZ)中运行应用程序来进一步提高典型应用程序的可靠性。

#### 使用 Pod 反亲和性规则

下面的清单告诉 Kubernetes 调度程序*首选*将 pod 放在不同的节点和 AZ 上。它不要求不同的节点或 AZ,因为如果这样做,一旦每个 AZ 中都有一个 pod 运行,Kubernetes 就无法再调度任何 pod。如果您的应用程序只需要三个副本,您可以对 `topologyKey: topology.kubernetes.io/zone` 使用 `requiredDuringSchedulingIgnoredDuringExecution`,Kubernetes 调度程序将不会在同一个 AZ 中调度两个 pod。

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

#### 使用 Pod 拓扑传播约束

与 pod 反亲和性规则类似,pod 拓扑传播约束允许您在不同的故障(或拓扑)域(如主机或 AZ)中分散您的应用程序。当您试图确保容错性和可用性时,这种方法效果很好,因为在不同的拓扑域中有多个副本。另一方面,pod 反亲和性规则很容易产生一个结果,即在一个拓扑域中只有一个副本,因为具有反亲和性的 pod 会产生排斥效果。在这种情况下,在专用节点上只有一个副本对于容错性来说并不理想,也不是资源的有效利用。使用拓扑传播约束,您可以更好地控制调度程序应该在拓扑域之间应用的传播或分布。以下是一些重要的属性:
1. `maxSkew` 用于控制或确定拓扑域之间不平衡的最大程度。例如,如果一个应用程序有 10 个副本,并部署在 3 个 AZ 上,您无法实现完全均匀的分布,但您可以影响分布的不平衡程度。在这种情况下,`maxSkew` 可以是 1 到 10 之间的任何值。值为 1 意味着您可能最终会得到类似 `4,3,3`、`3,4,3` 或 `3,3,4` 的分布。相比之下,值为 10 意味着您可能最终会得到类似 `10,0,0`、`0,10,0` 或 `0,0,10` 的分布。
2. `topologyKey` 是节点标签的一个键,定义了应该用于 pod 分布的拓扑域类型。例如,区域传播将具有以下键值对:
```
topologyKey: "topology.kubernetes.io/zone"
```
3. `whenUnsatisfiable` 属性用于确定如果无法满足所需的约束,您希望调度程序如何响应。
4. `labelSelector` 用于查找匹配的 pod,以便调度程序在根据您指定的约束决定放置 pod 时知道它们。

除了上述内容,您还可以进一步阅读 [Kubernetes 文档](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)了解其他字段。

![在 3 个 AZ 上的 Pod 拓扑传播约束](./images/pod-topology-spread-constraints.jpg)

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

### 运行 Kubernetes Metrics Server

安装 Kubernetes [Metrics Server](https://github.com/kubernetes-sigs/metrics-server) 来帮助扩展您的应用程序。Kubernetes 自动缩放器插件如 [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) 和 [VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) 需要跟踪应用程序的指标来对其进行扩展。Metrics Server 收集可用于做出扩展决策的资源指标。这些指标是从 kubelet 收集的,并以 [Metrics API 格式](https://github.com/kubernetes/metrics)提供。

Metrics Server 不保留任何数据,也不是监控解决方案。它的目的是向其他系统公开 CPU 和内存使用指标。如果您想随时间跟踪应用程序的状态,您需要一个监控工具,如 Prometheus 或 Amazon CloudWatch。

按照 [EKS 文档](https://docs.aws.amazon.com/eks/latest/userguide/metrics-server.html)在您的 EKS 集群中安装 Metrics Server。

## 水平 Pod 自动缩放器 (HPA)

HPA 可以根据需求自动扩展您的应用程序,并帮助您在高峰流量期间避免影响客户。它是 Kubernetes 中的一个控制循环,定期从提供资源指标的 API 查询指标。

HPA 可以从以下 API 检索指标:
1. `metrics.k8s.io` 也称为资源指标 API - 提供 pod 的 CPU 和内存使用情况
2. `custom.metrics.k8s.io` - 提供来自其他指标收集器(如 Prometheus)的指标;这些指标是 __内部__ 于您的 Kubernetes 集群。
3. `external.metrics.k8s.io` - 提供 __外部__ 于您的 Kubernetes 集群的指标(例如,SQS 队列深度,ELB 延迟)。

您必须使用这三个 API 之一来提供用于扩展应用程序的指标。

### 根据自定义或外部指标扩展应用程序

您可以使用自定义或外部指标根据 CPU 或内存利用率以外的指标来扩展您的应用程序。[自定义指标](https://github.com/kubernetes-sigs/custom-metrics-apiserver) API 服务器提供 `custom-metrics.k8s.io` API,HPA 可以使用它来自动扩展应用程序。

您可以使用 [Prometheus Adapter for Kubernetes Metrics APIs](https://github.com/directxman12/k8s-prometheus-adapter) 从 Prometheus 收集指标并与 HPA 一起使用。在这种情况下,Prometheus 适配器将以 [Metrics API 格式](https://github.com/kubernetes/metrics/blob/master/pkg/apis/metrics/types.go)公开 Prometheus 指标。

部署 Prometheus 适配器后,您可以使用 kubectl 查询自定义指标。
`kubectl get —raw /apis/custom.metrics.k8s.io/v1beta1/`

外部指标顾名思义,为水平 Pod 自动缩放器提供了使用集群外指标扩展部署的能力。例如,在批处理工作负载中,通常会根据正在进行的作业数量来扩展副本数量。

要使用 CloudWatch 指标自动扩展部署,例如[根据 SQS 队列深度扩展批处理处理器应用程序](https://github.com/awslabs/k8s-cloudwatch-adapter/blob/master/samples/sqs/README.md),您可以使用 [`k8s-cloudwatch-adapter`](https://github.com/awslabs/k8s-cloudwatch-adapter)。`k8s-cloudwatch-adapter` 是一个社区项目,不由 AWS 维护。

## 垂直 Pod 自动缩放器 (VPA)

VPA 自动调整 Pod 的 CPU 和内存预留,以帮助您"合理调整"应用程序。对于需要垂直扩展(通过增加资源分配来完成)的应用程序,您可以使用 [VPA](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) 自动扩展 Pod 副本或提供扩展建议。

如果 VPA 需要扩展您的应用程序,您的应用程序可能会暂时无法使用,因为 VPA 的当前实现不会执行对 Pod 的就地调整;相反,它将重新创建需要扩展的 Pod。

[EKS 文档](https://docs.aws.amazon.com/eks/latest/userguide/vertical-pod-autoscaler.html)包含设置 VPA 的演练。

[Fairwinds Goldilocks](https://github.com/FairwindsOps/goldilocks/) 项目提供了一个仪表板,用于可视化 VPA 对 CPU 和内存请求和限制的建议。它的 VPA 更新模式允许您根据 VPA 建议自动扩展 Pod。

## 更新应用程序

现代应用程序需要快速创新,同时具有高度的稳定性和可用性。Kubernetes 为您提供了工具,可以在不影响客户的情况下持续更新您的应用程序。

让我们看看一些最佳实践,使您能够快速部署更改而不牺牲可用性。

### 有一种回滚机制

有一个撤消按钮可以避免灾难。在将更新部署到生产集群之前,最佳做法是在单独的较低环境(测试或开发环境)中测试部署。使用 CI/CD 管道可以帮助您自动化和测试部署。使用持续部署管道,如果升级出现问题,您可以快速恢复到旧版本。

您可以使用部署来更新正在运行的应用程序。这通常通过更新容器镜像来完成。您可以使用 `kubectl` 像这样更新部署:

```bash
kubectl --record deployment.apps/nginx-deployment set image nginx-deployment nginx=nginx:1.16.1
```

`--record` 参数记录对部署的更改,如果您需要执行回滚,这将很有帮助。`kubectl rollout history deployment` 显示您集群中部署的记录更改。您可以使用 `kubectl rollout undo deployment <DEPLOYMENT_NAME>` 回滚更改。

默认情况下,当您更新需要重新创建 pod 的部署时,部署将执行[滚动更新](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/)。换句话说,Kubernetes 只会更新部署中正在运行的 pod 的一部分,而不是全部 pod。您可以通过 `RollingUpdateStrategy` 属性控制 Kubernetes 执行滚动更新的方式。

在执行部署的*滚动更新*时,您可以使用 [`Max Unavailable`](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#max-unavailable) 属性指定在更新期间最多可以有多少 Pod 不可用。部署的 `Max Surge` 属性允许您设置可以超过所需 Pod 数量的最大 Pod 数。

考虑调整 `max unavailable` 以确保滚动不会扰乱您的客户。例如,Kubernetes 默认设置 25% 的 `max unavailable`,这意味着如果您有 100 个 Pod,在滚动过程中可能只有 75 个 Pod 处于活动状态。如果您的应用程序需要至少 80 个 Pod,这种滚动可能会中断。相反,您可以将 `max unavailable` 设置为 20%,以确保在整个滚动过程中至少有 80 个功能性 Pod。

### 使用蓝/绿部署

变更本质上是有风险的,但无法撤消的变更可能会造成灾难性后果。允许您有效"回到过去"的变更过程使增强和试验更加安全。蓝/绿部署为您提供一种快速撤回更改的方法。在这种部署策略中,您为新版本创建一个环境。这个环境与当前版本的应用程序完全相同。一旦新环境配置好,就将流量路由到新环境。如果新版本产生了预期的结果而没有产生错误,就终止旧环境。否则,将流量恢复到旧版本。

您可以在 Kubernetes 中通过创建一个与现有版本的部署完全相同的新部署来执行蓝/绿部署。一旦您验证新部署中的 Pod 正在运行且没有错误,您就可以通过更改路由应用程序 Pod 的服务的 `selector` 规范来开始将流量发送到新部署。

许多持续集成工具,如 [Flux](https://fluxcd.io)、[Jenkins](https://www.jenkins.io) 和 [Spinnaker](https://spinnaker.io),都允许您自动化蓝/绿部署。AWS Containers Blog 包含一个使用 AWS Load Balancer Controller 的演练:[使用 AWS Load Balancer Controller 进行蓝/绿部署、金丝雀部署和 A/B 测试](https://aws.amazon.com/blogs/containers/using-aws-load-balancer-controller-for-blue-green-deployment-canary-deployment-and-a-b-testing/)

### 使用金丝雀部署

金丝雀部署是蓝/绿部署的一种变体,可以大大降低变更的风险。在这种部署策略中,您会创建一个副本数较少的新部署,与您的旧部署并行运行,并将少量流量转发到新部署。如果指标表明新版本的性能与现有版本一样好或更好,您就会逐步增加流量到新部署并扩展它,直到所有流量都转发到新部署。如果出现问题,您可以将所有流量路由回旧部署并停止向新部署发送流量。

尽管 Kubernetes 没有提供本机方式来执行金丝雀部署,但您可以使用诸如 [Flagger](https://github.com/weaveworks/flagger) 与 [Istio](https://docs.flagger.app/tutorials/istio-progressive-delivery) 或 [App Mesh](https://docs.flagger.app/install/flagger-install-on-eks-appmesh) 之类的工具。

## 健康检查和自我修复

没有软件是无错误的,但 Kubernetes 可以帮助您最小化软件故障的影响。过去,如果应用程序崩溃,必须由某人手动修复该情况,重新启动应用程序。Kubernetes 使您能够检测 Pod 中的软件故障,并自动用新副本替换它们。使用 Kubernetes,您可以监控应用程序的健康状况,并自动替换不健康的实例。

Kubernetes 支持三种类型的[健康检查](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/):

1. 存活探针
2. 启动探针(Kubernetes 1.16+支持)
3. 就绪探针

[Kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)是负责运行所有上述检查的 Kubernetes 代理。Kubelet 可以通过三种方式检查 Pod 的健康状况:kubelet 可以在 Pod 容器内运行 shell 命令,向其容器发送 HTTP GET 请求,或在指定端口上打开 TCP 套接字。

如果选择基于 `exec` 的探针,它在容器内运行 shell 脚本,请确保 shell 命令在 `timeoutSeconds` 值到期之前退出。否则,您的节点将有 `<defunct>` 进程,导致节点故障。

## 建议
### 使用存活探针删除不健康的 pod

存活探针可以检测*死锁*条件,即进程继续运行但应用程序变得无响应。例如,如果您正在运行一个监听端口 80 的 web 服务,您可以配置一个存活探针在 Pod 的端口 80 上发送 HTTP GET 请求。Kubelet 将定期向 Pod 发送 GET 请求并期望响应;如果 Pod 在 200-399 之间响应,则 kubelet 认为该 Pod 是健康的;否则,该 Pod 将被标记为不健康。如果 Pod 持续失败健康检查,kubelet 将终止它。

您可以使用 `initialDelaySeconds` 来延迟第一次探测。

使用存活探针时,请确保您的应用程序不会陷入所有 Pod 同时失败存活探针的情况,因为 Kubernetes 将尝试替换所有 Pod,这将使您的应用程序离线。此外,Kubernetes 将继续创建新的 Pod,这些 Pod 也将失败存活探针,给控制平面带来不必要的压力。避免将存活探针配置为依赖于 Pod 外部的因素,例如外部数据库。换句话说,对于不响应的外部于 Pod 的数据库,不应使 Pod 的存活探针失败。

Sandor Szücs 的文章[存活探针很危险](https://srcco.de/posts/kubernetes-liveness-probes-are-dangerous.html)描述了错误配置的探针可能造成的问题。

### 对于启动时间较长的应用程序,使用启动探针

当您的应用程序需要额外的启动时间时,您可以使用启动探针来延迟存活和就绪探针。例如,需要从数据库中加载缓存的 Java 应用程序可能需要长达两分钟的时间才能完全启动。在应用程序完全启动之前,任何存活或就绪探针都可能失败。配置启动探针将允许 Java 应用程序在执行其他探针之前变为*健康*状态。

在启动探针成功之前,所有其他探针都被禁用。您可以定义 Kubernetes 应该等待应用程序启动的最长时间。如果在最大配置时间后 Pod 仍然失败启动探针,它将被终止,并创建一个新的 Pod。

启动探针类似于存活探针 - 如果它们失败,Pod 将被重新创建。正如 Ricardo A. 在他的文章[神奇的探针及其配置](https://medium.com/swlh/fantastic-probes-and-how-to-configure-them-fef7e030bd2f)中解释的那样,当应用程序的启动时间不可预测时,应该使用启动探针。如果您知道您的应用程序需要十秒钟来启动,您应该使用带有 `initialDelaySeconds` 的存活/就绪探针。

### 使用就绪探针检测部分不可用

而存活探针检测应用程序中可通过终止 Pod(因此重新启动应用程序)来解决的故障,就绪探针检测应用程序可能*暂时*不可用的条件。在这些情况下,应用程序可能会暂时无法响应请求;但是,一旦此操作完成,应用程序预计将再次正常。

例如,在磁盘 I/O 操作期间,应用程序可能会暂时无法处理请求。在这里,终止应用程序的 Pod 并不是一个补救措施;与此同时,发送到其 Pod 的其他请求可能会失败。

您可以使用就绪探针来检测应用程序中的临时不可用性,并在它再次变得可用之前停止向其 Pod 发送流量。*与存活探针不同,失败的就绪探针会导致 Pod 不会从 Kubernetes 服务接收任何流量,而不是重新创建 Pod*。当就绪探针成功时,Pod 将恢复接收来自服务的流量。

与存活探针一样,避免配置依赖于 Pod 外部资源(如数据库)的就绪探针。这里有一个场景,一个配置不当的就绪探针可能会使应用程序无法正常工作 - 如果 Pod 的就绪探针在应用程序的数据库不可访问时失败,其他 Pod 副本也会同时失败,因为它们共享相同的健康检查标准。以这种方式设置探针将确保每当数据库不可用时,Pod 的就绪探针都会失败,Kubernetes 将停止向*所有* Pod 发送流量。

使用就绪探针的一个副作用是,它们可以增加更新部署所需的时间。新副本只有在就绪探针成功后才会接收流量;在此之前,旧副本将继续接收流量。

---

## 处理中断

Pod 有有限的生命周期 - 即使您有长期运行的 Pod,在适当的时候正确终止 Pod 也是明智的。根据您的升级策略,Kubernetes 集群升级可能需要您创建新的工作节点,这需要在新节点上重新创建所有 Pod。适当的终止处理和 Pod 中断预算可以帮助您在从旧节点删除 Pod 并在新节点上重新创建它们时避免服务中断。

升级工作节点的首选方式是创建新的工作节点并终止旧节点。在终止工作节点之前,您应该`排空`它。当工作节点被排空时,其所有 pod 都会*安全地*被驱逐。"安全"是一个关键词;当 pod 从工作节点被驱逐时,它们不会简单地收到 `SIGKILL` 信号。相反,会向 Pod 中每个容器的主进程(PID 1)发送 `SIGTERM` 信号。发送 `SIGTERM` 信号后,Kubernetes 会给该进程一些时间(宽限期)才发送 `SIGKILL` 信号。此宽限期默认为 30 秒;您可以通过在 kubectl 中使用 `grace-period` 标志或在 Podspec 中声明 `terminationGracePeriodSeconds` 来覆盖默认值。

`kubectl delete pod <pod name> —grace-period=<seconds>`

通常,容器中的主进程不是 PID 1。考虑这个基于 Python 的示例容器:

```
$ kubectl exec python-app -it ps
 PID USER TIME COMMAND
 1   root 0:00 {script.sh} /bin/sh ./script.sh
 5   root 0:00 python app.py
```

在这个例子中,shell 脚本接收 `SIGTERM`,但作为 Python 应用程序的主进程并没有收到 `SIGTERM` 信号。当 Pod 被终止时,Python 应用程序将被突然终止。通过将容器的 [`ENTRYPOINT`](https://docs.docker.com/engine/reference/builder/#entrypoint) 更改为启动 Python 应用程序,可以解决这个问题。或者,您可以使用像 [dumb-init](https://github.com/Yelp/dumb-init) 这样的工具来确保您的应用程序可以处理信号。

您还可以使用[容器钩子](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/#container-hooks)在容器启动或停止时执行脚本或 HTTP 请求。`PreStop` 钩子操作在容器收到 `SIGTERM` 信号之前运行,必须在发送此信号之前完成。`terminationGracePeriodSeconds` 值从 `PreStop` 钩子操作开始执行时开始计算,而不是从发送 `SIGTERM` 信号时开始计算。

## 建议

### 使用 Pod 中断预算保护关键工作负载

Pod 中断预算或 PDB 可以暂时阻止驱逐过程,如果应用程序的副本数量低于声明的阈值。一旦可用副本数量超过阈值,驱逐过程将继续。您可以使用 PDB 声明 `minAvailable` 和 `maxUnavailable` 副本数。例如,如果您希望您的应用程序至少有三个副本可用,您可以创建一个 PDB。

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

上述 PDB 策略告诉 Kubernetes 在三个或更多副本可用之前,暂停驱逐过程。节点排空会尊重 `PodDisruptionBudgets`。在 EKS 托管节点组升级期间,[节点以 15 分钟的超时时间排空](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-update-behavior.html)。15 分钟后,如果更新未强制(EKS 控制台中称为滚动更新),更新将失败。如果强制更新,则 pod 将被删除。

对于自管理节点,您还可以使用像 [AWS Node Termination Handler](https://github.com/aws/aws-node-termination-handler) 这样的工具,它确保 Kubernetes 控制平面适当地响应可能导致 EC2 实例无法使用的事件,例如 [EC2 维护](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-instances-status-check_sched.html)事件和 [EC2 Spot 中断](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-interruptions.html)。它使用 Kubernetes API 将节点隔离,以确保不会调度新的 Pod,然后对其进行排空,终止任何正在运行的 Pod。

您可以使用 Pod 反亲和性来调度部署的 Pod 在不同的节点上,以避免节点升级期间与 PDB 相关的延迟。

### 实践混沌工程
> 混沌工程是在分布式系统上进行实验的学科,目的是建立对系统在动荡条件下的能力的信心。

在他的博客中,Dominik Tornow 解释说,[Kubernetes 是一个声明式系统](https://medium.com/@dominik.tornow/the-mechanics-of-kubernetes-ac8112eaa302),其中"*用户向系统提供所需状态的表示。系统然后考虑当前状态和所需状态,以确定从当前状态过渡到所需状态的一系列命令*"。这意味着 Kubernetes 始终存储*所需状态*,如果系统偏离,Kubernetes 将采取行动来恢复状态。例如,如果一个工作节点变得不可用,Kubernetes 将重新调度 Pod 到另一个工作节点。同样,如果一个 `replica` 崩溃,[部署控制器](https://kubernetes.io/docs/concepts/architecture/controller/#design)将创建一个新的 `replica`。通过这种方式,Kubernetes 控制器会自动修复故障。

混沌工程工具,如 [Gremlin](https://www.gremlin.com),可帮助您测试 Kubernetes 集群的弹性,并识别单点故障。引入人为混沌的工具可以揭示系统性的弱点,提供机会识别瓶颈和配置错误,并在受控环境中纠正问题。混沌工程哲学主张有目的地破坏事物,并对基础设施进行压力测试,以最小化意外停机。

### 使用服务网格

您可以使用服务网格来提高应用程序的弹性。服务网格可以实现服务间通信,并增加微服务网络的可观察性。大多数服务网格产品都通过在每个服务旁运行一个小型网络代理来工作,该代理拦截和检查应用程序的网络流量。您可以在不修改应用程序的情况下将其置于网格中。使用服务代理内置的功能,您可以生成网络统计信息、创建访问日志,并为分布式跟踪添加 HTTP 头。

服务网格可以通过自动重试请求、设置超时、断路和限流等功能,帮助您使微服务更加有弹性。

如果您运营多个集群,您可以使用服务网格实现跨集群的服务间通信。

### 服务网格
+ [AWS App Mesh](https://aws.amazon.com/app-mesh/)
+ [Istio](https://istio.io)
+ [LinkerD](http://linkerd.io)
+ [Consul](https://www.consul.io)

---

## 可观察性

可观察性是一个包括监控、日志记录和跟踪的总称。基于微服务的应用程序本质上是分布式的。与监控单个系统就足够的单体应用程序不同,在分布式应用程序架构中,您需要监控每个组件的性能。您可以使用集群级监控、日志记录和分布式跟踪系统来识别集群中的问题,然后在它们影响您的客户之前解决这些问题。

Kubernetes 内置的故障排查和监控工具有限。Metrics Server 收集资源指标并将它们存储在内存中,但不持久化它们。您可以使用 kubectl 查看 Pod 的日志,但 Kubernetes 不会自动保留日志。分布式跟踪的实现是在应用程序代码级别完成的,或者使用服务网格。

Kubernetes 的可扩展性在这里发挥作用。Kubernetes 允许您引入首选的集中式监控、日志记录和跟踪解决方案。

## 建议

### 监控您的应用程序

现代应用程序需要监控的指标数量不断增加。如果您有一种自动跟踪应用程序的方法,您就可以专注于解决客户的挑战。集群范围的监控工具,如 [Prometheus](https://prometheus.io) 或 [CloudWatch Container Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights.html),可以监控您的集群和工作负载,并在出现问题(最好是在问题发生之前)时为您提供信号。

监控工具允许您创建警报,您的运营团队可以订阅这些警报。考虑制定规则来激活可能导致停机或影响应用程序性能的事件的警报。

如果您不确定应该监控哪些指标,您可以从这些方法中获取灵感:

- [RED 方法](https://www.weave.works/blog/a-practical-guide-from-instrumenting-code-to-specifying-alerts-with-the-red-method)。代表请求、错误和持续时间。
- [USE 方法](http://www.brendangregg.com/usemethod.html)。代表利用率、饱和度和错误。

Sysdig 的文章[Kubernetes 上的最佳警报实践](https://sysdig.com/blog/alerting-kubernetes/)包含了可能影响应用程序可用性的组件的全面列表。

### 使用 Prometheus 客户端库公开应用程序指标

除了监控应用程序的状态和聚合标准指标外,您还可以使用 [Prometheus 客户端库](https://prometheus.io/docs/instrumenting/clientlibs/)公开应用程序特定的自定义指标,以提高应用程序的可观察性。

### 使用集中式日志记录工具收集和持久化日志

EKS 中的日志记录分为两类:控制平面日志和应用程序日志。EKS 控制平面日志直接从控制平面提供审核和诊断日志到您帐户中的 CloudWatch Logs。应用程序日志是 Pod 内部生成的日志。应用程序日志包括运行业务逻辑应用程序的 Pod 以及 CoreDNS、Cluster Autoscaler、Prometheus 等 Kubernetes 系统组件生成的日志。

[EKS 提供五种类型的控制平面日志](https://docs.aws.amazon.com/eks/latest/userguide/control-plane-logs.html):

1. Kubernetes API 服务器组件日志
2. 审计
3. 认证器
4. 控制器管理器
5. 调度器

控制器管理器和调度器日志可帮助诊断控制平面问题,如瓶颈和错误。默认情况下,EKS 控制平面日志不会发送到 CloudWatch Logs。您可以为帐户中的每个集群启用控制平面日志记录,并选择要捕获的 EKS 控制平面日志类型。

收集应用程序日志需要在集群中安装日志聚合工具,如 [Fluent Bit](http://fluentbit.io)、[Fluentd](https://www.fluentd.org) 或 [CloudWatch Container Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/deploy-container-insights-EKS.html)。

Kubernetes 日志聚合工具作为 DaemonSet 运行,并从节点中抓取容器日志。然后将应用程序日志发送到集中式目的地进行存储。例如,CloudWatch Container Insights 可以使用 Fluent Bit 或 Fluentd 收集日志并将其发送到 CloudWatch Logs 进行存储。Fluent Bit 和 Fluentd 支持许多流行的日志分析系统,如 Elasticsearch 和 InfluxDB,让您可以通过修改 Fluent bit 或 Fluentd 的日志配置来更改日志的存储后端。

### 使用分布式跟踪系统识别瓶颈

典型的现代应用程序的组件分布在网络上,其可靠性取决于构成应用程序的每个组件的正常运行。您可以使用分布式跟踪解决方案来了解请求的流动方式以及系统之间的通信方式。跟踪可以向您展示应用程序网络中存在的瓶颈,并防止可能导致级联故障的问题。

您有两种选择来实现应用程序的跟踪:您可以在代码级别实现分布式跟踪,使用共享库,或使用服务网格。

在代码级别实现跟踪可能是不利的。在这种方法中,您必须对代码进行更改。如果您有多语言应用程序,这会更加复杂。您还需要维护另一个跨服务的库。

像 [LinkerD](http://linkerd.io)、[Istio](http://istio.io) 和 [AWS App Mesh](https://aws.amazon.com/app-mesh/) 这样的服务网格可以在您的应用程序代码中进行最少的更改的情况下实现分布式跟踪。您可以使用服务网格来标准化指标生成、日志记录和跟踪。

跟踪工具,如 [AWS X-Ray](https://aws.amazon.com/xray/) 和 [Jaeger](https://www.jaegertracing.io),支持共享库和服务网格实现。

考虑使用像 [AWS X-Ray](https://aws.amazon.com/xray/) 或 [Jaeger](https://www.jaegertracing.io) 这样的跟踪工具,它们支持两种(共享库和服务网格)实现,这样如果您后来采用服务网格,您就不必切换工具。