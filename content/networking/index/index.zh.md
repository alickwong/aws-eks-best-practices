
# Amazon EKS 网络最佳实践指南

要有效运营集群和应用程序,了解 Kubernetes 网络是至关重要的。Pod 网络,也称为集群网络,是 Kubernetes 网络的核心。Kubernetes 支持使用[容器网络接口](https://github.com/containernetworking/cni)(CNI)插件进行集群网络。

Amazon EKS 正式支持[Amazon Virtual Private Cloud (VPC)](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html) CNI 插件来实现 Kubernetes Pod 网络。VPC CNI 与 AWS VPC 实现了原生集成,并在底层模式下工作。在底层模式下,Pod 和主机位于同一网络层,共享网络命名空间。从集群和 VPC 的角度来看,Pod 的 IP 地址是一致的。

本指南介绍了在 Kubernetes 集群网络中的[Amazon VPC 容器网络接口](https://github.com/aws/amazon-vpc-cni-k8s)[(VPC CNI)](https://github.com/aws/amazon-vpc-cni-k8s)。VPC CNI 是 EKS 支持的默认网络插件,因此是本指南的重点。VPC CNI 可高度配置,以支持不同的用例。本指南还包括了关于不同 VPC CNI 用例、操作模式和子组件的专门章节,以及相关建议。

Amazon EKS 运行上游 Kubernetes,并通过认证达到 Kubernetes 一致性。尽管您可以使用其他 CNI 插件,但本指南不提供管理备用 CNI 的建议。请查看 [EKS 备用 CNI](https://docs.aws.amazon.com/eks/latest/userguide/alternate-cni-plugins.html)文档,了解管理备用 CNI 的合作伙伴和资源。

## Kubernetes 网络模型

Kubernetes 对集群网络提出了以下要求:

* 同一节点上调度的 Pod 必须能够相互通信,而无需使用 NAT(网络地址转换)。
* 在特定节点上运行的所有系统守护进程(后台进程,例如 [kubelet](https://kubernetes.io/docs/concepts/overview/components/))都能与同一节点上运行的 Pod 通信。
* 使用[主机网络](https://docs.docker.com/network/host/)的 Pod 必须能够与所有其他节点上的所有其他 Pod 联系,而无需使用 NAT。

有关 Kubernetes 期望兼容网络实现的详细信息,请参见[Kubernetes 网络模型](https://kubernetes.io/docs/concepts/services-networking/#the-kubernetes-network-model)。下图说明了 Pod 网络命名空间和主机网络命名空间之间的关系。

![主机网络和 2 个 Pod 网络命名空间的示意图](image.png)

## 容器网络接口 (CNI)

简化中文版本:

Kubernetes 支持 CNI 规范和插件来实现 Kubernetes 网络模型。CNI 包括一个[规范](https://github.com/containernetworking/cni/blob/main/SPEC.md)(当前版本 1.0.0)和用于编写配置容器网络接口的插件的库,以及许多支持的插件。CNI 仅关注容器的网络连接性,以及在删除容器时删除分配的资源。

通过传递 `--network-plugin=cni` 命令行选项来启用 CNI 插件。Kubelet 从 `--cni-conf-dir`(默认为 /etc/cni/net.d)读取文件,并使用该文件中的 CNI 配置来设置每个 Pod 的网络。CNI 配置文件必须与 CNI 规范(最低版本 v0.4.0)匹配,并且配置中引用的任何所需的 CNI 插件都必须存在于 `--cni-bin-dir`目录(默认为 /opt/cni/bin)中。如果目录中有多个 CNI 配置文件,*kubelet 将使用按名称以字典顺序排列的第一个配置文件*。

## Amazon Virtual Private Cloud (VPC) CNI

AWS 提供的 VPC CNI 是 EKS 集群的默认网络插件。VPC CNI 插件在您配置 EKS 集群时默认安装。VPC CNI 在 Kubernetes 工作节点上运行。VPC CNI 插件包括 CNI 二进制文件和 IP 地址管理(ipamd)插件。CNI 从 VPC 网络为 Pod 分配 IP 地址。ipamd 管理分配给每个 Kubernetes 节点的 AWS 弹性网络接口(ENI),并维护 IP 地址的预热池。VPC CNI 提供了预分配 ENI 和 IP 地址的配置选项,以实现快速 Pod 启动时间。请参考[Amazon VPC CNI](../vpc-cni/index.md)了解推荐的插件管理最佳实践。

Amazon EKS 建议您在创建集群时指定至少两个可用区的子网。Amazon VPC CNI 从节点子网中分配 IP 地址给 Pod。我们强烈建议检查子网中是否有足够的可用 IP 地址。部署 EKS 集群之前,请考虑[VPC 和子网](../subnets/index.md)的相关建议。

Amazon VPC CNI 从附加到节点主 ENI 的子网中分配一个预热池的 ENI 和辅助 IP 地址。这种 VPC CNI 模式称为"[辅助 IP 模式](../vpc-cni/index.md)"。IP 地址数量和 Pod 密度由 ENI 数量以及实例类型定义的每个 ENI 的 IP 地址限制决定。辅助模式是默认模式,适用于使用较小实例类型的小型集群。如果您遇到 Pod 密度问题,请考虑使用[前缀模式](../prefix-mode/index_linux.md)。您还可以通过为 ENI 分配前缀来增加节点上 Pod 可用的 IP 地址。

亚马逊 VPC CNI 与 AWS VPC 本地集成,允许用户应用现有的 AWS VPC 网络和安全最佳实践来构建 Kubernetes 集群。这包括使用 VPC 流日志、VPC 路由策略和安全组进行网络流量隔离的能力。默认情况下,Amazon VPC CNI 将与主 ENI 关联的安全组应用于 Pod。当您希望为 Pod 分配不同的网络规则时,请考虑启用 [Pod 安全组](../sgpp/index.md)。

默认情况下,VPC CNI 从分配给节点主 ENI 的子网中为 Pod 分配 IP 地址。在运行具有数千个工作负载的大型集群时,经常会遇到 IPv4 地址短缺的问题。AWS VPC 允许您通过[分配辅助 CIDR](https://docs.aws.amazon.com/vpc/latest/userguide/configure-your-vpc.html#add-cidr-block-restrictions)来扩展可用的 IP。AWS VPC CNI 允许您为 Pod 使用不同的子网 CIDR 范围。VPC CNI 的这个特性称为[自定义网络](../custom-networking/index.md)。您可以考虑使用自定义网络来使用 100.64.0.0/10 和 198.19.0.0/16 CIDR(CG-NAT)与 EKS。这实际上允许您创建一个环境,其中 Pod 不再消耗您的 VPC 中的任何 RFC1918 IP 地址。

自定义网络是解决 IPv4 地址耗尽问题的一个选项,但它需要运营开销。我们建议使用 IPv6 集群而不是自定义网络来解决这个问题。具体来说,如果您已经完全耗尽了 VPC 的所有可用 IPv4 地址空间,我们建议您迁移到 [IPv6 集群](../ipv6/index.md)。评估您组织支持 IPv6 的计划,并考虑投资 IPv6 是否具有更长远的价值。

EKS 对 IPv6 的支持主要是为了解决由于 IPv4 地址空间有限而导致的 IP 耗尽问题。为了解决客户遇到的 IPv4 耗尽问题,EKS 优先考虑 IPv6 独立 Pod 而不是双栈 Pod。也就是说,Pod 可能能够访问 IPv4 资源,但它们不会从 VPC CIDR 范围内分配 IPv4 地址。VPC CNI 会从 AWS 管理的 VPC IPv6 CIDR 块中为 Pod 分配 IPv6 地址。

## 子网计算器

该项目包括一个[子网计算器 Excel 文档](../subnet-calc/subnet-calc.xlsx)。该计算器文档模拟了在不同 ENI 配置选项(如 `WARM_IP_TARGET` 和 `WARM_ENI_TARGET`)下指定工作负载的 IP 地址消耗情况。该文档包含两个工作表,一个用于 Warm ENI 模式,另一个用于 Warm IP 模式。请查看[VPC CNI 指南](../vpc-cni/index.md)以了解更多信息。

输入:
- 子网 CIDR 大小
- Warm ENI 目标 *或* Warm IP 目标
- 实例列表
    - 类型、数量和每个实例调度的工作负载 Pod 数量

输出:
- 托管的总 Pod 数
- 消耗的子网 IP 数
- 剩余的子网 IP 数
- 实例级别详细信息
    - 每个实例的预热 IP/ENI 数量
    - 每个实例的活跃 IP/ENI 数量
