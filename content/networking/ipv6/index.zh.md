# 在 IPv6 EKS 集群上运行

<iframe width="560" height="315" src="https://www.youtube.com/embed/zdXpTT0bZXo" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

IPv6 模式下的 EKS 解决了大规模 EKS 集群中常见的 IPv4 耗尽挑战。EKS 对 IPv6 的支持主要集中在解决 IPv4 地址空间有限的问题,这是许多客户提出的重要问题,与 Kubernetes 的"[IPv4/IPv6 双栈](https://kubernetes.io/docs/concepts/services-networking/dual-stack/)"功能不同。
EKS/IPv6 还将提供使用 IPv6 CIDR 跨网络边界互连的灵活性,从而最大程度地减少 CIDR 重叠的可能性,从而解决双重问题(集群内部和跨集群)。
在 IPv6 模式下部署 EKS 集群(--ip-family ipv6)是不可逆的。简单地说,EKS IPv6 支持在整个集群生命周期内都是启用的。

在 IPv6 EKS 集群中,Pod 和 Service 将获得 IPv6 地址,同时保持与传统 IPv4 端点的兼容性。这包括外部 IPv4 端点访问集群内部服务以及 Pod 访问外部 IPv4 端点的能力。

Amazon EKS IPv6 支持利用原生 VPC IPv6 功能。每个 VPC 都被分配一个 IPv4 地址前缀(CIDR 块大小可以从 /16 到 /28)和一个唯一的 /56 IPv6 地址前缀(固定)来自 Amazon 的 GUA(全局单播地址);您可以为 VPC 中的每个子网分配一个 /64 地址前缀。IPv4 功能,如路由表、网络访问控制列表、对等和 DNS 解析,在启用 IPv6 的 VPC 中的工作方式相同。这个 VPC 被称为双栈 VPC,遵循双栈子网,下图描述了支持 EKS/IPv6 集群的 IPV4 和 IPv6 VPC 基础设施模式:

![双栈 VPC,EKS 集群在 IPv6 模式下的必备基础](./eks-ipv6-foundation.png)

在 IPv6 世界中,每个地址都是可路由的互联网地址。默认情况下,VPC 从公共 GUA 范围分配 IPv6 CIDR。VPC 不支持从 [Unique Local Address (ULA)](https://en.wikipedia.org/wiki/Unique_local_address) 范围(fd00::/8 或 fc00::/8)分配私有 IPv6 地址,即使您想分配自己拥有的 IPv6 CIDR 也是如此。通过在 VPC 中实现出口互联网网关(EIGW),支持从私有子网到互联网的出口流量,同时阻止所有入站流量。
下图描述了 EKS/IPv6 集群中 Pod 的 IPv6 互联网出口流量:

![双栈 VPC,EKS 集群在 IPv6 模式下,Pod 在私有子网中出口到 IPv6 互联网端点](./eks-egress-ipv6.png)

有关实施 IPv6 子网的最佳实践,可以在 [VPC 用户指南](https://docs.aws.amazon.com/whitepapers/latest/ipv6-on-aws/IPv6-on-AWS.html)中找到。

在 IPv6 EKS 集群中,节点和 Pod 接收公共 IPv6 地址。EKS 根据唯一本地 IPv6 单播地址(ULA)为服务分配 IPv6 地址。IPv6 集群的 ULA 服务 CIDR 在集群创建阶段自动分配,不能指定,这与 IPv4 不同。下图描述了 EKS/IPv6 集群控制平面和数据平面的基础设施模式:

![双栈 VPC,EKS 集群在 IPv6 模式下,控制平面 ULA,数据平面 IPv6 GUA 用于 EC2 和 Pod](./eks-cluster-ipv6-foundation.png)

## 概述

EKS/IPv6 仅在前缀模式下受支持(VPC-CNI 插件 ENI IP 分配模式)。了解更多关于[前缀模式](https://aws.github.io/aws-eks-best-practices/networking/prefix-mode/index_linux/)的信息。
> 前缀分配仅适用于基于 Nitro 的 EC2 实例,因此 EKS/IPv6 仅在集群数据平面使用基于 EC2 Nitro 的实例时受支持。

简单地说,一个 /80 的 IPv6 前缀(每个工作节点)将产生约 10^14 个 IPv6 地址,限制因素将不再是 IP 而是 Pod 密度(资源方面)。

IPv6 前缀分配仅发生在 EKS 工作节点引导时。
这种行为被认为可以缓解 EKS/IPv4 集群中 Pod 高流失率导致的 Pod 调度延迟,这是由于 VPC CNI 插件(ipamd)为及时分配私有 IPv4 地址而生成的受限 API 调用。它还被认为可以使 VPC-CNI 插件高级调节参数[WARM_IP/ENI*、MINIMUM_IP*](https://github.com/aws/amazon-vpc-cni-k8s#warm_ip_target)变得不必要。

下图放大了 IPv6 工作节点的弹性网络接口(ENI):

![工作节点子网的插图,包括具有多个 IPv6 地址的主 ENI](./image-2.png)

每个 EKS 工作节点都被分配 IPv4 和 IPv6 地址,以及相应的 DNS 条目。对于给定的工作节点,只消耗双栈子网中的单个 IPv4 地址。EKS 对 IPv6 的支持使您能够通过一种高度固执己见的出口 IPv4 模型与 IPv4 端点(AWS、本地、互联网)进行通信。EKS 实现了一个主机本地 CNI 插件,作为 VPC CNI 插件的次要插件,为 Pod 分配和配置 IPv4 地址。CNI 插件从 169.254.172.0/22 范围为 Pod 配置一个特定于主机的非可路由 IPv4 地址。分配给 Pod 的 IPv4 地址*唯一于工作节点*,并且*不会在工作节点之外广播*。169.254.172.0/22 最多可提供 1024 个唯一的 IPv4 地址,可支持大型实例类型。

下图描述了 IPv6 Pod 连接到集群边界外的 IPv4 端点(非互联网)的流程:

![EKS/IPv6,IPv4 出口流量仅](./eks-ipv4-snat-cni.png)

在上图中,Pod 将对端点执行 DNS 查找,在收到 IPv4 "A" 响应后,Pod 的节点专有 IPv4 地址将通过源网络地址转换(SNAT)转换为附加到 EC2 工作节点的主网络接口的私有 IPv4(VPC)地址。

EKS/IPv6 Pod 还需要使用公共 IPv4 地址连接到互联网上的 IPv4 端点,为此存在类似的流程。
下图描述了 IPv6 Pod 连接到集群边界外(可路由互联网)的 IPv4 端点的流程:

![EKS/IPv6,IPv4 互联网出口流量仅](./eks-ipv4-snat-cni-internet.png)

在上图中,Pod 将对端点执行 DNS 查找,在收到 IPv4 "A" 响应后,Pod 的节点专有 IPv4 地址将通过源网络地址转换(SNAT)转换为附加到 EC2 工作节点的私有 IPv4(VPC)地址。然后,Pod IPv4 地址(源 IPv4:EC2 主 IP)被路由到 IPv4 NAT 网关,其中 EC2 主 IP 被转换(SNAT)为有效的可路由互联网 IPv4 公共 IP 地址(NAT 网关分配的公共 IP)。

任何跨节点的 Pod 到 Pod 通信都使用 IPv6 地址。VPC CNI 配置 iptables 来处理 IPv6,同时阻止任何 IPv4 连接。

Kubernetes 服务将仅从唯一的[本地 IPv6 单播地址(ULA)](https://datatracker.ietf.org/doc/html/rfc4193)获得 IPv6 地址(ClusterIP)。IPv6 集群的 ULA 服务 CIDR 在 EKS 集群创建阶段自动分配,不能修改。下图描述了 Pod 到 Kubernetes 服务的流程:

![EKS/IPv6,IPv6 Pod 到 IPv6 k8s 服务(ClusterIP ULA)流程](./Pod-to-service-ipv6.png)

服务通过 AWS 负载均衡器暴露到互联网。负载均衡器接收公共 IPv4 和 IPv6 地址,即双栈负载均衡器。对于访问 IPv6 集群 Kubernetes 服务的 IPv4 客户端,负载均衡器执行 IPv4 到 IPv6 的转换。

Amazon EKS 建议在私有子网中运行工作节点和 Pod。您可以在公共子网中创建公共负载均衡器,将流量负载均衡到位于私有子网中的节点上的 Pod。
下图描述了 IPv4 互联网用户访问 EKS/IPv6 Ingress 服务的情况:

![IPv4 互联网用户到 EKS/IPv6 Ingress 服务](./ipv4-internet-to-eks-ipv6.png)

> 注意:上述模式需要部署[最新版本](https://kubernetes-sigs.github.io/aws-load-balancer-controller)的 AWS 负载均衡器控制器

### EKS 控制平面 <-> 数据平面通信

EKS 将在双栈模式(IPv4/IPv6)下配置跨账户 ENI(X-ENI)。Kubernetes 节点组件(如 kubelet 和 kube-proxy)被配置为支持双栈。Kubelet 和 kube-proxy 在 hostNetwork 模式下运行,并绑定到节点主网络接口附加的 IPv4 和 IPv6 地址。Kubernetes api-server 通过 X-ENI 与 Pod 和节点组件进行通信,这种通信始终使用 IPv6 模式。Pod 通过 X-ENI 与 api-server 通信,Pod 到 api-server 的通信始终使用 IPv6 模式。

![集群包括 X-ENI 的插图](./image-5.png)

## 建议

### 保持对 IPv4 EKS API 的访问

EKS API 仅通过 IPv4 访问。这也包括集群 API 端点。您将无法从仅 IPv6 的网络访问集群端点和 API。您的网络需要(1)支持 NAT64/DNS64 等 IPv6 过渡机制,以便 IPv6 和 IPv4 主机之间进行通信,以及(2)支持 IPv4 端点的 DNS 服务转换。

### 根据计算资源进行调度

单个 IPv6 前缀足以在单个节点上运行许多 Pod。这也有效地消除了 ENI 和 IP 对节点上最大 Pod 数量的限制。尽管 IPv6 消除了对最大 Pod 的直接依赖,但在使用较小实例类型(如 m5.large)的前缀附加时,您很可能会在耗尽 IP 地址之前耗尽实例的 CPU 和内存资源。如果您使用自管理节点组或使用自定义 AMI ID 的托管节点组,您必须手动设置 EKS 推荐的最大 Pod 值。

您可以使用以下公式来确定在 IPv6 EKS 集群上可以部署在节点上的最大 Pod 数量。

* ((实例类型的网络接口数量(每个网络接口的前缀数量-1)* 16) + 2

* ((3 个 ENI)*((每个 ENI 的 10 个辅助 IP-1)* 16)) + 2 = 460 (实际)

托管节点组会自动为您计算最大 Pod 数。避免更改 EKS 推荐的最大 Pod 数值,以避免由于资源限制导致的 Pod 调度失败。

### 评估现有自定义网络的目的

如果当前启用了[自定义网络](https://aws.github.io/aws-eks-best-practices/networking/custom-networking/),Amazon EKS 建议您重新评估在 IPv6 下是否仍需要它。如果您选择使用自定义网络来解决 IPv4 耗尽问题,那么在 IPv6 下就不再需要了。如果您正在使用自定义网络来满足安全要求,例如为节点和 Pod 设置单独的网络,您可以提交一个[EKS 路线图请求](https://github.com/aws/containers-roadmap/issues)。

### EKS/IPv6 集群中的 Fargate Pod

EKS 支持在 Fargate 上运行的 Pod 使用 IPv6。在 Fargate 上运行的 Pod 将使用从 VPC CIDR 范围(IPv4 和 IPv6)中分配的 IPv6 和 VPC 可路由私有 IPv4 地址。简单地说,您的 EKS/Fargate Pod 集群范围密度将受到可用 IPv4 和 IPv6 地址的限制。如果底层子网没有可用的 IPv4 地址,无论 IPv6 地址是否可用,您都无法调度新的 Fargate Pod。

### 部署 AWS 负载均衡器控制器(LBC)

**上游内置 Kubernetes 服务控制器不支持 IPv6**。我们建议使用[最新版本](https://kubernetes-sigs.github.io/aws-load-balancer-controller)的 AWS 负载均衡器控制器插件。LBC 将仅在使用以下注释的 Kubernetes 服务/Ingress 定义时部署双栈 NLB 或双栈 ALB:"alb.ingress.kubernetes.io/ip-address-type: dualstack"和"alb.ingress.kubernetes.io/target-type: ip"

AWS 网络负载均衡器不支持双栈 UDP 协议地址类型。如果您对低延迟、实时流媒体、在线游戏和物联网有强烈需求,我们建议运行 IPv4 集群。要了解如何管理 UDP 服务的运行状况检查,请参考["如何将 UDP 流量路由到 Kubernetes"](https://aws.amazon.com/blogs/containers/how-to-route-udp-traffic-into-kubernetes/)。