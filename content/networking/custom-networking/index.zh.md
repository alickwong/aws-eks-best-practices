!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

# 自定义网络

默认情况下, Amazon VPC CNI 将为 Pods 分配从主子网中选择的 IP 地址。主子网是主 ENI 附加到的子网 CIDR,通常是节点/主机的子网。

如果子网 CIDR 太小,CNI 可能无法获取足够的辅助 IP 地址来分配给您的 Pods。这是 EKS IPv4 集群的一个常见挑战。

自定义网络是解决这个问题的一种解决方案。

自定义网络通过从辅助 VPC 地址空间(CIDR)分配节点和 Pod IP 来解决 IP 耗尽问题。自定义网络支持 ENIConfig 自定义资源。ENIConfig 包括备用子网 CIDR 范围(从辅助 VPC CIDR 中划分),以及 Pods 将属于的安全组。启用自定义网络时,VPC CNI 在 ENIConfig 中定义的子网中创建辅助 ENI。CNI 从 ENIConfig CRD 中定义的 CIDR 范围内分配 Pods 的 IP 地址。

由于主 ENI 不被自定义网络使用,节点上可运行的 Pods 的最大数量较低。主机网络 Pods 继续使用分配给主 ENI 的 IP 地址。此外,主 ENI 用于处理源网络转换并路由 Pods 流量到节点外部。

## 示例配置

虽然自定义网络将接受 VPC 范围内的有效次要 CIDR 范围,但我们建议您使用来自 CG-NAT 空间的 CIDR(/16),即 100.64.0.0/10 或 198.19.0.0/16,因为这些范围在企业环境中使用的可能性较低。有关您可以与 VPC 一起使用的允许和受限 CIDR 块关联的更多信息,请参见 VPC 文档中的 [IPv4 CIDR 块关联限制](https://docs.aws.amazon.com/vpc/latest/userguide/configure-your-vpc.html#add-cidr-block-restrictions)部分。

如下图所示,工作节点的主弹性网络接口(ENI)仍使用主 VPC CIDR 范围(在本例中为 10.0.0.0/16),但辅助 ENI 使用辅助 VPC CIDR 范围(在本例中为 100.64.0.0/16)。现在,为了让 Pods 使用 100.64.0.0/16 CIDR 范围,您必须配置 CNI 插件以使用自定义网络。您可以按照[此处](https://docs.aws.amazon.com/eks/latest/userguide/cni-custom-network.html)记录的步骤进行操作。

![pods on secondary subnet 的插图](./image.png)

如果您希望 CNI 使用自定义网络,请将 `AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG` 环境变量设置为 `true`。
```
kubectl set env daemonset aws-node -n kube-system AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true
```

当 `AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true` 时,CNI 将从 `ENIConfig` 中定义的子网中分配 Pod IP 地址。`ENIConfig` 自定义资源用于定义 Pod 将被调度的子网。
```
apiVersion : crd.k8s.amazonaws.com/v1alpha1
kind : ENIConfig
metadata:
  name: us-west-2a
spec: 
  securityGroups:
    - sg-0dff111a1d11c1c11
  subnet: subnet-011b111c1f11fdf11
```


在创建 `ENIconfig` 自定义资源后，您需要创建新的工作节点并排空现有节点。现有的工作节点和 Pod 将保持不受影响。

## 建议

### 何时使用自定义网络

如果您正在处理 IPv4 耗尽的问题且无法使用 IPv6，我们建议您考虑使用自定义网络。Amazon EKS 对 [RFC6598](https://datatracker.ietf.org/doc/html/rfc6598) 空间的支持使您能够扩展 Pod 超出 [RFC1918](https://datatracker.ietf.org/doc/html/rfc1918) 地址耗尽的挑战。请考虑使用前缀委派与自定义网络以提高节点上的 Pod 密度。

如果您有一个安全要求,需要在具有不同安全组要求的不同网络上运行 Pod,那么您可能会考虑使用自定义网络。启用自定义网络后,Pod 将使用在 ENIConfig 中定义的不同子网或安全组,而不是节点的主网络接口。

自定义网络确实是部署多个 EKS 集群并将其连接到本地数据中心服务的理想选择。您可以增加 VPC 中可供 EKS 使用的私有地址(RFC1918)数量,用于诸如 Amazon Elastic Load Balancing 和 NAT-GW 等服务,同时对跨多个集群的 Pod 使用不可路由的 CG-NAT 空间。使用[过境网关](https://aws.amazon.com/transit-gateway/)和共享服务 VPC(包括跨多个可用区的 NAT 网关,以实现高可用性)的自定义网络可以实现可扩展且可预测的流量流。[此博客文章](https://aws.amazon.com/blogs/containers/eks-vpc-routable-ip-address-conservation/)描述了一种架构模式,这是使用自定义网络将 EKS Pod 连接到数据中心网络的最推荐方式之一。

### 何时避免使用自定义网络

#### 准备实施 IPv6

自定义网络可以缓解 IP 耗尽问题,但需要额外的运营开销。如果您当前正在部署双栈(IPv4/IPv6) VPC,或者您的计划包括 IPv6 支持,我们建议您实施 IPv6 集群。您可以设置 IPv6 EKS 集群并迁移您的应用程序。在 IPv6 EKS 集群中,Kubernetes 和 Pod 都会获得 IPv6 地址,并能够与 IPv4 和 IPv6 端点进行通信。请查看[运行 IPv6 EKS 集群](../ipv6/index.md)的最佳实践。

#### 耗尽 CG-NAT 空间

此外,如果您当前正在使用 CG-NAT 空间的 CIDR,或者无法将辅助 CIDR 链接到您的集群 VPC,您可能需要探索其他选择,如使用替代的 CNI。我们强烈建议您获得商业支持,或拥有调试和向开源 CNI 插件项目提交补丁的内部知识。请参阅[替代 CNI 插件](https://docs.aws.amazon.com/eks/latest/userguide/alternate-cni-plugins.html)用户指南以了解更多详细信息。

#### 使用私有 NAT 网关

亚马逊 VPC 现在提供[私有 NAT 网关](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)功能。亚马逊的私有 NAT 网关使私有子网中的实例能够连接到其他 VPC 和本地网络,即使存在重叠的 CIDR 块。考虑使用[博客文章](https://aws.amazon.com/blogs/containers/addressing-ipv4-address-exhaustion-in-amazon-eks-clusters-using-private-nat-gateways/)中描述的方法来部署私有 NAT 网关,以解决由于重叠 CIDR 块而导致的 EKS 工作负载通信问题,这是我们客户提出的一个重要问题。自定义网络无法单独解决重叠 CIDR 的困难,并且会增加配置挑战。

本博客文章中使用的网络架构遵循了 Amazon VPC 文档中[启用重叠网络之间的通信](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-scenarios.html#private-nat-overlapping-networks)的建议。如本博客文章所示,您可以结合使用私有 NAT 网关和 RFC6598 地址来管理客户的私有 IP 耗尽问题。EKS 集群和工作节点部署在不可路由的 100.64.0.0/16 VPC 辅助 CIDR 范围内,而私有 NAT 网关和 NAT 网关部署在可路由的 RFC1918 CIDR 范围内。博客解释了如何使用过渡网关连接 VPC,以便在具有重叠不可路由 CIDR 范围的 VPC 之间进行通信。对于 EKS 资源在 VPC 的不可路由地址范围内需要与其他 VPC 通信的用例,客户可以选择使用 VPC 对等互联来互连这些 VPC。这种方法可能会带来潜在的成本节省,因为所有数据传输都是在可用区内通过 VPC 对等连接进行的,现在是免费的。

![使用私有 NAT 网关的网络流量示意图](./image-3.png)

#### 节点和 Pod 的独特网络

如果您需要出于安全原因将节点和 Pod 隔离到特定的网络,我们建议您将节点和 Pod 部署到更大的辅助 CIDR 块(例如 100.64.0.0/8)中的子网。在您的 VPC 中安装新的 CIDR 后,您可以使用辅助 CIDR 部署另一个节点组,并排空原始节点,以自动将 pod 重新部署到新的工作节点。有关如何实施此操作的更多信息,请参见[博客文章](https://aws.amazon.com/blogs/containers/optimize-ip-addresses-usage-by-pods-in-your-amazon-eks-cluster/)。
自定义网络在下图所示的设置中并未使用。相反，Kubernetes 工作节点部署在您的 VPC 的次要 VPC CIDR 范围内的子网上,例如 100.64.0.0/10。您可以保持 EKS 集群运行(控制平面将保留在原始子网上),但节点和 Pod 将被移动到次要子网。这是另一种,尽管不太常见的技术,用于缓解 VPC 中 IP 耗尽的危险。我们建议在将 Pod 重新部署到新的工作节点之前,先排空旧节点。

![工作节点部署在次要子网上的示意图](./image-2.png)

### 使用可用区标签自动化配置

您可以使用 Kubernetes 自动应用工作节点可用区 (AZ) 的相应 ENIConfig。

Kubernetes 会自动为您的工作节点添加标签 [`topology.kubernetes.io/zone`](http://topology.kubernetes.io/zone)。当您每个可用区只有一个次要子网(备用 CIDR)时,Amazon EKS 建议使用可用区作为您的 ENI 配置名称。请注意,标签 `failure-domain.beta.kubernetes.io/zone` 已被弃用,并被标签 `topology.kubernetes.io/zone` 取代。

1. 将 `name` 字段设置为您的 VPC 的可用区。
2. 使用以下命令启用自动配置:
```
kubectl set env daemonset aws-node -n kube-system AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true
```


如果您每个可用区都有多个辅助子网,则需要创建一个特定的 `ENI_CONFIG_LABEL_DEF`。您可以考虑将 `ENI_CONFIG_LABEL_DEF` 配置为 [`k8s.amazonaws.com/eniConfig`](http://k8s.amazonaws.com/eniConfig),并使用自定义 eniConfig 名称标记节点,例如 [`k8s.amazonaws.com/eniConfig=us-west-2a-subnet-1`](http://k8s.amazonaws.com/eniConfig=us-west-2a-subnet-1) 和 [`k8s.amazonaws.com/eniConfig=us-west-2a-subnet-2`](http://k8s.amazonaws.com/eniConfig=us-west-2a-subnet-2)。

### 在配置辅助网络时替换 Pods

启用自定义网络不会修改现有节点。自定义网络是一种破坏性操作。与其在启用自定义网络后对集群中的所有工作节点进行滚动替换,我们建议更新 [EKS 入门指南](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html) 中的 AWS CloudFormation 模板,添加一个自定义资源,调用 Lambda 函数来更新 `aws-node` Daemonset,以在配置工作节点之前启用自定义网络。

如果在切换到自定义 CNI 网络功能之前,您的集群中有任何节点上运行着 Pods,您应该隔离并[排空这些节点](https://aws.amazon.com/premiumsupport/knowledge-center/eks-worker-node-actions/)以优雅地关闭 Pods,然后终止这些节点。只有与 ENIConfig 标签或注释匹配的新节点才使用自定义网络,因此在这些新节点上调度的 Pods 可以从辅助 CIDR 中分配 IP。

### 计算每个节点的最大 Pods 数

由于节点的主 ENI 不再用于分配 Pod IP 地址,因此每个 EC2 实例类型可以运行的 Pods 数量会减少。为了解决这一限制,您可以使用自定义网络的前缀分配。使用前缀分配,每个辅助 IP 都会在辅助 ENI 上替换为 /28 前缀。

考虑使用自定义网络的 m5.large 实例的最大 Pods 数。

不使用前缀分配的最大 Pods 数为 29

* ((3 个 ENI - 1) * (10 个辅助 IP 每个 ENI - 1)) + 2 = 20

启用前缀附加可将 Pods 数增加到 290。

* (((3 个 ENI - 1) * ((10 个辅助 IP 每个 ENI - 1) * 16)) + 2 = 290

但是,我们建议将 max-pods 设置为 110 而不是 290,因为该实例的虚拟 CPU 数量相当小。对于更大的实例,EKS 建议 max pods 值为 250。当在较小的实例类型(如 m5.large)上使用前缀附加时,您可能会在耗尽实例的 CPU 和内存资源之前就耗尽 IP 地址。

!!! info
当 CNI 前缀将 /28 前缀分配给 ENI 时,它必须是一个连续的 IP 地址块。如果生成前缀的子网高度碎片化,前缀附加可能会失败。您可以通过为集群创建一个新的专用 VPC 或者为前缀附加预留一组 CIDR 子网来缓解这种情况发生。有关此主题的更多信息,请访问[子网 CIDR 预留](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-cidr-reservation.html)。

### 识别 CG-NAT 空间的现有使用情况

自定义网络可以缓解 IP 耗尽问题,但无法解决所有挑战。如果您已经在集群中使用 CG-NAT 空间,或者无法将辅助 CIDR 与集群 VPC 关联,我们建议您探索其他选择,如使用备用 CNI 或转移到 IPv6 集群。
