# VPC 和子网注意事项

运行 EKS 集群需要了解 AWS VPC 网络以及 Kubernetes 网络。

我们建议您在设计 VPC 或将集群部署到现有 VPC 之前,先了解 EKS 控制平面通信机制。

在设计用于 EKS 的 VPC 和子网时,请参考[集群 VPC 注意事项](https://docs.aws.amazon.com/eks/latest/userguide/network_reqs.html)和[Amazon EKS 安全组注意事项](https://docs.aws.amazon.com/eks/latest/userguide/sec-group-reqs.html)。

## 概述

### EKS 集群架构

EKS 集群由两个 VPC 组成:

* 托管 Kubernetes 控制平面的 AWS 管理 VPC。此 VPC 不会出现在客户账户中。
* 托管 Kubernetes 节点的客户管理 VPC。这是容器运行以及其他客户管理 AWS 基础设施(如集群使用的负载均衡器)的位置。此 VPC 出现在客户账户中。您需要在创建集群之前创建客户管理 VPC。如果您没有提供 VPC,eksctl 会创建一个。

客户 VPC 中的节点需要能够连接到 AWS VPC 中托管的 API 服务器端点。这允许节点向 Kubernetes 控制平面注册并接收运行应用程序 Pod 的请求。

节点通过 (a) EKS 公共端点或 (b) EKS 管理的跨账户[弹性网络接口](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html)(X-ENI)连接到 EKS 控制平面。创建集群时,您需要指定至少两个 VPC 子网。EKS 在创建集群时指定的每个子网(也称为集群子网)中放置一个 X-ENI。Kubernetes API 服务器使用这些跨账户 ENI 与部署在客户管理集群 VPC 子网上的节点进行通信。

![集群网络的一般说明,包括负载均衡器、节点和 pod。](./image.png)

当节点启动时,执行 EKS 引导脚本并安装 Kubernetes 节点配置文件。作为每个实例启动过程的一部分,启动容器运行时代理、kubelet 和 Kubernetes 节点代理。

为了注册节点,Kubelet 会联系 Kubernetes 集群端点。它会与 VPC 外部的公共端点或 VPC 内部的私有端点建立连接。Kubelet 定期接收 API 指令并提供状态更新和心跳。

### EKS 控制平面通信

EKS 有两种方式控制对[集群端点](https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html)的访问。端点访问控制允许您选择端点是可从公共互联网访问还是只能通过您的 VPC 访问。您可以启用公共端点(这是默认设置)、私有端点或同时启用两者。

集群 API 端点的配置决定了节点与控制平面通信的路径。请注意,这些端点设置可以随时通过 EKS 控制台或 API 进行更改。

#### 公共端点

这是新 Amazon EKS 集群的默认行为。仅启用集群的公共端点时,源自集群 VPC 内部的 Kubernetes API 请求(例如节点到控制平面的通信)会离开 VPC,但不会离开 Amazon 网络。为了使节点能够连接到控制平面,它们必须具有公共 IP 地址,并且必须有到互联网网关或 NAT 网关的路由,以便使用 NAT 网关的公共 IP 地址。

#### 公共和私有端点

同时启用公共和私有端点时,来自 VPC 内部的 Kubernetes API 请求通过 VPC 内的 X-ENI 与控制平面通信。您的集群 API 服务器可从互联网访问。

#### 私有端点

仅启用私有端点时,互联网上没有对您的 API 服务器的公共访问。所有到您集群 API 服务器的流量必须来自您的集群 VPC 或连接的网络。节点通过 VPC 内的 X-ENI 与 API 服务器通信。请注意,集群管理工具必须能够访问私有端点。了解有关[如何从 Amazon VPC 外部连接到私有 Amazon EKS 集群端点](https://aws.amazon.com/premiumsupport/knowledge-center/eks-private-cluster-endpoint-vpc/)的更多信息。

请注意,集群的 API 服务器端点由公共 DNS 服务器解析为 VPC 中的私有 IP 地址。过去,端点只能从 VPC 内部解析。

### VPC 配置

Amazon VPC 支持 IPv4 和 IPv6 寻址。Amazon EKS 默认支持 IPv4。VPC 必须与 IPv4 CIDR 块相关联。您还可以选择将多个 IPv4 [无类域间路由](http://en.wikipedia.org/wiki/CIDR_notation)(CIDR)块和多个 IPv6 CIDR 块关联到您的 VPC。创建 VPC 时,您必须从 [RFC 1918](http://www.faqs.org/rfcs/rfc1918.html)中指定的私有 IPv4 地址范围为 VPC 指定一个 IPv4 CIDR 块。允许的块大小介于 `/16` 前缀(65,536 个 IP 地址)和 `/28` 前缀(16 个 IP 地址)之间。

创建新 VPC 时,您可以附加一个 IPv6 CIDR 块,在更改现有 VPC 时最多可以附加五个。IPv6 CIDR 块的前缀长度可以介于 /44 和 /60 之间,对于 IPv6 子网,它可以介于 /44 和 /64 之间。您可以从 Amazon 维护的 IPv6 地址池中请求 IPv6 CIDR 块。有关更多信息,请参阅 VPC 用户指南中的[VPC CIDR 块](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-cidr-blocks.html)部分。

Amazon EKS 集群支持 IPv4 和 IPv6。默认情况下,EKS 集群使用 IPv4 IP。在创建集群时指定 IPv6 将启用 IPv6 集群。IPv6 集群需要双堆栈 VPC 和子网。

Amazon EKS 建议您在创建集群时使用至少两个位于不同可用区的子网。您在创建集群时传递的子网称为集群子网。创建集群时,Amazon EKS 在您指定的子网中创建最多 4 个跨账户(x-account 或 x-ENI) ENI。x-ENI 始终部署,用于集群管理流量,如日志传递、exec 和代理。有关完整的[VPC 和子网要求](https://docs.aws.amazon.com/eks/latest/userguide/network_reqs.html#network-requirements-subnets)详细信息,请参阅 EKS 用户指南。

Kubernetes 工作节点可以在集群子网中运行,但不建议这样做。在[集群升级](https://aws.github.io/aws-eks-best-practices/upgrades/#verify-available-ip-addresses)期间,Amazon EKS 在集群子网中配置额外的 ENI。当您的集群扩展时,工作节点和 pod 可能会消耗集群子网中可用的 IP。因此,为了确保有足够的可用 IP,您可能需要考虑使用具有 /28 子网掩码的专用集群子网。

Kubernetes 工作节点可以在公共或私有子网中运行。子网是公共还是私有取决于子网中的流量是否通过[互联网网关](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html)路由。公共子网有一个通往互联网的路由表条目,但私有子网没有。

从其他地方到达您的节点的流量称为*入站*。从节点出发离开网络的流量称为*出站*。在配置有互联网网关的子网中具有公共或弹性 IP 地址(EIP)的节点允许来自 VPC 外部的入站流量。私有子网通常包含[NAT 网关](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html),它只允许来自 VPC 内部的入站流量进入节点,同时仍允许来自节点的出站流量离开 VPC。

在 IPv6 世界中,每个地址都是可路由的互联网地址。与节点和 pod 关联的 IPv6 地址是公共的。通过在 VPC 中实现[出口专用互联网网关(EIGW)](https://docs.aws.amazon.com/vpc/latest/userguide/egress-only-internet-gateway.html)来支持私有子网,允许出站流量但阻止所有入站流量。有关实施 IPv6 子网的最佳实践,请参阅[VPC 用户指南](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Scenario2.html)。

### 您可以通过三种不同的方式配置 VPC 和子网:

#### 仅使用公共子网

在同一公共子网中创建节点和入口资源(如负载均衡器)。使用 [`kubernetes.io/role/elb`](http://kubernetes.io/role/elb) 标记公共子网以构建面向互联网的负载均衡器。在此配置中,集群端点可以配置为公共、私有或同时公共和私有。

#### 使用私有和公共子网

节点在私有子网中创建,而入口资源在公共子网中实例化。您可以启用公共、私有或同时公共和私有访问集群端点。根据集群端点的配置,节点流量将通过 NAT 网关或 ENI 进入。

#### 仅使用私有子网

节点和入口都在私有子网中创建。使用 [`kubernetes.io/role/internal-elb`](http://kubernetes.io/role/internal-elb:1) 子网标签来构建内部负载均衡器。访问您的集群端点将需要 VPN 连接。您必须为 EC2 和所有 Amazon ECR 和 S3 存储库激活 [AWS PrivateLink](https://docs.aws.amazon.com/vpc/latest/userguide/endpoint-service.html)。只应启用集群的私有端点。我们建议在配置私有集群之前仔细阅读[EKS 私有集群要求](https://docs.aws.amazon.com/eks/latest/userguide/private-clusters.html)。

### 跨 VPC 通信

在需要多个 VPC 和部署到这些 VPC 的单独 EKS 集群的情况下,有许多场景。

您可以使用 [Amazon VPC Lattice](https://aws.amazon.com/vpc/lattice/) 在多个 VPC 和账户之间一致且安全地连接服务(无需依赖 VPC 对等、AWS PrivateLink 或 AWS Transit Gateway 等其他连接性)。在[此处](https://aws.amazon.com/blogs/networking-and-content-delivery/build-secure-multi-account-multi-vpc-connectivity-for-your-applications-with-amazon-vpc-lattice/)了解更多信息。

![Amazon VPC Lattice,流量流](./vpc-lattice.gif)

Amazon VPC Lattice 在 IPv4 和 IPv6 的链路本地地址空间中运行,提供可能具有重叠 IPv4 地址的服务之间的连接。为了提高操作效率,我们强烈建议将 EKS 集群和节点部署到不重叠的 IP 范围。如果您的基础设施包括具有重叠 IP 范围的 VPC,您需要相应地设计您的网络。我们建议使用[私有 NAT 网关](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html#nat-gateway-basics)或 VPC CNI 与[过境网关](https://docs.aws.amazon.com/whitepapers/latest/aws-vpc-connectivity-options/aws-transit-gateway.html)结合使用的[自定义网络](../custom-networking/index.md)模式来集成 EKS 上的工作负载,以解决重叠 CIDR 挑战,同时保留可路由的 RFC1918 IP 地址。

![使用自定义网络的私有 Nat 网关,流量流](./private-nat-gw.gif)

如果您是服务提供商,并希望与您客户的 VPC 在单独的账户中共享您的 Kubernetes 服务和入口(ALB 或 NLB),请考虑使用 [AWS PrivateLink](https://docs.aws.amazon.com/vpc/latest/privatelink/privatelink-share-your-services.html)(也称为端点服务)。

### 跨多个账户共享 VPC

许多企业采用共享 Amazon VPC 作为简化网络管理、降低成本和提高 AWS 组织中多个 AWS 账户的安全性的一种方式。他们利用 AWS Resource Access Manager (RAM) 与个别 AWS 账户、组织单元(OU)或整个 AWS 组织安全地共享受支持的[AWS 资源](https://docs.aws.amazon.com/ram/latest/userguide/shareable.html)。

您可以使用 AWS RAM 在来自另一个 AWS 账户的共享 VPC 子网中部署 Amazon EKS 集群、托管节点组和其他支持的 AWS 资源(如负载均衡器、安全组、端点等)。下图描述了一个示例高级架构。这允许中央网络团队控制 VPC、子网等网络构件,同时允许应用程序或平台团队在各自的 AWS 账户中部署 Amazon EKS 集群。此方案的完整演练可在[此 github 存储库](https://github.com/aws-samples/eks-shared-subnets)中找到。

![在跨 AWS 账户共享的 VPC 子网中部署 Amazon EKS。](./eks-shared-subnets.png)

#### 使用共享子网时的注意事项

* Amazon EKS 集群和工作节点可以在同一 VPC 的共享子网中创建。Amazon EKS 不支持跨多个 VPC 创建集群。

* Amazon EKS 使用 AWS VPC 安全组(SG)来控制 Kubernetes 控制平面与集群工作节点之间的流量。安全组还用于控制工作节点与其他 VPC 资源以及外部 IP 地址之间的流量。您必须在参与账户中创建这些安全组。确保您打算用于 pod 的安全组也位于参与账户中。您可以配置安全组的入站和出站规则,以允许必要的流量进出位于中央 VPC 账户中的安全组。

* 在 Amazon EKS 集群所在的参与账户中创建 IAM 角色和相关策略。这些 IAM 角色和策略对于授予 Amazon EKS 管理的 Kubernetes 集群以及在 Fargate 上运行的节点和 pod 所需的权限至关重要。这些权限使 Amazon EKS 能够代表您调用其他 AWS 服务。

* 您可以采取以下方法,允许 k8s pod 跨账户访问 AWS 资源,如 Amazon S3 存储桶、Dynamodb 表等:
    * **基于资源的策略方法**:如果 AWS 服务支持资源策略,您可以添加适当的基于资源的策略,以允许跨账户访问分配给 kubernetes pod 的 IAM 角色。在这种情况下,OIDC 提供程序、IAM 角色和权限策略存在于应用程序账户中。要查找支持资源策略的 AWS 服务,请参考[与 IAM 配合使用的 AWS 服务](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_aws-services-that-work-with-iam.html),查看"资源基础"列中有"是"的服务。

    * **OIDC 提供程序方法**:IAM 资源(如 OIDC 提供程序、IAM 角色、权限和信任策略)将在资源所在的其他参与 AWS 账户中创建。这些角色将分配给应用程序账户中的 Kubernetes pod,以便它们可以访问跨账户资源。请参考[用于 Kubernetes 服务帐户的跨账户 IAM 角色](https://aws.amazon.com/blogs/containers/cross-account-iam-roles-for-kubernetes-service-accounts/)博客,了解此方法的完整演练。

* 您可以将 Amazon Elastic Loadbalancer (ELB)资源(ALB 或 NLB)部署到应用程序或中央网络账户,以将流量路由到 k8s pod。请参阅[通过跨账户负载均衡器公开 Amazon EKS Pod](https://aws.amazon.com/blogs/containers/expose-amazon-eks-pods-through-cross-account-load-balancer/)演练,了解在中央网络账户中部署 ELB 资源的详细说明。此选项提供了增强的灵活性,因为它授予中央网络账户对负载均衡器资源安全配置的完全控制权。

* 当使用 Amazon VPC CNI 的`自定义网络功能`时,您需要使用中央网络账户中列出的可用区(AZ) ID 映射来创建每个`ENIConfig`。这是由于在每个 AWS 账户中,物理 AZ 到 AZ 名称的映射是随机的。

### 安全组

[*安全组*](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html)控制允许到达和离开其关联资源的流量。Amazon EKS 使用安全组来管理[控制平面和节点之间的通信](https://docs.aws.amazon.com/eks/latest/userguide/sec-group-reqs.html)。创建集群时,Amazon EKS 会创建一个名为 `eks-cluster-sg-my-cluster-uniqueID` 的安全组。EKS 将这些安全组关联到托管的 ENI 和节点。默认规则允许节点和集群之间的所有流量自由流动,并允许所有出站流量到任何目的地。

创建集群时,您可以指定自己的安全组。请参阅[安全组建议](https://docs.aws.amazon.com/eks/latest/userguide/sec-group-reqs.html),了解在指定自己的安全组时的注意事项。

## 建议

### 考虑多可用区部署

AWS 区域提供多个物理隔离的可用区(AZ),这些可用区通过低延迟、高吞吐量和高度冗余的网络连接。借助可用区,您可以设计和操作在可用区之间自动故障转移而不中断的应用程序。Amazon EKS 强烈建议将 EKS 集群部署到多个可用区。创建集群时,请考虑指定至少两个可用区的子网。

Kubelet 运行在节点上时,会自动向节点对象添加标签,如 [`topology.kubernetes.io/region=us-west-2`和`topology.kubernetes.io/zone=us-west-2d`](http://topology.kubernetes.io/region=us-west-2,topology.kubernetes.io/zone=us-west-2d)。我们建议结合[Pod 拓扑传播约束](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)使用节点标签,以控制 Pod 在区域之间的分布。这些提示使 Kubernetes [调度器](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)能够放置 Pod,以获得更好的预期可用性,降低整个工作负载受相关故障影响的风险。请参阅[将节点分配给 Pod](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector),了解节点选择器和 AZ 传播约束的示例。

您可以在创建节点时定义子网或可用区。如果未配置子网,节点将放置在集群子网中。EKS 对托管节点组的支持会自动将节点分散到可用容量的多个可用区。[Karpenter](https://karpenter.sh/)将通过根据工作负载定义的拓扑传播限制来扩展节点到指定的 AZ。

AWS Elastic Load Balancers 由 AWS Load Balancer Controller 为 Kubernetes 集群管理。它为 Kubernetes ingress 资源配置应用程序负载均衡器(ALB),为 Kubernetes 类型为 Loadbalancer 的服务配置网络负载均衡器(NLB)。Elastic Load Balancer 控制器使用[标签](https://aws.amazon.com/premiumsupport/knowledge-center/eks-vpc-subnet-discovery/)来发现子网。ELB 控制器需要至少两个可用区(AZ)才能成功配置入口资源。考虑在至少两个 AZ 中设置子网,以利用地理冗余的安全性和可靠性。

### 将节点部署到私有子网

包含公共和私有子网的 VPC 是在 EKS 上部署 Kubernetes 工作负载的理想方法。考虑设置至少两个公共子网和两个私有子网,位于两个不同的可用区。公共子网的相关路由表包含通往互联网网关的路由。Pod 可以通过 NAT 网关与互联网交互。在 IPv6 环境中,私有子网由[出口专用互联网网关](https://docs.aws.amazon.com/vpc/latest/userguide/egress-only-internet-gateway.html)(EIGW)支持。

在私有子网中实例化节点提供了对节点流量的最大控制,适用于绝大多数 Kubernetes 应用程序。入口资源(如负载均衡器)在公共子网中实例化,并将流量路由到在私有子网上运行的 Pod。

如果您需要严格的安全性和网络隔离,请考虑仅使用私有模式。在此配置中,在 VPC 的 AWS 区域内部署三个私有子网,位于不同的可用区。部署到子网的资源无法访问互联网,互联网也无法访问子网中的资源。为了使您的 Kubernetes 应用程序访问其他 AWS 服务,您必须配置 PrivateLink 接口和/或网关端点。您可以使用 AWS Load Balancer Controller 设置内部负载均衡器,将流量重定向到 Pod。必须为控制器标记私有子网([`kubernetes.io/role/internal-elb: 1`](http://kubernetes.io/role/internal-elb))以配置负载均衡器。为了使节点能够注册到集群,集群端点必须设置为私有模式。有关完整要求和注意事项,请访问[私有集群指南](https://docs.aws.amazon.com/eks/latest/userguide/private-clusters.html)。

### 考虑集群端点的公共和私有模式

Amazon EKS 提供公共模式、公共和私有模式以及私有模式三种集群端点模式。默认模式是公共模式,但我们建议将集群端点配置为公共和私有模式。此选项允许 Kubernetes API 调用在集群 VPC 内部(如节点到控制平面的通信)使用私有 VPC 端点,流量保持在集群 VPC 内。但是,您的集群 API 服务器可从互联网访问。但是,我们强烈建议限制可以使用公共端点的 CIDR 块。[了解如何配置公共和私有端点访问,包括限制 CIDR 块。](https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html#modify-endpoint-access)

当您需要安全性和网络隔离时,我们建议使用私有端点。我们建议使用[EKS 用户指南](https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html#private-access)中列出的任何一种选项来私下连接到 API 服务器。

### 谨慎配置安全组

Amazon EKS 支持使用自定义安全组。任何自定义安全组都必须允许节点与 Kubernetes 控制平面之间的通信。请检查[端口要求](https://docs.aws.amazon.com/eks/latest/userguide/sec-group-reqs.html),并在您的组织不允许开放通信时手动配置规则。

EKS 在集群创建期间应用您提供的自定义安全组到托管接口(X-ENI)。但是,它不会立即将它们与节点关联。在创建节点组时,强烈建议[手动关联自定义安全组](https://eksctl.io/usage/schema/#nodeGroups-securityGroups)。请考虑启用[securityGroupSelectorTerms](https://karpenter.sh/docs/concepts/nodeclasses/#specsecuritygroupselectorterms),以使 Karpenter 节点模板在节点自动扩展期间发现自定义安全组。

我们强烈建议创建一个安全组,允许所有节点间通信流量。在引导过程中,节点需要对互联网的出站连接才能访问集群端点。评估出站访问要求,如内部连接和容器注册表访问,并相应地设置规则。在投入生产之前,我们强烈建议您仔细检查开发环境中的连接。

### 在每个可用区部署 NAT 网关

如果您在私有子网(IPv4 和 IPv6)中部署节点,请考虑在每个可用区(AZ)创建一个 NAT 网关,以确保区域独立的架构并减少跨 AZ 支出。每个 AZ 中的 NAT 网关都具有冗余性。

### 使用 Cloud9 访问私有集群

AWS Cloud9 是一个可以在没有入站访问的私有子网中安全运行的基于 web 的 IDE,使用 AWS Systems Manager。Cloud9 实例上也可以禁用出站访问。[了解更多关于使用 Cloud9 访问私有集群和子网的信息。](https://aws.amazon.com/blogs/security/isolating-network-access-to-your-aws-cloud9-environments/)

![AWS Cloud9 控制台连接到无入站 EC2 实例的说明。](./image-2.jpg)