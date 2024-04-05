
# 优化 IP 地址利用率

由于应用程序现代化,容器化环境正以快速的速度不断扩大规模。这意味着越来越多的工作节点和 pod 正在被部署。

[Amazon VPC CNI](../vpc-cni/) 插件从 VPC 的 CIDR(s) 中为每个 pod 分配一个 IP 地址。这种方法可以使用诸如 VPC Flow Logs 和其他监控解决方案等工具完全了解 Pod 地址。根据您的工作负载类型,这可能会导致大量 IP 地址被 pod 消耗。

在设计 AWS 网络架构时,优化 Amazon EKS IP 消耗对于 VPC 和节点级别都很重要。这将有助于您缓解 IP 耗尽问题,并提高每个节点的 pod 密度。

在本节中,我们将讨论可以帮助您实现这些目标的技术。

## 优化节点级 IP 消耗

[前缀委派](https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html)是 Amazon Virtual Private Cloud (Amazon VPC) 的一项功能,允许您将 IPv4 或 IPv6 前缀分配给您的 Amazon Elastic Compute Cloud (Amazon EC2) 实例。它增加了每个网络接口 (ENI) 的 IP 地址,从而提高了每个节点的 pod 密度并提高了计算效率。前缀委派也支持自定义网络。

有关详细信息,请参见[使用 Linux 节点的前缀委派](../prefix-mode/index_linux/)和[使用 Windows 节点的前缀委派](../prefix-mode/index_windows/)部分。

## 缓解 IP 耗尽

为了防止您的集群消耗所有可用的 IP 地址,我们强烈建议您在考虑增长的情况下设置 VPC 和子网的大小。

采用 [IPv6](../ipv6/) 是避免这些问题的好方法。但是,对于那些可扩展性需求超出初始规划且无法采用 IPv6 的组织来说,改善 VPC 设计是应对 IP 地址耗尽的推荐方法。Amazon EKS 客户中最常用的技术是向 VPC 添加非路由的辅助 CIDR,并配置 VPC CNI 在为 Pod 分配 IP 地址时使用这些额外的 IP 空间。这通常被称为[自定义网络](../custom-networking/)。

我们将介绍您可以用来优化分配给节点的 IP 地址热池的 Amazon VPC CNI 变量。我们将以一些不属于 Amazon EKS 本身但可以帮助缓解 IP 耗尽的其他架构模式来结束本节。

### 使用 IPv6 (推荐)

采用 IPv6 是解决 RFC1918 限制的最简单方法;我们强烈建议您将采用 IPv6 作为选择网络架构的首选选项。IPv6 提供了显著更大的总 IP 地址空间,集群管理员可以专注于迁移和扩展应用程序,而无需投入精力来解决 IPv4 的限制。

亚马逊 EKS 集群同时支持 IPv4 和 IPv6。默认情况下,EKS 集群使用 IPv4 地址空间。在集群创建时指定 IPv6 地址空间将启用 IPv6 的使用。在 IPv6 EKS 集群中,pod 和服务将获得 IPv6 地址,同时**保持了旧版 IPv4 端点连接到运行在 IPv6 集群上的服务以及反之亦然的能力**。集群内部的所有 pod 到 pod 通信都将通过 IPv6 进行。在 VPC (/56) 内,IPv6 子网的 CIDR 块大小固定为 /64。这提供了约 18 quintillion 个 IPv6 地址,允许您在 EKS 上扩展部署。

有关详细信息,请参见[在 Amazon EKS 集群上运行 IPv6](../ipv6/)部分,如需实践经验,请参见[了解 Amazon EKS 上的 IPv6](https://catalog.workshops.aws/ipv6-on-aws/en-US/lab-6)部分,这是[使用 IPv6 的动手实践研讨会](https://catalog.workshops.aws/ipv6-on-aws/en-US)的一部分。

![EKS 集群在 IPv6 模式下的流量](./ipv6.gif)

### 优化 IPv4 集群中的 IP 消耗

本节专门针对正在运行遗留应用程序和/或尚未准备好迁移到 IPv6 的客户。虽然我们鼓励所有组织尽快迁移到 IPv6,但我们也认识到一些组织可能仍需要寻找其他方法来扩展其容器工作负载的 IPv4 能力。因此,我们还将为您介绍使用 Amazon EKS 集群优化 IPv4 (RFC1918) 地址空间消耗的架构模式。

#### 规划增长

作为防止 IP 耗尽的第一道防线,我们强烈建议您在设计 IPv4 VPC 和子网时考虑增长因素,以防止集群耗尽所有可用的 IP 地址。如果子网没有足够的可用 IP 地址,您将无法创建新的 Pod 或节点。

在构建 VPC 和子网之前,建议从所需的工作负载规模开始倒推。例如,当使用 [eksctl](https://eksctl.io/)(一个用于在 EKS 上创建和管理集群的简单 CLI 工具)构建集群时,默认情况下会创建 /19 子网。/19 掩码对于大多数工作负载类型来说都是合适的,可以分配超过 8000 个地址。

!!! attention
    在设置 VPC 和子网大小时,除了 pod 和节点之外,还可能有其他一些元素会消耗 IP 地址,例如负载均衡器、RDS 数据库和其他 VPC 内部服务。
此外,Amazon EKS 可以创建多达 4 个弹性网络接口(X-ENI),这些接口是为了允许与控制平面进行通信(更多信息[在此](../subnets/))。在集群升级期间,Amazon EKS 会创建新的 X-ENI,并在升级成功时删除旧的 X-ENI。因此,我们建议为与 EKS 集群关联的子网使用至少 /28(16 个 IP 地址)的掩码。
您可以使用[示例 EKS 子网计算器](../subnet-calc/subnet-calc.xlsx)电子表格来规划您的网络。该电子表格根据工作负载和 VPC ENI 配置计算 IP 使用情况。将 IP 使用情况与 IPv4 子网进行比较,以确定配置和子网大小是否足以满足您的工作负载。请注意,如果 VPC 中的子网耗尽可用 IP 地址,我们建议[使用 VPC 的原始 CIDR 块创建新的子网](https://docs.aws.amazon.com/vpc/latest/userguide/working-with-subnets.html#create-subnets)。请注意,现在[Amazon EKS 允许修改集群子网和安全组](https://aws.amazon.com/about-aws/whats-new/2023/10/amazon-eks-modification-cluster-subnets-security/)。

#### 扩展 IP 空间

如果即将耗尽 RFC1918 IP 空间,您可以使用[自定义网络](../custom-networking/)模式来保护可路由 IP,方法是在专用的附加子网中调度 Pod。
虽然自定义网络将接受 VPC 范围内的有效次要 CIDR 范围,但我们建议您使用 CG-NAT 空间的 CIDR (/16),即 `100.64.0.0/10` 或 `198.19.0.0/16`,因为这些范围在企业环境中使用的可能性较小。

有关详细信息,请参见[自定义网络](../custom-networking/)的专用部分。

![自定义网络,流量流](./custom-networking.gif)

#### 优化 IP 预热池

使用默认配置,VPC CNI 会保留整个 ENI(及其关联的 IP)在预热池中。这可能会消耗大量 IP,特别是在较大的实例类型上。

如果您的集群子网可用 IP 地址有限,请仔细检查这些 VPC CNI 配置环境变量:

* `WARM_IP_TARGET` 
* `MINIMUM_IP_TARGET`
* `WARM_ENI_TARGET`

您可以将 `MINIMUM_IP_TARGET` 的值配置为与您预期在节点上运行的 Pod 数量相匹配。这样做可以确保在创建 Pod 时,CNI 可以从预热池分配 IP 地址,而无需调用 EC2 API。

请注意,将 `WARM_IP_TARGET` 的值设置过低会导致对 EC2 API 的额外调用,这可能会导致请求被限制。对于大型集群,请与 `MINIMUM_IP_TARGET` 一起使用,以避免请求被限制。

要配置这些选项,您可以下载 `aws-k8s-cni.yaml` 清单并设置环境变量。在撰写本文时,最新版本位于[此处](https://github.com/aws/amazon-vpc-cni-k8s/blob/master/config/master/aws-k8s-cni.yaml)。检查配置值的版本是否与安装的 VPC CNI 版本匹配。

!!! 警告
    在更新 CNI 时,这些设置将被重置为默认值。请在更新之前备份 CNI,并在更新成功后检查配置设置是否需要重新应用。

您可以在不停机的情况下实时调整 CNI 参数以满足现有应用程序的需求,但您应该选择能够支持可扩展性需求的值。例如,如果您正在处理批处理工作负载,我们建议将默认的 `WARM_ENI_TARGET` 更新为匹配 Pod 扩展需求。将 `WARM_ENI_TARGET` 设置为高值可始终维护运行大型批处理工作负载所需的温暖 IP 池,从而避免数据处理延迟。

!!! warning
    改善您的 VPC 设计是应对 IP 地址耗尽的推荐解决方案。考虑 IPv6 和辅助 CIDR 等解决方案。调整这些值以最小化温暖 IP 的数量应该是在排除其他选项后的临时解决方案。错误配置这些值可能会干扰集群操作。

    **在对生产系统进行任何更改之前**,请务必查看[此页面](https://github.com/aws/amazon-vpc-cni-k8s/blob/master/docs/eni-and-ip-target.md)上的注意事项。

#### 监控 IP 地址库存

除了上述解决方案,可见性 IP 利用率也很重要。您可以使用 [CNI 指标助手](https://docs.aws.amazon.com/eks/latest/userguide/cni-metrics-helper.html)监控子网的 IP 地址库存。可用的一些指标包括:

* 集群可支持的最大 ENI 数量
* 已分配的 ENI 数量
* 当前分配给 Pod 的 IP 地址数量
* 可用 IP 地址的总数和最大数量

您还可以设置 [CloudWatch 警报](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html)以在子网即将耗尽 IP 地址时获得通知。请访问 EKS 用户指南以了解 [CNI 指标助手](https://docs.aws.amazon.com/eks/latest/userguide/cni-metrics-helper.html)的安装说明。

!!! warning
    确保 VPC CNI 的 `DISABLE_METRICS` 变量设置为 false。

#### 进一步考虑

还有一些不属于亚马逊 EKS 固有的架构模式可以帮助解决 IP 耗尽问题。例如,您可以[优化 VPC 之间的通信](../subnets/#communication-across-vpcs)或[跨多个账户共享 VPC](../subnets/#sharing-vpc-across-multiple-accounts)来限制 IPv4 地址分配。

在此处了解更多关于这些模式的信息:

* [设计超大规模亚马逊 VPC 网络](https://aws.amazon.com/blogs/networking-and-content-delivery/designing-hyperscale-amazon-vpc-networks/)
* [使用亚马逊 VPC Lattice 构建安全的多账户多 VPC 连接](https://aws.amazon.com/blogs/networking-and-content-delivery/build-secure-multi-account-multi-vpc-connectivity-for-your-applications-with-amazon-vpc-lattice/)。
