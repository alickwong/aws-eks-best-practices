---
日期: 2023-09-22
作者:
  - Lukonde Mwila
---
# 成本优化 - 网络

为高可用性 (HA) 设计系统是实现弹性和容错的最佳实践。在实践中,这意味着将您的工作负载和底层基础设施分散在给定AWS区域的多个可用区 (AZ) 中。确保您的Amazon EKS环境具有这些特性将提高系统的整体可靠性。与此同时,您的EKS环境可能还由各种构造(即VPC)、组件(即ELB)和集成(即ECR和其他容器注册表)组成。

高可用系统和其他特定用例组件的组合可能会对数据传输和处理产生重大影响。这反过来会影响由于数据传输和处理而产生的成本。

下面详述的做法将帮助您设计和优化您的EKS环境,以实现不同领域和用例的成本效益。

## Pod到Pod通信

根据您的设置,Pod之间的网络通信和数据传输可能会对运行Amazon EKS工作负载的总体成本产生重大影响。本节将介绍不同的概念和方法来减轻与Pod间通信相关的成本,同时考虑高可用 (HA) 架构、应用程序性能和弹性。

### 限制流量到可用区

频繁的跨区域流量(在AZ之间分配的流量)可能会对您的网络相关成本产生重大影响。以下是一些策略,介绍如何控制EKS集群中Pod之间的跨区域流量量。

**使用拓扑感知路由(以前称为拓扑感知提示)**

当使用拓扑感知路由时,了解Services、EndpointSlices和`kube-proxy`如何协同工作来路由流量很重要。当创建一个Service时,会创建多个EndpointSlices。每个EndpointSlice都包含一个端点列表,其中包含运行在哪些节点上的Pod地址以及任何其他拓扑信息。`kube-proxy`是一个守护进程,在集群中的每个节点上运行,并履行内部路由的作用,但它是基于它从创建的EndpointSlices中获取的内容。

当在Kubernetes Service上启用[*拓扑感知路由*](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/)时,EndpointSlice控制器将根据集群分布在不同区域的情况,将端点按比例分配到不同的区域。对于这些端点,EndpointSlice控制器还将设置一个_提示_来表示该端点应该为哪个区域提供服务。`kube-proxy`将根据这些_提示_来路由来自某个区域的流量到相应的端点。

下面是一个启用_拓扑感知路由_的Service的代码示例。

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

**使用自动扩缩器:配置节点到特定AZ**

我们强烈建议在多个AZ中运行您的工作负载,以提高应用程序的可靠性,特别是在AZ出现问题时。如果您愿意牺牲可靠性来降低网络相关成本,您可以将节点限制在单个AZ。

要在同一AZ中运行所有Pod,可以在同一AZ中配置工作节点,或者使用[Cluster Autoscaler (CA)](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)在同一AZ中调度Pod。对于[Karpenter](https://karpenter.sh/),使用"[_topology.kubernetes.io/zone"_](http://topology.kubernetes.io/zone%E2%80%9D)并指定要创建工作节点的AZ。例如,下面的Karpenter配置程序片段在us-west-2a AZ中配置节点。

**Karpenter**

```yaml hl_lines="5-9"
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
name: single-az
spec:
  requirements:
  - key: "topology.kubernetes.io/zone"
    operator: In
    values: ["us-west-2a"]
```

**Cluster Autoscaler (CA)**

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

**使用Pod分配和节点亲和性**

另外,如果您有在多个AZ中运行的工作节点,每个节点都会有一个_[topology.kubernetes.io/zone](http://topology.kubernetes.io/zone%E2%80%9D)_标签,其值为其AZ(如us-west-2a或us-west-2b)。您可以利用`nodeSelector`或`nodeAffinity`将Pod调度到单个AZ中的节点。例如,以下清单文件将在运行在AZ us-west-2a中的节点上调度Pod。

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

### 限制流量到节点

在某些情况下,仅限制流量到区域级别是不够的。除了降低成本外,您可能还需要降低某些应用程序之间频繁内部通信的网络延迟。为了实现最佳网络性能并降低成本,您需要一种方法来限制流量到特定节点。例如,微服务A应该始终与节点1上的微服务B通信,即使在高可用 (HA) 设置中也是如此。微服务A在节点1上与微服务B在节点2上通信可能会对这类应用程序的所需性能产生负面影响,特别是如果节点2位于另一个AZ的话。

**使用服务内部流量策略**

为了限制Pod网络流量到节点,您可以使用_[服务内部流量策略](https://kubernetes.io/docs/concepts/services-networking/service-traffic-policy/)_。默认情况下,发送到工作负载服务的流量将随机分布在不同生成的端点上。因此,在HA架构中,这意味着来自微服务A的流量可能会流向任何微服务B的副本,位于任何给定的节点和AZ上。但是,通过将服务的内部流量策略设置为`Local`,流量将被限制到从该流量发起的节点上的端点。此策略规定专门使用节点本地端点。由此推论,该工作负载的网络流量相关成本将低于集群范围内的分发。同时,延迟也会更低,使您的应用程序更加高性能。

!!! note
    需要注意的是,这个功能不能与Kubernetes中的拓扑感知路由结合使用。

下面是设置_内部流量策略_的Service的代码示例。

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

* 为每个通信Pod运行足够的副本
* 使用[拓扑传播约束](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)保持Pod的相对均匀分布
* 使用[pod亲和性规则](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#inter-pod-affinity-and-anti-affinity)来共置通信Pod

在这个例子中,您有2个微服务A的副本和3个微服务B的副本。如果微服务A的副本分布在节点1和节点2之间,而微服务B的3个副本都在节点3上,那么由于`Local`内部流量策略,它们将无法通信,因为没有可用的节点本地端点,流量将被丢弃。

如果微服务B确实有2个副本在节点1和节点2上,那么peer应用程序之间就会有通信。但是,您仍然会有一个微服务B的孤立副本,没有任何peer副本可以通信。

!!! note
    在某些场景中,如上图所示的孤立副本可能不会引起关注,如果它仍然能够服务来自外部的传入流量。

**将服务内部流量策略与拓扑传播约束结合使用**

将_内部流量策略_与_拓扑传播约束_结合使用可以确保您为不同节点上的通信微服务有正确数量的副本。

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

**将服务内部流量策略与Pod亲和性规则结合使用**

另一种方法是在使用服务内部流量策略时利用Pod亲和性规则。使用Pod亲和性,您可以影响调度程序,因为某些Pod由于频繁通信而需要共置。通过对某些Pod应用严格的调度约束(`requiredDuringSchedulingIgnoredDuringExecution`),这将在调度程序将Pod放置在节点上时给出更好的Pod共置结果。

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

## 负载均衡器到Pod通信

EKS工作负载通常由负载均衡器前端,该负载均衡器将流量分配到EKS集群中的相关Pod。您的架构可能包括内部和/或外部面向的负载均衡器。根据您的架构和网络流量配置,负载均衡器和Pod之间的通信可能会对数据传输费用产生重大影响。

您可以使用[AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller)自动管理ELB资源(ALB和NLB)的创建。您在这种设置中产生的数据传输费用将取决于网络流量的路径。AWS Load Balancer Controller支持两种网络流量模式,_实例模式_和_IP模式_。

使用_实例模式_时,将在集群中的每个节点上打开一个NodePort。负载均衡器将然后在节点之间均匀代理流量。如果一个节点上有目标Pod运行,那么就不会产生任何数据传输费用。但是,如果目标Pod位于与接收流量的NodePort不同的节点和不同的AZ上,那么就会有从`kube-proxy`到目标Pod的额外网络跳跃。在这种情况下,将产生跨AZ数据传输费用。由于流量在节点之间的均匀分布,很可能会产生额外的数据传输费用,涉及从`kube-proxy`到相关目标Pod的跨区域网络流量跳跃。

下图描述了从负载均衡器到NodePort,然后从`kube-proxy`到不同AZ中另一个节点上的目标Pod的网络路径。这是_实例模式_设置的一个示例。

![LB to Pod](../images/lb_2_pod.png)

使用_IP模式_时,网络流量直接从负载均衡器代理到目标Pod。因此,这种方法_不会产生任何数据传输费用_。

!!! tip
    建议您将负载均衡器设置为_IP流量模式_以减少数据传输费用。对于这种设置,也很重要的是确保您的负载均衡器部署在VPC中的所有子网上。

下图描述了网络流量从负载均衡器流向网络_IP模式_中的Pod的路径。

![IP mode](../images/ip_mode.png)

## 从容器注册表传输数据

### Amazon ECR

向Amazon ECR私有注册表传输数据是免费的。_同区域内的数据传输不收费_,但向互联网和跨区域传输数据将按互联网数据传输费率在传输的两端收费。

您应该利用ECR内置的[镜像复制功能](https://docs.aws.amazon.com/AmazonECR/latest/userguide/replication.html)将相关的容器镜像复制到与您的工作负载相同的区域。这样,复制只会收费一次,所有同区域(区域内)的镜像拉取都是免费的。

您可以通过_使用[接口VPC端点](https://docs.aws.amazon.com/whitepapers/latest/aws-privatelink/what-are-vpc-endpoints.html)连接到同区域的ECR存储库_,进一步降低从ECR拉取镜像(数据传输出)的成本。连接到ECR的公共AWS端点(通过NAT网关和互联网网关)的替代方法将产生更高的数据处理和传输成本。下一节将更详细地介绍如何降低工作负载与AWS服务之间的数据传输成本。

如果您运行的工作负载有特别大的镜像,您可以构建自己的定制Amazon Machine Images (AMIs),预缓存容器�像。这可以减少初始镜像拉取时间和从容器注册表到EKS工作节点的潜在数据传输成本。

## 向互联网和AWS服务传输数据

将Kubernetes工作负载与其他AWS服务或第三方工具和平台集成是一种常见做法,通过互联网进行。用于路由流量到相关目的地的底层网络基础设施可能会影响数据传输过程中产生的成本。

### 使用NAT网关

NAT网关是执行网络地址转换(NAT)的网络组件。下图描述了EKS集群中的Pod与其他AWS服务(Amazon ECR、DynamoDB和S3)以及第三方平台进行通信。在这个例子中,Pod在不同AZ的私有子网中运行。为了发送和接收来自互联网的流量,部署了一个位于一个AZ公有子网中的NAT网关,允许任何具有私有IP地址的资源共享一个公有IP地址访问互联网。这个NAT网关又与互联网网关组件通信,允许数据包发送到最终目的地。

![NAT Gateway](../images/nat_gw.png)

在使用NAT网关进行此类用例时,_您可以通过在每个AZ部署一个NAT网关来最小化数据传输成本_。这样,路由到互联网的流量将通过同一AZ中的NAT网关,避免了跨AZ数据传输。但是,尽管您将节省跨AZ数据传输的成本,这种设置的影响是您的架构中会产生额外的NAT网关成本。

这种推荐的方法如下图所示。

![Recommended approach](../images/recommended_approach.png)

### 使用VPC端点

为了进一步降低这种架构的成本,_您应该使用[VPC端点](https://docs.aws.amazon.com/whitepapers/latest/aws-privatelink/what-are-vpc-endpoints.html)在您的工作负载和AWS服务之间建立连接_。VPC端点允许您在不通过互联网传输数据/网络数据包的情况下从VPC内部访问AWS服务。所有流量都是内部的,并保持在AWS网络内。有两种类型的VPC端点:接口VPC端点([许多AWS服务支持](https://docs.aws.amazon.com/vpc/latest/privatelink/aws-services-privatelink-support.html))和网关VPC端点(只有S3和DynamoDB支持)。

**网关VPC端点**

_网关VPC端点没有每小时费用或数据传输费用_。使用网关VPC端点时,需要注意它们不能跨VPC边界扩展。它们不能用于VPC对等、VPN网络或通过Direct Connect。

**接口VPC端点**

VPC端点有[每小时收费](https://aws.amazon.com/privatelink/pricing/),并且根据AWS服务的不同,可能会或可能不会有与基础ENI相关的额外数据处理费用。为了降低与接口VPC端点相关的跨AZ数据传输成本,您可以在每个AZ创建一个VPC端点。您可以在同一VPC中创建多个VPC端点,即使它们指向同一个AWS服务。

下图显示了Pod通过VPC端点与AWS服务进行通信。

![VPC Endpoints](../images/vpc_endpoints.png)

## 跨VPC传输数据

在某些情况下,您可能有位于不同VPC(在同一AWS区域内)的工作负载需要相互通信。这可以通过允许流量通过连接到各自VPC的互联网网关来跨越公共互联网来实现。这种通信可以通过在公有子网中部署EC2实例、NAT网关或NAT实例来启用。但是,包含这些组件的设置将产生处理/传输进出VPC的数据的费用。如果来自不同VPC的流量在跨AZ移动,则在数据传输过程中还会产生额外费用。下图描述了使用NAT网关和互联网网关在不同VPC之间建立通信的设置。

![Between VPCs](../images/between_vpcs.png)

### VPC对等连接

为了降低此类用例的成本,您可以使用[VPC对等](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html)。使用VPC对等连接,在同一AZ内的网络流量不会产生任何数据传输费用。如果流量跨AZ,将产生费用。尽管如此,VPC对等方法仍然是在同一AWS区域内跨VPC工作负载通信的推荐成本效益方法。但是,需要注意的是,VPC对等主要适用于1:1 VPC连接,因为它不支持传递网络。

下图是工作负载通过VPC对等连接进行通信的高级表示。

![Peering](../images/peering.png)

### 传递网络连接

正如前一节所指出的,VPC对等连接不支持传递网络连接。如果您想连接3个或更多具有传递网络需求的VPC,那么您应该使用[Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html) (TGW)。这将使您能够克服VPC对等或多个VPC对等连接之间的操作开销。您[按小时收费](https://aws.amazon.com/transit-gateway/pricing/)并根据发送到TGW的数据收费。_通过TGW流动的跨AZ流量没有目标成本_。

下图显示了通过TGW在不同VPC但同一AWS区域内的工作负载之间流动的跨AZ流量。

![Transitive](../images/transititive.png)

## 使用服务网格

服务网格提供强大的网络功能,可用于降低EKS集群环境中的网络相关成本。但是,您应该仔细考虑采用服务网格会给您的环境带来的操作任务和复杂性。

### 限制流量到可用区

**使用Istio的区域加权分发**

Istio可以在路由发生后应用网络策略。这是使用[目标规则](https://istio.io/latest/docs/reference/config/networking/destination-rule/)完成的,例如[区域加权分发](https://istio.io/latest/docs/tasks/traffic-management/locality-load-balancing/distribute/)。使用这个功能,您可以根据流量的源头(外部或公共负载均衡器,或集群内部的Pod)来控制流量到某个目的地的权重(以百分比表示)。当所有Pod端点可用时,区域将根据加权轮询负载均衡算法进行选择。如果某些端点不健康或不可用,[区域权重将自动调整](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/locality_weight.html)以反映可用端点的变化。

!!! note
    在实现区域加权分发之前,您应该首先了解您的网络流量模式以及目标规则策略可能对应用程序行为产生的影响。因此,拥有像[AWS X-Ray](https://aws.amazon.com/xray/)或[Jaeger](https://www.jaegertracing.io/)这样的分布式跟踪机制非常重要。

上述Istio目标规则也可以应用于管理从负载均衡器到EKS集群中Pod的流量。区域加权分发规则可以应用于接收来自高可用负载均衡器(特别是Ingress网关)流量的Service。这些规则允许您根据流量的区域源头(在这种情况下是负载均衡器)来控制流量的分配 - 如果配置正确,与随机或均匀分发到不同AZ中Pod副本的负载均衡器相比,跨区域流出流量会更少。

下面是Istio目标规则资源的代码示例。如下所示,该资源为来自`eu-west-1`区域3个不同AZ的传入流量指定了加权配置。这些配置声明,来自给定AZ的大部分(在本例中为70%)传入流量应该代理到同一AZ中的目标。

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

!!! note
    可分配到目标的最小权重为1%。这样做的原因是为了在主目标端点不健康或不可用的情况下维持故障转移区域和区域。

下图描述了在_eu-west-1_区域有一个高可用负载均衡器,并应用了区域加权分发的场景。该图的目标规则配置为将60%来自_eu-west-1a_的流量发送到同一AZ中的Pod,而40%来自_eu-west-1a_的流量应该发送到eu-west-1b中的Pod。

![Istio Traffic Control](../images/istio-traffic-control.png)

### 限制流量到可用区和节点

**将服务内部流量策略与Istio结合使用**

为了减轻与_外部_传入流量和_内部_Pod之间通信相关的网络成本,您可以结合使用Istio的目标规则和Kubernetes Service _内部流量策略_。将Istio目标规则与服务内部流量策略结合使用的方式在很大程度上取决于3个因素:

* 微服务的角色
* 跨微服务的网络流量模式
* 微服务应如何部署在Kubernetes集群拓扑中

下图显示了嵌套请求的网络流量情况,以及上述政策如何控制流量。

![External and Internal traffic policy](../images/external-and-internal-traffic-policy.png)

1. 最终用户向**APP A**发出请求,**APP A**又向**APP C**发出嵌套请求。这个请求首先发送到一个高可用负载均衡器,该负载均衡器在AZ 1和AZ 2中都有实例,如上图所示。
2. 外部传入请求然后由Istio虚拟服务路由到正确的目的地。
3. 请求被路由后,Istio目标规则控制了基于流量源头(AZ 1或AZ 2)的流量分配。
4. 流量然后进入**APP A**的Service,并被代理到相应的Pod端点。如图所示,80%的传入流量被发送到AZ 1中的Pod端点,20%的传入流量被发送到AZ 2。
5. **APP A**然后向**APP C**发出内部请求。**APP C**的Service启用了内部流量策略(`internalTrafficPolicy``: Local`)。
6. 来自**APP A**(在*NODE 1*上)到**APP C**的内部请求成功,因为**APP C**有可用的节点本地端点。
7. 来自**APP A**(在*NODE 3*上)到**APP C**的内部请求失败,因为**APP C**没有在NODE 3上的可用_节点本地端点_。

下面的截图是从一个实时示例中捕获的。第一组截图展示了成功的外部请求到`graphql`以及从`graphql`到共置的`orders`副本在节点`ip-10-0-0-151.af-south-1.compute.internal`上的成功嵌套请求。

![Before](../images/before.png)
![Before results](../images/before-results.png)

使用Istio,您可以验证和导出代理感知的任何[上游集群](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/intro/terminology)和端点的统计信息。这可以帮助您了解网络流量的情况,以及工作负载服务之间的分发份额。继续使用相同的示例,可以使用以下命令获取`graphql`代理感知的`orders`端点:

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

在这种情况下,`graphql`代理只知道与它共享节点的`orders`端点。如果您从`orders`服务中删除`internalTrafficPolicy: Local`设置,并重新运行类似的命令,那么结果将返回分布在不同节点上的所有副本端点。此外,通过检查相应端点的`rq_total`,您会注意到网络分发相对均匀。因此,如果这些端点与运行在不同AZ中的上游服务相关联,那么这种跨区域的网络分发将导致更高的成本。

如前一节所述,您可以通过使用pod亲和性来共置频繁通信的Pod。

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

当`graphql`和`orders`副本不在同一节点(`ip-10-0-0-151.af-south-1.compute.internal`)上时,对`graphql`的第一个请求成功,如Postman截图中的`200 response code`所示,而从`graphql`到`orders`的嵌套请求则失败,返回`503 response code`。

![After](../images/after.png)
![After results](../images/after-results.png)

## 其他资源

* [使用Istio解决EKS上的延迟和数据传输成本](https://aws.amazon.com/blogs/containers/addressing-latency-and-data-transfer-costs-on-eks-using-istio/)
* [探索拓扑感知提示对Amazon Elastic Kubernetes Service网络流量的影响](https://aws.amazon.com/blogs/containers/exploring-the-effect-of-topology-aware-hints-on-network-traffic-in-amazon-elastic-kubernetes-service/)
* [获取您的Amazon EKS跨AZ Pod到Pod网络字节的可见性](https://aws.amazon.com/blogs/containers/getting-visibility-into-your-amazon-eks-cross-az-pod-to-pod-network-bytes/)
* [使用Istio优化AZ流量](https://youtu.be/EkpdKVm9kQY)
* [使用拓扑感知路由优化AZ流量](https://youtu.be/KFgE_lNVfz4)
* [使用服务内部流量策略优化Kubernetes成本和性能](https://youtu.be/-uiF_zixEro)
* [使用Istio和服务内部流量策略优化Kubernetes成本和性能](https://youtu.be/edSgEe7Rihc)
* [常见架构的数据传输成本概述](https://aws.amazon.com/blogs/architecture/overview-of-data-transfer-costs-for-common-architectures/)
* [了解AWS容器服务的数据传输成本](https://aws.amazon.com/blogs/containers/understanding-data-transfer-costs-for-aws-container-services/)