
成本优化 - 网络

为了实现高可用性(HA)和容错能力,构建高可用系统是最佳实践。在实践中,这意味着将您的工作负载和底层基础设施分散在给定AWS区域的多个可用区(AZ)中。确保您的Amazon EKS环境具有这些特性将提高系统的整体可靠性。与此同时,您的EKS环境可能还由各种构造(如VPC)、组件(如ELB)和集成(如ECR和其他容器注册表)组成。

高可用系统和其他特定用例组件的组合可能会显著影响数据传输和处理的方式。这反过来又会影响由于数据传输和处理而产生的成本。

下面详述的做法将帮助您设计和优化您的EKS环境,以实现不同领域和用例的成本效益。

## Pod到Pod通信

根据您的设置,Pod之间的网络通信和数据传输可能会对运行Amazon EKS工作负载的总体成本产生重大影响。本节将介绍不同的概念和方法,以在考虑高可用(HA)架构、应用程序性能和弹性的同时,降低Pod间通信的成本。

### 限制流量到可用区

频繁的跨区域流量(在AZ之间分配的流量)可能会对您的网络相关成本产生重大影响。以下是一些策略,介绍如何控制您的EKS集群中Pod之间的跨区域流量。

_如果您想获得集群中Pod之间跨区域流量(如以字节为单位的数据传输量)的细粒度可见性,[请参考此帖子](https://aws.amazon.com/blogs/containers/getting-visibility-into-your-amazon-eks-cross-az-pod-to-pod-network-bytes/)。_

**使用拓扑感知路由(以前称为拓扑感知提示)**

![拓扑感知路由](../images/topo_aware_routing.png)

使用拓扑感知路由时,了解Services、EndpointSlices和`kube-proxy`如何协同工作来路由流量很重要。如上图所示,Services是接收流量的稳定网络抽象层。创建Service时,会创建多个EndpointSlices。每个EndpointSlice都包含一个端点列表,其中包含一组Pod地址以及它们运行所在的节点和任何其他拓扑信息。`kube-proxy`是一个守护进程,在集群中的每个节点上运行,并履行内部路由的作用,但它是基于从创建的EndpointSlices中获取的内容。
当启用并实施[*拓扑感知路由*](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/)的Kubernetes Service时，EndpointSlice控制器将按比例将端点分配到集群分布的不同区域。对于这些端点中的每一个,EndpointSlice控制器还将设置一个区域提示。提示描述了端点应该为哪个区域提供流量服务。然后,`kube-proxy`将根据应用的提示,将流量从一个区域路由到一个端点。

下图显示了EndpointSlices如何根据提示进行组织,以便`kube-proxy`可以知道应该根据其区域起点转发到哪个目的地。没有提示,就没有这种分配或组织,流量将被代理到不同区域的目的地,而不考虑它的来源。

![Endpoint Slice](../images/endpoint_slice.png)

在某些情况下,EndpointSlice控制器可能会为不同的区域应用提示,这意味着端点最终可能会为来自不同区域的流量提供服务。这样做的原因是试图在不同区域的端点之间保持流量的均匀分布。

下面是一个代码片段,展示了如何为Service启用拓扑感知路由。
```yaml hl_lines="6-7"
apiVersion: v1
kind: Service
metadata:
  name: orders-service
  namespace: ecommerce
    annotations:
      service.kubernetes.io/topology-mode: Auto
spec:
  selector:
    app: orders
  type: ClusterIP
  ports:
  - protocol: TCP
    port: 3003
    targetPort: 3003
```

以下截图显示了 EndpointSlice 控制器成功将提示应用于在 AZ `eu-west-1a` 中运行的 Pod 副本的端点。

![Slice shell](../images/slice_shell.png)

!!! note
    请注意,拓扑感知路由仍处于 **beta** 阶段。此外,当工作负载广泛且均匀地分布在集群拓扑中时,此功能会更加可预测。因此,强烈建议将其与增加应用程序可用性的调度约束一起使用,例如 [pod 拓扑传播约束](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)。

**使用自动缩放器:将节点配置到特定的 AZ**

_我们强烈建议_ 在多个 AZ 中运行高可用性环境中的工作负载。这可以提高应用程序的可靠性,特别是在 AZ 出现问题时。如果您愿意牺牲可靠性来降低网络相关成本,可以将节点限制在单个 AZ 中。

要在同一个 AZ 中运行所有 Pod,可以在同一个 AZ 中配置工作节点,或者在同一个 AZ 中的工作节点上调度 Pod。要在单个 AZ 中配置节点,请使用 [Cluster Autoscaler (CA)](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler) 定义一个节点组,其子网属于同一个 AZ。对于 [Karpenter](https://karpenter.sh/)，请使用 "[_topology.kubernetes.io/zone"_](http://topology.kubernetes.io/zone%E2%80%9D) 并指定要在其中创建工作节点的 AZ。例如,下面的 Karpenter 配置程序片段在 us-west-2a AZ 中配置节点。

**Karpenter**
```yaml hl_lines="5-9"
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
name: single-az
spec:
  requirements:
  - key: "topology.kubernetes.io/zone“
    operator: In
    values: ["us-west-2a"]
```

**集群自动缩放器 (CA)**

![Cluster Autoscaler Architecture](https://github.com/kubernetes/autoscaler/raw/master/cluster-autoscaler/docs/images/architecture.png)

集群自动缩放器是一个 Kubernetes 组件,它可以根据工作负载的需求自动调整集群的大小。它可以根据 Pod 的请求和限制,以及集群的资源使用情况,来决定何时添加或删除节点。这有助于确保集群始终有足够的资源来运行应用程序,同时又不会浪费资源。

集群自动缩放器可以与 Kubernetes 的其他组件无缝集成,如 Horizontal Pod Autoscaler (HPA) 和 Vertical Pod Autoscaler (VPA)。这使得您可以构建一个全面的自动化解决方案,以优化集群的性能和成本。
```yaml hl_lines="7-8"
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-ca-cluster
  region: us-east-1
  version: "1.21"
availabilityZones:
- us-east-1a
managedNodeGroups:
- name: managed-nodes
  labels:
    role: managed-nodes
  instanceType: t3.medium
  minSize: 1
  maxSize: 10
  desiredCapacity: 1
...
```

**使用 Pod 分配和节点亲和性**

此外，如果您有在多个可用区(AZ)中运行的工作节点，每个节点都会有标签 _[topology.kubernetes.io/zone](http://topology.kubernetes.io/zone")_ 其值为其所在的 AZ(如 us-west-2a 或 us-west-2b)。您可以利用 `nodeSelector` 或 `nodeAffinity` 将 Pod 调度到单个 AZ 中的节点。例如,以下清单文件将在 us-west-2a 可用区中的节点上调度 Pod。
```yaml hl_lines="7-9"
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  nodeSelector:
    topology.kubernetes.io/zone: us-west-2a
  containers:
  - name: nginx
    image: nginx 
    imagePullPolicy: IfNotPresent
```

限制流量到节点

在区域级别限制流量并不总是足够的。除了降低成本之外,您可能还需要减少某些应用程序之间频繁通信的网络延迟。为了实现最佳网络性能并降低成本,您需要一种方法来限制流量到特定节点。例如,微服务A应该始终与节点1上的微服务B通信,即使在高可用(HA)设置中也是如此。让微服务A在节点1上与节点2上的微服务B通信可能会对这类应用程序的所需性能产生负面影响,特别是如果节点2位于完全不同的可用区(AZ)中。

**使用服务内部流量策略**

为了限制Pod网络流量到节点,您可以利用_[服务内部流量策略](https://kubernetes.io/docs/concepts/services-networking/service-traffic-policy/)_。默认情况下,发送到工作负载服务的流量将随机分布在不同生成的端点上。因此,在HA架构中,这意味着来自微服务A的流量可能会流向任何微服务B的副本,位于不同的AZ上的任何节点。但是,如果将服务的内部流量策略设置为`Local`,流量将被限制到来自该流量的节点上的端点。此策略规定专门使用节点本地端点。这意味着,该工作负载的网络流量相关成本将低于集群范围内的分发。同时,延迟也会更低,使您的应用程序更加高性能。

!!! 注意
    请注意,此功能不能与Kubernetes中的拓扑感知路由结合使用。

![Local internal traffic](../images/local_traffic.png)

下面是一个代码片段,展示如何为服务设置_内部流量策略_。
```yaml hl_lines="14"
apiVersion: v1
kind: Service
metadata:
  name: orders-service
  namespace: ecommerce
spec:
  selector:
    app: orders
  type: ClusterIP
  ports:
  - protocol: TCP
    port: 3003
    targetPort: 3003
  internalTrafficPolicy: Local
```

为了避免由于流量下降而导致应用程序出现意外行为,您应该考虑以下方法:

* 为每个通信 Pod 运行足够的副本
* 使用[拓扑传播约束](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)实现 Pod 的相对均匀分布
* 对于通信 Pod 的共同位置,利用[pod 亲和性规则](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity)

在此示例中,您有 2 个微服务 A 副本和 3 个微服务 B 副本。如果微服务 A 的副本分布在节点 1 和 2 之间,而微服务 B 的 3 个副本全部位于节点 3,则由于 `Local` 内部流量策略,它们将无法通信。当没有可用的节点本地端点时,流量将被丢弃。

![node-local_no_peer](../images/no_node_local_1.png)

如果微服务 B 确实有 2 个副本位于节点 1 和 2,那么对等应用程序之间将会有通信。但是,您仍然会有一个微服务 B 的孤立副本,没有任何对等副本可以通信。

![node-local_with_peer](../images/no_node_local_2.png)

!!! note
    在某些场景中,如上图所示的孤立副本可能不会引起关注,如果它仍然能够提供服务(例如响应来自外部流量的请求)。

**结合拓扑传播约束使用服务内部流量策略**

将_内部流量策略_与_拓扑传播约束_结合使用可以帮助确保您在不同节点上为通信微服务提供正确数量的副本。
```yaml hl_lines="16-22"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: express-test
spec:
  replicas: 6
  selector:
    matchLabels:
      app: express-test
  template:
    metadata:
      labels:
        app: express-test
        tier: backend
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: "topology.kubernetes.io/zone"
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: express-test
```

**使用 Pod 亲和性规则与服务内部流量策略**

另一种方法是在使用服务内部流量策略时利用 Pod 亲和性规则。使用 Pod 亲和性,您可以影响调度程序将某些 Pod 共置,因为它们频繁通信。通过对某些 Pod 应用严格的调度约束(`requiredDuringSchedulingIgnoredDuringExecution`)，这将在调度程序将 Pod 放置在节点上时为 Pod 共置提供更好的结果。
```yaml hl_lines="11-20"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: graphql
  namespace: ecommerce
  labels:
    app.kubernetes.io/version: "0.1.6"
    ...
    spec:
      serviceAccountName: graphql-service-account
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - orders
            topologyKey: "kubernetes.io/hostname"
```


## 负载均衡器到 Pod 的通信

EKS 工作负载通常由负载均衡器前置,该负载均衡器将流量分配到 EKS 集群中相关的 Pod。您的架构可能包括内部和/或外部面向的负载均衡器。根据您的架构和网络流量配置,负载均衡器和 Pod 之间的通信可能会产生大量的数据传输费用。

您可以使用 [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller) 自动管理 ELB 资源(ALB 和 NLB)的创建。在这种设置中产生的数据传输费用将取决于网络流量的路径。AWS Load Balancer Controller 支持两种网络流量模式,_实例模式_和 _IP 模式_。

使用 _实例模式_ 时,将在 EKS 集群中的每个节点上打开一个 NodePort。然后负载均衡器将在节点之间均匀代理流量。如果目标 Pod 在同一节点上运行,则不会产生数据传输费用。但是,如果目标 Pod 位于不同的节点和不同的可用区,则从 kube-proxy 到目标 Pod 将有一个额外的网络跳跃。在这种情况下,将产生跨可用区的数据传输费用。由于流量在节点之间均匀分布,很可能会产生与从 kube-proxy 到相关目标 Pod 的跨区域网络流量跳跃相关的额外数据传输费用。

下图描述了从负载均衡器到 NodePort 的网络路径,以及随后从 `kube-proxy` 到不同可用区中另一个节点上的目标 Pod 的网络路径。这是 _实例模式_ 设置的一个示例。

![LB to Pod](../images/lb_2_pod.png)

使用 _IP 模式_ 时,网络流量直接从负载均衡器代理到目标 Pod。因此,这种方法不涉及任何 _数据传输费用_。

!!! 提示
    建议您将负载均衡器设置为 _IP 流量模式_ 以减少数据传输费用。对于这种设置,还需要确保负载均衡器部署在 VPC 中的所有子网上。

下图描述了在 _IP 模式_ 下,从负载均衡器到 Pod 的网络路径。

![IP mode](../images/ip_mode.png)

## 来自容器注册表的数据传输

### Amazon ECR

向 Amazon ECR 私有注册表的数据传输是免费的。_同区域内的数据传输不收费_,但向互联网和跨区域的数据传输将按互联网数据传输费率在传输的两端收费。
您应该利用 ECR 内置的[镜像复制功能](https://docs.aws.amazon.com/AmazonECR/latest/userguide/replication.html)将相关的容器镜像复制到与您的工作负载相同的区域。这样,复制只需收费一次,同一区域(区域内)的所有镜像拉取都将免费。

您可以进一步通过_使用[接口 VPC 端点](https://docs.aws.amazon.com/whitepapers/latest/aws-privatelink/what-are-vpc-endpoints.html)连接到区域内的 ECR 存储库_来降低从 ECR 拉取镜像(数据传输出)的成本。连接到 ECR 的公共 AWS 端点(通过 NAT 网关和互联网网关)的替代方法将产生更高的数据处理和传输成本。下一节将更详细地介绍如何降低工作负载与 AWS 服务之间的数据传输成本。

如果您正在运行使用特别大的镜像的工作负载,您可以构建自己的自定义 Amazon 机器镜像(AMI),其中包含预缓存的容器镜像。这可以减少初始镜像拉取时间和从容器注册表到 EKS 工作节点的潜在数据传输成本。

## 互联网和 AWS 服务的数据传输

将 Kubernetes 工作负载与其他 AWS 服务或第三方工具和平台集成是一种常见做法,通过互联网进行。用于路由流量到达相关目的地的底层网络基础设施可能会影响数据传输过程中产生的成本。

### 使用 NAT 网关

NAT 网关是执行网络地址转换(NAT)的网络组件。下图描述了 EKS 集群中的 Pod 与其他 AWS 服务(Amazon ECR、DynamoDB 和 S3)以及第三方平台进行通信。在此示例中,Pod 在单独的可用区的私有子网中运行。为了从互联网发送和接收流量,部署了一个 NAT 网关到一个可用区的公有子网,允许任何具有私有 IP 地址的资源共享单个公有 IP 地址以访问互联网。这个 NAT 网关又与互联网网关组件通信,允许数据包发送到最终目的地。

当使用 NAT 网关进行此类用例时,_您可以通过在每个可用区部署一个 NAT 网关来最小化数据传输成本_。这样,路由到互联网的流量将通过同一可用区的 NAT 网关,避免了可用区间的数据传输。但是,尽管您将节省可用区间数据传输的成本,但这种设置的影响是您将产生额外的 NAT 网关成本。

下图描述了推荐的方法。

![推荐的方法](../images/recommended_approach.png)

### 使用 VPC 端点

为进一步降低此类架构的成本,_您应该使用[VPC 端点](https://docs.aws.amazon.com/whitepapers/latest/aws-privatelink/what-are-vpc-endpoints.html)在您的工作负载和 AWS 服务之间建立连接_。VPC 端点允许您在不通过互联网传输数据/网络数据包的情况下从 VPC 内部访问 AWS 服务。所有流量都是内部的,并保留在 AWS 网络中。VPC 端点有两种类型:接口 VPC 端点([许多 AWS 服务支持](https://docs.aws.amazon.com/vpc/latest/privatelink/aws-services-privatelink-support.html))和网关 VPC 端点(仅 S3 和 DynamoDB 支持)。

**网关 VPC 端点**

_使用网关 VPC 端点不会产生时间费用或数据传输费用_。使用网关 VPC 端点时,需要注意它们不能跨 VPC 边界扩展。它们不能用于 VPC 对等、VPN 网络或通过 Direct Connect。

**接口 VPC 端点**

VPC 端点有[时间费用](https://aws.amazon.com/privatelink/pricing/),并且根据 AWS 服务的不同,可能会或可能不会有与通过基础 ENI 进行的数据处理相关的额外费用。为了降低与接口 VPC 端点相关的跨 AZ 数据传输成本,您可以在每个 AZ 中创建一个 VPC 端点。您可以在同一 VPC 中创建多个 VPC 端点,即使它们指向同一个 AWS 服务。

下图显示了 Pods 通过 VPC 端点与 AWS 服务进行通信。

![VPC 端点](../images/vpc_endpoints.png)

## 跨 VPC 的数据传输

在某些情况下,您可能有位于不同 VPC(在同一 AWS 区域内)的工作负载需要相互通信。这可以通过允许流量通过连接到各自 VPC 的互联网网关遍历公共互联网来实现。此类通信可以通过在公共子网中部署 EC2 实例、NAT 网关或 NAT 实例等基础设施组件来启用。但是,包括这些组件的设置将产生处理/传输进出 VPC 数据的费用。如果来自不同 VPC 的流量跨 AZ 移动,则还将产生数据传输费用。下图描述了一个使用 NAT 网关和互联网网关来建立不同 VPC 之间工作负载通信的设置。

![跨 VPC](../images/between_vpcs.png)

### VPC 对等连接

为了降低此类用例的成本,您可以使用[VPC Peering](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html)。使用VPC Peering连接,同一可用区内的网络流量没有数据传输费用。如果流量跨越可用区,将产生成本。尽管如此,VPC Peering方法仍然被推荐用于同一AWS区域内不同VPC之间的成本有效通信。但是,需要注意的是,VPC对等主要适用于1:1 VPC连接,因为它不支持传输性网络。

下图是通过VPC对等连接进行工作负载通信的高级表示。

![Peering](../images/peering.png)

### 传输性网络连接

如前一节所述,VPC Peering连接不支持传输性网络连接。如果您想要连接3个或更多具有传输性网络要求的VPC,那么您应该使用[Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html)(TGW)。这将使您能够克服VPC Peering的局限性或在多个VPC之间建立多个VPC Peering连接所带来的任何操作开销。您将[按小时计费](https://aws.amazon.com/transit-gateway/pricing/)并支付发送到TGW的数据费用。_通过TGW流动的同一区域内的跨可用区流量没有目标成本。_

下图显示了通过TGW在不同VPC但同一AWS区域内的工作负载之间流动的跨可用区流量。

![Transitive](../images/transititive.png)

## 使用服务网格

服务网格提供强大的网络功能,可用于降低EKS集群环境中的网络相关成本。但是,您应该仔细考虑采用服务网格会为您的环境带来的操作任务和复杂性。

### 限制流量到可用区

**使用Istio的区域加权分发**
Istio 使您能够在路由发生后将网络策略应用于流量。这是通过[目标规则](https://istio.io/latest/docs/reference/config/networking/destination-rule/)完成的,例如[基于位置的加权分发](https://istio.io/latest/docs/tasks/traffic-management/locality-load-balancing/distribute/)。使用此功能,您可以根据流量的来源控制流量可以流向某个目的地的权重(以百分比表示)。该流量的来源可以是来自外部(或面向公众的)负载均衡器,也可以是来自集群内部的 Pod。当所有 Pod 端点可用时,将根据加权轮询负载均衡算法选择位置。如果某些端点不健康或不可用,则[位置权重将自动调整](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/locality_weight.html)以反映可用端点的变化。

!!! note
    在实施基于位置的加权分发之前,您应该首先了解您的网络流量模式以及目标规则策略可能对应用程序行为产生的影响。因此,建议您使用诸如 [AWS X-Ray](https://aws.amazon.com/xray/) 或 [Jaeger](https://www.jaegertracing.io/) 等工具建立分布式跟踪机制。

上述详细的 Istio 目标规则也可以应用于管理从负载均衡器到 EKS 集群中 Pod 的流量。基于位置的加权分发规则可以应用于接收来自高可用性负载均衡器(特别是入口网关)流量的服务。这些规则允许您根据流量的区域来源来控制流量的分配 - 在本例中为负载均衡器。如果配置正确,与负载均衡器平均或随机分发流量到不同可用区 Pod 副本相比,跨区域流量的出口将更少。

下面是 Istio 中目标规则资源的代码块示例。如下所示,此资源为来自 `eu-west-1` 区域 3 个不同可用区的入站流量指定了加权配置。这些配置声明,来自给定可用区的大部分入站流量(在本例中为 70%)应该被代理到与其来源相同的可用区的目标位置。
```yaml hl_lines="7-11"
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: express-test-dr
spec:
  host: express-test.default.svc.cluster.local
  trafficPolicy:
    loadBalancer:                        
      localityLbSetting:
        distribute:
        - from: eu-west-1/eu-west-1a/    
          to:
            "eu-west-1/eu-west-1a/*": 70 
            "eu-west-1/eu-west-1b/*": 20
            "eu-west-1/eu-west-1c/*": 10
        - from: eu-west-1/eu-west-1b/*    
          to:
            "eu-west-1/eu-west-1a/*": 20 
            "eu-west-1/eu-west-1b/*": 70
            "eu-west-1/eu-west-1c/*": 10
        - from: eu-west-1/eu-west-1c/*    
          to:
            "eu-west-1/eu-west-1a/*": 20 
            "eu-west-1/eu-west-1b/*": 10
            "eu-west-1/eu-west-1c/*": 70**
    connectionPool:
      http:
        http2MaxRequests: 10
        maxRequestsPerConnection: 10
    outlierDetection:
      consecutiveGatewayErrors: 1
      interval: 1m
      baseEjectionTime: 30s
```


注意!!!
可分配到目标地的最小权重为1%。这是为了在主目标端点变得不健康或不可用的情况下维持故障转移区域和区域。

下图描绘了一个场景,其中在_eu-west-1_区域有一个高可用性负载均衡器,并应用了基于位置的加权分发。此图的目标规则策略配置为将来自_eu-west-1a_的60%流量发送到同一AZ中的Pod,而40%的来自_eu-west-1a_的流量应该发送到eu-west-1b中的Pod。

![Istio流量控制](../images/istio-traffic-control.png)

### 限制流量到可用性区域和节点

**使用Istio的服务内部流量策略**

为了减轻与_外部_传入流量和Pod之间_内部_流量相关的网络成本,您可以结合Istio的目标规则和Kubernetes服务_内部流量策略_。将Istio目标规则与服务内部流量策略相结合的方式将主要取决于3个因素:

* 微服务的角色
* 微服务之间的网络流量模式
* 微服务应如何部署在Kubernetes集群拓扑中

下图显示了嵌套请求的网络流量情况,以及上述策略如何控制流量。

![外部和内部流量策略](../images/external-and-internal-traffic-policy.png)

1. 最终用户向**APP A**发出请求,后者又向**APP C**发出嵌套请求。此请求首先发送到高可用性负载均衡器,该负载均衡器在AZ 1和AZ 2中都有实例,如上图所示。
2. 外部传入的请求然后由Istio虚拟服务路由到正确的目标。
3. 请求被路由后,Istio目标规则会根据请求的来源(AZ 1或AZ 2)控制流量在各AZ之间的分配。
4. 流量然后进入**APP A**的服务,并被代理到相应的Pod端点。如图所示,80%的传入流量发送到AZ 1中的Pod端点,20%的传入流量发送到AZ 2。
5. **APP A**然后向**APP C**发出内部请求。**APP C**的服务启用了内部流量策略(`internalTrafficPolicy``: Local`)。
6. 来自**APP A**(位于*NODE 1*)的内部请求到**APP C**成功,因为**APP C**有可用的节点本地端点。
7. 来自**APP A**(位于*NODE 3*)的内部请求到**APP C**失败,因为**APP C**没有在NODE 3上部署任何副本,因此没有可用的_节点本地端点_。
以下屏幕截图是从此方法的实时示例中捕获的。第一组屏幕截图演示了对 `graphql` 的成功外部请求,以及从 `graphql` 到节点 `ip-10-0-0-151.af-south-1.compute.internal` 上的共置 `orders` 副本的成功嵌套请求。

![Before](../images/before.png)
![Before results](../images/before-results.png)

使用 Istio,您可以验证和导出代理感知的任何 [上游集群](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/intro/terminology) 和端点的统计信息。这可以帮助提供网络流量以及工作负载服务之间分布份额的概况。继续使用相同的示例,可以使用以下命令获取 `graphql` 代理感知的 `orders` 端点:
```bash
kubectl exec -it deploy/graphql -n ecommerce -c istio-proxy -- curl localhost:15000/clusters | grep orders 
```

```bash
...
orders-service.ecommerce.svc.cluster.local::10.0.1.33:3003::**rq_error::0**
orders-service.ecommerce.svc.cluster.local::10.0.1.33:3003::**rq_success::119**
orders-service.ecommerce.svc.cluster.local::10.0.1.33:3003::**rq_timeout::0**
orders-service.ecommerce.svc.cluster.local::10.0.1.33:3003::**rq_total::119**
orders-service.ecommerce.svc.cluster.local::10.0.1.33:3003::**health_flags::healthy**
orders-service.ecommerce.svc.cluster.local::10.0.1.33:3003::**region::af-south-1**
orders-service.ecommerce.svc.cluster.local::10.0.1.33:3003::**zone::af-south-1b**
...
```

在这种情况下，`graphql`代理只知道它与之共享节点的副本的`orders`端点。如果从订单服务中删除`internalTrafficPolicy: Local`设置，并重新运行上述命令，则结果将返回分布在不同节点上的副本的所有端点。此外，通过检查各个端点的`rq_total`，您会注意到网络分布相对均匀。因此，如果端点与在不同可用区运行的上游服务相关联，则跨区域的这种网络分布将导致更高的成本。

如上一节所述，您可以通过使用pod亲和性来共置频繁通信的Pod。
```yaml hl_lines="11-20"
...
spec:
...
  template:
    metadata:
      labels:
        app: graphql
        role: api
        workload: ecommerce
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - orders
            topologyKey: "kubernetes.io/hostname"
      nodeSelector:
        managedBy: karpenter
        billing-team: ecommerce
...
```


当 `graphql` 和 `orders` 副本不共存于同一节点 (`ip-10-0-0-151.af-south-1.compute.internal`) 时,第一个对 `graphql` 的请求成功,如 Postman 截图所示,返回 `200 response code`,而从 `graphql` 到 `orders` 的第二个嵌套请求失败,返回 `503 response code`。

![After](../images/after.png)
![After results](../images/after-results.png)

## 其他资源

* [使用 Istio 解决 EKS 上的延迟和数据传输成本](https://aws.amazon.com/blogs/containers/addressing-latency-and-data-transfer-costs-on-eks-using-istio/)
* [探索拓扑感知提示对亚马逊弹性 Kubernetes 服务网络流量的影响](https://aws.amazon.com/blogs/containers/exploring-the-effect-of-topology-aware-hints-on-network-traffic-in-amazon-elastic-kubernetes-service/)
* [获取亚马逊 EKS 跨可用区 pod 到 pod 网络字节的可见性](https://aws.amazon.com/blogs/containers/getting-visibility-into-your-amazon-eks-cross-az-pod-to-pod-network-bytes/)
* [使用 Istio 优化可用区流量](https://youtu.be/EkpdKVm9kQY)
* [使用拓扑感知路由优化可用区流量](https://youtu.be/KFgE_lNVfz4)
* [使用服务内部流量策略优化 Kubernetes 成本和性能](https://youtu.be/-uiF_zixEro)
* [使用 Istio 和服务内部流量策略优化 Kubernetes 成本和性能](https://youtu.be/edSgEe7Rihc)
* [常见架构的数据传输成本概述](https://aws.amazon.com/blogs/architecture/overview-of-data-transfer-costs-for-common-architectures/)
* [了解 AWS 容器服务的数据传输成本](https://aws.amazon.com/blogs/containers/understanding-data-transfer-costs-for-aws-container-services/)
