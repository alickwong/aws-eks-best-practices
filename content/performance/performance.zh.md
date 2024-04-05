!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

# 性能效率支柱

性能效率支柱关注于有效利用计算资源以满足要求,以及如何在需求变化和技术发展时保持该效率。本节提供了在AWS上构建性能效率的深入最佳实践指南。

## 定义

为确保有效利用EKS容器服务,您应该收集有关整个架构的所有方面的数据,从高级设计到选择EKS资源类型。通过定期审查您的选择,您可以确保利用不断发展的Amazon EKS和容器服务。监控将确保您了解任何偏离预期性能的情况,以便您可以采取行动。

EKS容器的性能效率由三个方面组成:

- 优化您的容器
- 资源管理
- 可扩展性管理

## 最佳实践

### 优化您的容器

您可以在没有太多麻烦的情况下在Docker容器中运行大多数应用程序。您需要做一些事情来确保它在生产环境中有效运行,包括简化构建过程。以下最佳实践将帮助您实现这一目标。

#### 建议

- **使您的容器镜像无状态:** 使用Docker镜像创建的容器应该是临时的和不可变的。换句话说,容器应该是可丢弃和独立的,即可以构建一个新的容器并将其放置,而无需进行任何配置更改。设计您的容器使其无状态。如果您想使用持久性数据,请使用[卷](https://docs.docker.com/engine/admin/volumes/volumes/)。如果您想存储服务使用的机密或敏感应用程序数据,您可以使用AWS [Systems Manager](https://aws.amazon.com/systems-manager/)[Parameter Store](https://aws.amazon.com/ec2/systems-manager/parameter-store/)或第三方产品或开源解决方案,如[HashiCorp Valut](https://www.vaultproject.io/)和[Consul](https://www.consul.io/)进行运行时配置。
- [**最小基础镜像**](https://docs.docker.com/develop/develop-images/baseimages/) **:** 从小型基础镜像开始。Dockerfile中的每个其他指令都建立在这个镜像之上。基础镜像越小,生成的镜像就越小,下载速度也越快。例如, [alpine:3.7](https://hub.docker.com/r/library/alpine/tags/)镜像比[centos:7](https://hub.docker.com/r/library/centos/tags/)镜像小71 MB。您甚至可以使用[scratch](https://hub.docker.com/r/library/scratch/)基础镜像,这是一个空镜像,您可以在其上构建自己的运行时环境。

- **避免不必要的软件包:** 在构建容器镜像时,只包含应用程序所需的依赖项,避免安装不必要的软件包。例如,如果您的应用程序不需要SSH服务器,就不要包含它。这将减少复杂性、依赖项、文件大小和构建时间。使用.dockerignore文件排除与构建无关的文件。
- [**使用多阶段构建**](https://docs.docker.com/v17.09/engine/userguide/eng-image/multistage-build/#use-multi-stage-builds):多阶段构建允许您在第一个"构建"容器中构建应用程序,并在另一个容器中使用结果,同时使用相同的Dockerfile。更具体地说,在多阶段构建中,您在Dockerfile中使用多个FROM语句。每个FROM指令可以使用不同的基础,并且每个指令都开始一个新的构建阶段。您可以选择性地从一个阶段复制制品到另一个阶段,留下您不想在最终镜像中包含的所有内容。这种方法大大减小了最终镜像的大小,而无需努力减少中间层和文件的数量。
- **最小化层数:** Dockerfile中的每个指令都会为Docker镜像添加一个额外的层。指令和层的数量应保持最小,因为这会影响构建性能和时间。例如,下面的第一个指令将创建多个层,而第二个指令通过使用&&(链接)减少了层的数量,这将有助于提供更好的性能。这是在Dockerfile中减少创建层数的最佳方法。
- 
    ```
            RUN apt-get -y update
            RUN apt-get install -y python
            RUN apt-get -y update && apt-get install -y python
    ```
            
- **正确标记您的镜像:** 在构建镜像时,始终使用有用和有意义的标签对它们进行标记。这是一种很好的方式来组织和记录描述镜像的元数据,例如,包括来自CI服务器(如CodeBuild或Jenkins)的唯一计数器(如构建ID),以帮助识别正确的镜像。如果您在Docker命令中没有提供标签,则默认使用latest标签。我们建议不要使用自动创建的latest标签,因为使用此标签您将自动运行未来的主要版本,这可能会包含应用程序的突破性变更。最佳做法是避免使用latest标签,而是使用您的CI服务器创建的唯一摘要。

- **使用** [**构建缓存**](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/) **来提高构建速度**：缓存允许您利用现有的缓存映像,而不是从头开始构建每个映像。例如,您应该尽可能晚地将应用程序的源代码添加到Dockerfile中,以便基础映像和应用程序的依赖项被缓存,并且在每次构建时不会重新构建。要重复使用已缓存的映像,默认情况下在Amazon EKS中,kubelet将尝试从指定的注册表中拉取每个映像。但是,如果容器的imagePullPolicy属性设置为IfNotPresent或Never,则使用本地映像(分别优先或独占)。

- **映像安全性：** 使用公共映像可能是开始使用容器并将其部署到Kubernetes的一种很好的方式。但是,在生产中使用它们可能会带来一系列挑战。尤其是在安全性方面。确保遵循打包和分发容器/应用程序的最佳实践。例如,不要在容器中嵌入密码,您可能还需要控制它们内部的内容。建议使用私有存储库,如[Amazon ECR](https://aws.amazon.com/ecr/),并利用内置的[映像扫描](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning.html)功能来识别容器映像中的软件漏洞。

- **合理调整容器大小：** 在容器中开发和运行应用程序时,需要考虑几个关键领域。容器大小和应用程序部署管理方式可能会对您提供的服务的最终用户体验产生负面影响。为帮助您取得成功,以下最佳实践将帮助您合理调整容器大小。确定应用程序所需的资源数量后,您应该设置Kubernetes的请求和限制,以确保应用程序正确运行。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*(a) 测试应用程序*:  收集关键统计数据和其他性能指标 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;根据这些数据,您可以确定容器所需的最佳内存和 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;CPU 配置。关键统计数据包括: __*CPU、延迟、I/O、内存使用、 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;网络*__ 。如有必要,请进行单独的负载测试,确定容器的预期、平均和峰值内存和CPU使用情况。同时考虑容器中可能并行运行的所有进程。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;     建议使用 [CloudWatch Container insights](https://aws.amazon.com/blogs/mt/introducing-container-insights-for-amazon-ecs/) 或合作伙伴产品,这将提供 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;正确的信息来调整容器和Worker节点的大小。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;*(b)独立测试服务:*  由于许多应用程序在真正的 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;微服务架构中相互依赖,因此需要以高度独立的方式进行测试 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;,这意味着服务不仅能够独立正常运行,还能作为一个 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;协调一致的系统正常运行。

### 资源管理

在采用Kubernetes时,最常见的问题之一是"*我应该把什么放在Pod中?*"。例如,一个三层LAMP应用程序容器。我们应该将此应用程序保留在同一个pod中吗?虽然这可以作为单个pod有效运行,但这是Pod创建的反模式。有两个原因:

***(a)*** 如果您将两个容器放在同一个Pod中,您将被迫使用相同的扩展策略,这对生产环境来说并不理想,您也无法根据使用情况有效管理或限制资源。*例如:* 您可能只需要扩展前端,而不是前端和后端(MySQL)作为一个单元,也如果您想增加专门分配给后端的资源,您无法单独这样做。

***(b)*** 如果您有两个独立的 pod，一个用于前端，另一个用于后端。扩展将非常容易，并且您可以获得更好的可靠性。

上述做法可能并不适用于所有用例。在上述示例中，前端和后端可能会落在不同的机器上,它们将通过网络相互通信。因此,您需要问"***如果它们被放置并在不同的机器上运行,我的应用程序是否能正常工作?***"如果答案是"***否***",可能是由于应用程序设计或其他技术原因,那么将容器分组到单个 pod 中是有意义的。如果答案是"***是***",那么多个 pod 就是正确的方法。

#### 建议

+ **每个容器打包一个单一的应用程序:**
容器在单一应用程序运行时发挥最佳作用。这个应用程序应该有一个单一的父进程。例如,不要在同一个容器中运行 PHP 和 MySQL:这会更难调试,而且您无法单独水平扩展 PHP 容器。这种分离使您能够更好地将应用程序的生命周期与容器的生命周期绑定。您的容器应该是无状态和不可变的。无状态意味着任何状态(任何类型的持久性数据)都存储在容器外部,例如,您可以根据需要使用不同类型的外部存储,如持久磁盘、Amazon EBS 和 Amazon EFS,或托管数据库如 Amazon RDS。不可变意味着容器在其生命周期中不会被修改:没有更新、补丁和配置更改。要更新应用程序代码或应用补丁,您需要构建一个新的镜像并部署它。

+ **使用标签来标识 Kubernetes 对象:**
[标签](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/#labels)允许以批量方式查询和操作 Kubernetes 对象。它们还可用于识别和组织 Kubernetes 对象到组。因此,定义标签应该位于任何 Kubernetes 最佳实践列表的最前面。

+ **设置资源请求限制:**
设置请求限制是用于控制容器可以消耗的系统资源(如 CPU 和内存)数量的机制。这些设置是容器初始启动时保证获得的资源。如果容器请求资源,容器编排器(如 Kubernetes)将只在可以提供该资源的节点上调度它。另一方面,限制可确保容器永远不会超过某个值。容器只允许达到限制,然后就会受到限制。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; 在下面的 Pod 清单示例中,我们添加了 1.0 CPU 和 256 MB 内存的限制
```
        apiVersion: v1
        kind: Pod
        metadata:
          name: nginx-pod-webserver
          labels:
            name: nginx-pod
        spec:
          containers:
          - name: nginx
            image: nginx:latest
            resources:
              limits:
                memory: "256Mi"
                cpu: "1000m"
              requests:
                memory: "128Mi"
                cpu: "500m"
            ports:
            - containerPort: 80

         
```


定义这些请求和限制是最佳实践。如果您没有包含这些值,调度程序就无法了解所需的资源。没有这些信息,调度程序可能会将pod调度到没有足够资源提供可接受的应用程序性能的节点上。

+ **限制并发中断的数量:**
使用 _PodDisruptionBudget_,这些设置允许您在自愿驱逐事件期间设置最小可用和最大不可用pod的策略。驱逐的一个例子是在对节点执行维护或排空节点时。

_示例:_ 一个Web前端可能希望确保在任何给定时间都有8个pod可用。在这种情况下,驱逐可以驱逐尽可能多的pod,只要有八个可用。
```
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: frontend-demo
spec:
  minAvailable: 8
  selector:
    matchLabels:
      app: frontend
```


&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;**注意:** 您也可以通过使用 `maxAvailable` 或 `maxUnavailable` 参数来指定 pod 中断预算作为百分比。

+ **使用命名空间:**
命名空间允许物理集群被多个团队共享。命名空间允许将创建的资源划分到逻辑命名组中。这使您可以为每个命名空间设置资源配额、基于角色的访问控制 (RBAC) 和网络策略。它提供了软多租户功能。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;例如,如果您在单个 Amazon EKS 集群上运行三个应用程序,并由三个不同的团队访问,每个团队都需要多个资源约束和不同的 QoS 级别,您可以为每个团队创建一个命名空间,并为每个团队分配 CPU 和内存等资源的配额。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;您还可以通过启用 [LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/) 准入控制器在 Kubernetes 命名空间级别指定默认限制。这些默认限制将限制给定 Pod 可以使用的 CPU 或内存量,除非 Pod 配置中明确覆盖了这些默认值。

+ **管理资源配额:**
每个命名空间都可以分配资源配额。指定配额允许限制命名空间内所有资源的集群资源消耗。资源配额可以通过 [ResourceQuota](https://kubernetes.io/docs/concepts/policy/resource-quotas/) 对象定义。命名空间中存在 ResourceQuota 对象可确保执行资源配额。

+ **为 Pod 配置健康检查:**

健康检查是一种简单的方法,可以让系统知道您的应用程序实例是否正在工作。如果您的应用程序实例无法正常工作,则其他服务不应访问它或向它发送请求。相反,请求应发送到另一个正在工作的应用程序实例。系统还应将您的应用程序恢复到健康状态。默认情况下,所有正在运行的 pod 都将重启策略设置为始终,这意味着节点内运行的 kubelet 将在容器遇到错误时自动重新启动 pod。健康检查通过[容器探针](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes)的概念扩展了 kubelet 的这种功能。

Kubernetes 提供两种[健康检查](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)类型:就绪和存活探针。例如,考虑您的某个应用程序通常运行很长时间,但转换为非运行状态并且只能通过重新启动来恢复。您可以使用存活探针来检测和补救这种情况。使用健康检查可以提高您的应用程序的可靠性和更高的正常运行时间。

+ **高级调度技术:**
通常,调度程序确保 pod 仅放置在具有足够空闲资源的节点上,并且在节点之间,它们试图平衡节点、部署、副本等的资源利用率。但有时您希望控制 pod 的调度方式。例如,也许您希望确保某些 pod 仅在具有专用硬件(如需要 GPU 机器进行 ML 工作负载)的节点上进行调度。或者您想将频繁通信的服务共置。

Kubernetes 提供了许多[高级调度功能](https://kubernetes.io/blog/2017/03/advanced-scheduling-in-kubernetes/)和多个过滤器/约束,以将 pod 调度到正确的节点。例如,在使用 Amazon EKS 时,您可以使用[污点和容忍](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#taints-and-toleations-beta-feature)来限制哪些工作负载可以在特定节点上运行。您还可以使用[节点选择器](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#nodeselector)和[亲和力和反亲和力](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#affinity-and-anti-affinity)构造来控制 pod 调度,甚至可以为此目的构建自己的自定义调度程序。

#### 可扩展性管理
容器是无状态的。它们诞生,当它们死亡时,它们不会复活。在 Amazon EKS 上,您可以利用许多技术,不仅可以扩展您的容器化应用程序,还可以扩展 Kubernetes 工作节点。

#### 建议

在 Amazon EKS 上,您可以配置[水平 Pod 自动缩放器](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)。它会根据观察到的 CPU 利用率(或使用基于应用程序提供的指标的[自定义指标](https://git.k8s.io/community/contributors/design-proposals/instrumentation/custom-metrics-api.md))自动调整复制控制器、部署或副本集中 Pod 的数量。

您可以使用[垂直 Pod 自动缩放器](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler),它会自动调整 Pod 的 CPU 和内存预留,以帮助"合理调整"您的应用程序。这种调整可以提高集群资源利用率,并为其他 Pod 释放 CPU 和内存。这在您的生产数据库"MongoDB"与无状态应用程序前端的缩放方式不同的场景中很有用。在这种情况下,您可以使用 VPA 来扩展 MongoDB Pod。

要启用 VPA,您需要使用 Kubernetes 指标服务器,它是集群中资源使用数据的聚合器。它默认未部署在 Amazon EKS 集群中。您需要在[配置 VPA](https://docs.aws.amazon.com/eks/latest/userguide/vertical-pod-autoscaler.html)之前对其进行配置,或者您也可以使用 Prometheus 为垂直 Pod 自动缩放器提供指标。

虽然 HPA 和 VPA 缩放部署和 Pod,但[集群自动缩放器](https://github.com/kubernetes/autoscaler)将扩展和收缩工作节点池的大小。它根据当前利用率调整 Kubernetes 集群的大小。当有 Pod 由于资源不足而无法调度到任何当前节点上,或者添加新节点会增加集群资源的整体可用性时,集群自动缩放器会增加集群的大小。请遵循此[分步指南](https://eksworkshop.com/scaling/deploy_ca/)设置集群自动缩放器。如果您使用 AWS Fargate 上的 Amazon EKS,AWS 将为您管理控制平面。

请查看可靠性支柱以获取详细信息。

#### 监控
#### 部署最佳实践
#### 权衡
