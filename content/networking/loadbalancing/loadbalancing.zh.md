!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

# 避免使用 Kubernetes 应用程序和 AWS 负载均衡器时出现的错误和超时

在创建必要的 Kubernetes 资源(Service、Deployment、Ingress 等)后,您的 pod 应该能够通过弹性负载均衡器接收来自客户端的流量。但是,当您对应用程序或 Kubernetes 环境进行更改时,您可能会发现出现错误、超时或连接重置。这些更改可能会触发应用程序部署或扩缩操作(手动或自动)。

不幸的是,即使您的应用程序没有记录任何问题,也可能会出现这些错误。这是因为控制集群资源的 Kubernetes 系统可能运行得比控制负载均衡器目标注册和健康状况的 AWS 系统更快。您的 pod 也可能在应用程序准备好接收请求之前就开始接收流量。

让我们回顾一下 pod 变为就绪状态以及如何将流量路由到 pod 的过程。

## Pod 就绪性

这张图来自 [2019 年 Kubecon 演讲](https://www.youtube.com/watch?v=Vw9GmSeomFg),展示了 pod 变为就绪状态并接收 `LoadBalancer` 服务流量的步骤:
![readiness.png](readiness.png)
*[Ready? A Deep Dive into Pod Readiness Gates for Service Health... - Minhan Xia & Ping Zou](https://www.youtube.com/watch?v=Vw9GmSeomFg)*  
当一个 pod 是 NodePort 服务的成员时,Kubernetes 将经历以下步骤:

1. pod 在 Kubernetes 控制平面上创建(即通过 `kubectl` 命令或扩缩操作)。
2. `kube-scheduler` 调度 pod,并将其分配到集群中的一个节点。
3. 在分配的节点上运行的 kubelet 将接收更新(通过 `watch`)并与其本地容器运行时通信,以启动 pod 中指定的容器。
    1. 当容器已经启动运行(并可选地通过 `ReadinessProbes`)时,kubelet 将通过向 `kube-apiserver` 发送更新来将 pod 状态更新为 `Ready`。
4. Endpoint 控制器将接收更新(通过 `watch`)说有一个新的 `Ready` pod 需要添加到服务的 Endpoints 列表中,并将 pod IP/端口元组添加到相应的端点数组中。
5. `kube-proxy` 接收更新(通过 `watch`)说有一个新的 IP/端口需要添加到 Service 的 iptables 规则中。
    1. 工作节点上的本地 iptables 规则将被更新,添加 NodePort 服务的目标 pod。

!!! 注意
    当使用 Ingress 资源和 Ingress 控制器(如 AWS 负载均衡器控制器)时,第 5 步由相关控制器处理,而不是 `kube-proxy`。控制器将采取必要的配置步骤(如注册/注销目标到负载均衡器),以允许流量按预期流动。

当 pod 被终止或转换到未就绪状态时，会发生类似的过程。API 服务器将收到来自控制器、kubelet 或 kubectl 客户端的更新,以终止 pod。步骤 3-5 将继续进行,但会从端点列表和 iptables 规则中删除 Pod IP/Tuple,而不是插入。

### 对部署的影响

下图显示了应用程序部署触发 pod 替换时采取的步骤:
![deployments.png](deployments.png)
*[Ready? A Deep Dive into Pod Readiness Gates for Service Health... - Minhan Xia & Ping Zou](https://www.youtube.com/watch?v=Vw9GmSeomFg)*  
值得注意的是,第二个 Pod 将不会部署,直到第一个 pod 达到"就绪"状态。步骤 4 和 5 也将与上述部署操作并行发生。

这意味着,当部署控制器继续处理下一个 pod 时,传播新 pod 状态的操作可能仍在进行中。由于此过程也会终止较旧版本的 pod,这可能会导致 pod 已达到就绪状态,但这些更改仍在传播,而较旧版本的 pod 已被终止的情况。

当与 AWS 等云提供商的负载均衡器一起工作时,这个问题会更加严重,因为上述描述的 Kubernetes 系统默认情况下不考虑负载均衡器上的注册时间或运行状况检查。**这意味着部署更新可能完全循环经过 pod,但负载均衡器尚未完成对新 pod 的运行状况检查或注册,这可能会导致中断。**

当 pod 被终止时,也会出现类似的问题。根据负载均衡器配置,pod 可能需要一两分钟才能取消注册并停止接受新请求。**Kubernetes 不会延迟滚动部署以等待此注销,这可能会导致负载均衡器仍在向已被终止的目标 pod 的 IP/端口发送流量的状态。**

为了避免这些问题,我们可以添加配置,以确保 Kubernetes 系统的操作更符合 AWS 负载均衡器的行为。

## 建议

### 使用 IP 目标类型负载均衡器

当创建 `LoadBalancer` 类型服务时,流量从负载均衡器发送到集群中的*任何节点*,通过**实例目标类型**注册。然后每个节点将从 `NodePort` 重定向流量到服务端点数组中的 Pod/IP 元组,该目标可能在单独的工作节点上运行。

!!! note
    请记住,该数组应仅包含"就绪"的 pod

![nodeport.png](nodeport.png)

这增加了您的请求的另一个跳转,并增加了负载均衡器配置的复杂性。例如,如果上述负载均衡器配置了会话亲和性,则该亲和性只能保持在负载均衡器和后端节点之间(取决于亲和性配置)。

由于负载均衡器没有直接与后端 Pod 通信,因此控制 Kubernetes 系统中的流量流和时间变得更加困难。

在使用 [AWS Load Balancer Controller](https://github.com/kubernetes-sigs/aws-load-balancer-controller) 时,**IP 目标类型**可用于直接将 Pod IP/端口元组注册到负载均衡器:
![ip.png](ip.png)
这简化了从负载均衡器到目标 Pod 的流量路径。这意味着当注册新目标时,我们可以确保该目标是"就绪"的 Pod IP 和端口,负载均衡器的运行状况检查将直接命中 Pod,并且在查看 VPC 流量日志或监控实用程序时,负载均衡器和 Pod 之间的流量将易于跟踪。

使用 IP 注册还允许我们直接控制流量对后端 Pod 的时间和配置,而不是试图通过 `NodePort` 规则管理连接。

### 利用 Pod 就绪门

[Pod 就绪门](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-readiness-gate)是 Pod 达到"就绪"状态之前必须满足的其他要求。

>[[...] AWS Load Balancer 控制器可以在构成您的入口或服务后端的 pod 上设置就绪状态条件。只有当 ALB/NLB 目标组中的相应目标显示"健康"状态时,pod 上的条件状态才会设置为 `True`。这可以防止部署的滚动更新在新创建的 pod 在 ALB/NLB 目标组中显示"健康"并准备好接受流量之前终止旧 pod。](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/deploy/pod_readiness_gate/)

就绪门确保 Kubernetes 在部署期间创建新副本时不会"太快",并避免 Kubernetes 已完成部署但新 Pod 尚未完成注册的情况。

要启用这些,您需要:

1. 部署最新版本的 [AWS Load Balancer Controller](https://github.com/kubernetes-sigs/aws-load-balancer-controller) (**[*如果升级旧版本,请参考文档*](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/deploy/upgrade/migrate_v1_v2/)**)
2. [为目标 pod 所在的命名空间贴上标签](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/deploy/pod_readiness_gate/)，标签为 `elbv2.k8s.aws/pod-readiness-gate-inject: enabled`，以自动注入 Pod 就绪门。
为确保命名空间中的所有 pod 都获得就绪门控配置,您需要在创建 pod 之前创建 Ingress 或 Service 并标记命名空间。

### 在终止之前确保 Pod 已从负载均衡器中注销

当 pod 被终止时,pod 就绪部分的步骤 4 和 5 会与容器进程接收终止信号同时发生。这意味着,如果您的容器能够快速关闭,它可能会比负载均衡器能够注销目标更快地关闭。为了避免这种情况,请调整 Pod 规格:

1. 添加一个 `preStop` 生命周期钩子,以允许应用程序注销并优雅地关闭连接。此钩子在由于 API 请求或管理事件(如存活/启动探测失败、抢占、资源争用等)而终止容器之前立即调用。关键是,[在发送终止信号之前,会调用此钩子并允许其完成](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination),前提是宽限期足够长以容纳执行。
```
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 180"] 
```

像上面这样简单的睡眠命令可用于在 pod 被标记为 `Terminating`（负载均衡器注销开始）和向容器进程发送终止信号之间引入短暂延迟。如果需要,此挂钩也可用于更高级的应用程序终止/关闭过程。

2. 将 `terminationGracePeriodSeconds` 扩展到容纳整个 `prestop` 执行时间以及应用程序响应终止信号进行优雅关闭所需的时间。在下面的示例中,宽限期扩展到 200 秒,这允许整个 `sleep 180` 命令完成,然后再额外 20 秒,以确保我的应用程序可以优雅地关闭。
```
    spec:
      terminationGracePeriodSeconds: 200
      containers:
      - name: webapp
        image: webapp-st:v1.3
        [...]
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 180"] 
```

确保 Pod 具有就绪探针

在 Kubernetes 中创建 Pod 时,默认就绪状态为"就绪",但大多数应用程序需要一些时间来实例化并准备好接受请求。[您可以在 Pod 规范中定义一个 `readinessProbe`](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)，使用 exec 命令或网络请求来确定应用程序是否已完成启动并准备好接受流量。

使用定义了 `readinessProbe` 的 Pod 在创建时处于"未就绪"状态,只有当 `readinessProbe` 成功时才会变为"就绪"状态。这可确保应用程序在启动完成之前不会投入使用。

建议使用存活探针允许在进入故障状态时重新启动应用程序,例如死锁,但应谨慎处理有状态应用程序,因为存活探针失败将触发应用程序重新启动。[启动探针](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-startup-probes)也可用于启动缓慢的应用程序。

下面的探针使用针对端口 80 的 HTTP 探针来检查 Web 应用程序何时准备就绪(相同的探针配置也用于存活探针):
```
        [...]
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /
            port: 80
          failureThreshold: 1
          periodSeconds: 10
          initialDelaySeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 80
          periodSeconds: 5
        [...]
```

配置 Pod 中断预算

[Pod 中断预算 (PDB)](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/#pod-disruption-budgets) 限制了复制应用程序中同时处于停机状态的 Pod 的数量,这是由于[自愿中断](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/#voluntary-and-involuntary-disruptions)造成的。例如,基于法定人数的应用程序希望确保运行的副本数量永远不会降低到法定人数所需的数量以下。Web 前端可能希望确保提供负载的副本数量永远不会低于总数的某个百分比。

PDB 将保护应用程序免受诸如节点被排空或应用程序部署等影响。PDB 确保在采取这些操作时,仍有最小数量或百分比的 Pod 保持可用。

!!! attention
    PDB 不会保护应用程序免受主机操作系统故障或网络连接丢失等不自愿中断的影响。

下面的示例确保始终至少有 1 个带有标签 `app: echoserver` 的 Pod 可用。[您可以为您的应用程序配置正确的副本数量,或使用百分比](https://kubernetes.io/docs/tasks/run-application/configure-pdb/#think-about-how-your-application-reacts-to-disruptions):
```
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: echoserver-pdb
  namespace: echoserver
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: echoserver
```


### 优雅地处理终止信号

当 pod 被终止时,容器内运行的应用程序将收到两个[信号](https://www.gnu.org/software/libc/manual/html_node/Standard-Signals.html)。第一个是 [`SIGTERM` 信号](https://www.gnu.org/software/libc/manual/html_node/Termination-Signals.html),这是一个"礼貌"的请求,要求进程停止执行。这个信号可以被阻塞或应用程序可以简单地忽略这个信号,因此在 `terminationGracePeriodSeconds` 过去后,应用程序将收到 [`SIGKILL` 信号](https://www.gnu.org/software/libc/manual/html_node/Termination-Signals.html)。`SIGKILL` 用于强制停止进程,它不能被[阻塞、处理或忽略](https://man7.org/linux/man-pages/man7/signal.7.html),因此总是致命的。

这些信号由容器运行时用来触发您的应用程序关闭。`SIGTERM` 信号也将在 `preStop` 钩子执行**之后**发送。使用上述配置,`preStop` 钩子将确保 pod 已从负载均衡器中注销,因此应用程序可以在收到 `SIGTERM` 信号时优雅地关闭任何剩余的开放连接。

!!! note
    [在使用"包装脚本"作为应用程序入口点的容器环境中,信号处理可能会很复杂](https://petermalmgren.com/signal-handling-docker/),因为脚本将成为 PID 1,可能无法将信号转发到您的应用程序。

### 谨慎对待注销延迟

弹性负载均衡停止向正在注销的目标发送请求。默认情况下,弹性负载均衡会等待 300 秒才完成注销过程,这可以帮助目标上的正在进行的请求完成。要更改弹性负载均衡等待的时间,请更新注销延迟值。
正在注销的目标的初始状态为 `draining`。在注销延迟过去后,注销过程完成,目标的状态变为 `unused`。如果目标是自动缩放组的一部分,它可以被终止并替换。

如果正在注销的目标没有正在进行的请求和活动连接,弹性负载均衡会立即完成注销过程,无需等待注销延迟。

!!! attention
    即使目标注销已完成,目标的状态也会显示为 `draining`,直到注销延迟超时过期。超时过期后,目标将转换为 `unused` 状态。

[如果正在注销的目标在注销延迟过去之前终止连接,客户端将收到 500 级错误响应](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html#deregistration-delay)。
这可以使用 Ingress 资源上的注解进行配置,使用 [`alb.ingress.kubernetes.io/target-group-attributes` 注解](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/ingress/annotations/#target-group-attributes)。例如:
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echoserver-ip
  namespace: echoserver
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/load-balancer-name: echoserver-ip
    alb.ingress.kubernetes.io/target-group-attributes: deregistration_delay.timeout_seconds=30
spec:
  ingressClassName: alb
  rules:
    - host: echoserver.example.com
      http:
        paths:
          - path: /
            pathType: Exact
            backend:
              service:
                name: echoserver-service
                port:
                  number: 8080
```

