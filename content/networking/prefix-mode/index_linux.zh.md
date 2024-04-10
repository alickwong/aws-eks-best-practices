# Linux 的前缀模式

Amazon VPC CNI 将网络前缀分配给 [Amazon EC2 网络接口](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-prefix-eni.html)，以增加节点可用的 IP 地址数量并提高每个节点的 Pod 密度。您可以配置 Amazon VPC CNI 加载项的 1.9.0 或更高版本，将 IPv4 和 IPv6 CIDR 分配给网络接口，而不是分配单个辅助 IP 地址。

前缀模式默认在 IPv6 集群上启用，这是唯一支持的选项。VPC CNI 将 /80 IPv6 前缀分配给 ENI 上的一个插槽。有关更多信息，请参阅本指南的 [IPv6 部分](../ipv6/index.md)。

使用前缀分配模式，每个实例类型的弹性网络接口的最大数量保持不变,但是您现在可以配置 Amazon VPC CNI 分配 /28 (16 个 IP 地址) IPv4 地址前缀,而不是将单个 IPv4 地址分配给网络接口上的插槽。当 `ENABLE_PREFIX_DELEGATION` 设置为 true 时,VPC CNI 会从分配给 ENI 的前缀中分配 IP 地址给 Pod。请按照 [EKS 用户指南](https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html)中提到的说明启用前缀 IP 模式。

![两个工作节点子网的插图,比较 ENI 辅助 IPvs 和具有委托前缀的 ENI](./image.png)

可以分配给网络接口的 IP 地址的最大数量取决于实例类型。分配给网络接口的每个前缀都算作一个 IP 地址。例如,`c5.large` 实例的网络接口限制为 `10` 个 IPv4 地址。每个网络接口都有一个主 IPv4 地址。如果网络接口没有辅助 IPv4 地址,您可以为该网络接口分配最多 9 个前缀。对于分配给网络接口的每个额外 IPv4 地址,您可以分配一个较少的前缀。请查看 AWS EC2 文档,了解[每个实例类型的网络接口 IP 地址数](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI)和[为网络接口分配前缀](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-prefix-eni.html)。

在工作节点初始化期间,VPC CNI 会将一个或多个前缀分配给主 ENI。CNI 通过维护一个预热池来预先分配前缀,以加快 Pod 启动。可以通过设置环境变量来控制预热池中保留的前缀数量。

* `WARM_PREFIX_TARGET`,超出当前需求的前缀数量。
* `WARM_IP_TARGET`,超出当前需求的 IP 地址数量。
* `MINIMUM_IP_TARGET`,任何时候都要保持可用的最小 IP 地址数量。
* 如果设置了 `WARM_IP_TARGET` 和 `MINIMUM_IP_TARGET`,它们将覆盖 `WARM_PREFIX_TARGET`。

随着更多 Pod 被调度,将为现有 ENI 请求更多前缀。首先,VPC CNI 尝试为现有 ENI 分配一个新前缀。如果 ENI 已满,VPC CNI 将尝试为节点分配一个新 ENI。将附加新 ENI,直到达到实例类型定义的最大 ENI 限制。附加新 ENI 后,ipamd 将分配一个或多个前缀,以维持 `WARM_PREFIX_TARGET`、`WARM_IP_TARGET` 和 `MINIMUM_IP_TARGET` 设置。

![分配 IP 给 Pod 的流程图](./image-2.jpeg)

## 建议

### 何时使用前缀模式

如果您在工作节点上遇到 Pod 密度问题,请使用前缀模式。为了避免 VPC CNI 错误,我们建议在迁移到前缀模式之前,检查子网是否有连续的 /28 前缀块。请参阅"[使用子网预留来避免子网碎片化(IPv4)](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-cidr-reservation.html)"部分了解子网预留的详细信息。

为了向后兼容,已将 [max-pods](https://github.com/awslabs/amazon-eks-ami/blob/master/files/eni-max-pods.txt) 限制设置为支持辅助 IP 模式。要增加 Pod 密度,请将 `max-pods` 值指定给 Kubelet,并将 `--use-max-pods=false` 作为节点的用户数据。您可以考虑使用 [max-pod-calculator.sh](https://github.com/awslabs/amazon-eks-ami/blob/master/files/max-pods-calculator.sh) 脚本来计算 EKS 推荐的给定实例类型的最大 Pod 数量。请参阅 EKS [用户指南](https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html)了解示例用户数据。

```
./max-pods-calculator.sh --instance-type m5.large --cni-version ``1.9``.0 --cni-prefix-delegation-enabled
```

前缀分配模式对于使用 [CNI 自定义网络](https://docs.aws.amazon.com/eks/latest/userguide/cni-custom-network.html)的用户特别相关,在这种情况下,主 ENI 不用于 Pod。使用前缀分配,您仍然可以在几乎每种 Nitro 实例类型上附加更多 IP,即使主 ENI 不用于 Pod。

### 何时避免使用前缀模式

如果您的子网非常碎片化,没有足够的可用 IP 地址来创建 /28 前缀,请避免使用前缀模式。如果生成前缀的子网碎片化严重(大量使用的子网,二级 IP 地址分散),前缀附加可能会失败。通过创建一个新的子网并为其预留一个前缀,可以避免这个问题。

在前缀模式下,分配给工作节点的安全组将由 Pod 共享。如果您有安全要求,需要在共享计算资源上运行具有不同网络安全要求的应用程序,请考虑使用 [Pod 安全组](../sgpp/index.md)。

### 在同一节点组中使用相似的实例类型

您的节点组可能包含许多类型的实例。如果一个实例的最大 Pod 数量较低,则该值将应用于节点组中的所有节点。考虑在节点组中使用相似的实例类型,以最大化节点利用率。如果您使用 Karpenter 进行自动节点扩缩,我们建议在配置器 API 的要求部分设置 [node.kubernetes.io/instance-type](https://karpenter.sh/docs/concepts/nodepools/)。

!!! warning 
    特定节点组中所有节点的最大 Pod 数量由该组中任何单个实例类型的最低最大 Pod 数量定义。

### 配置 `WARM_PREFIX_TARGET` 以节省 IPv4 地址

[安装清单](https://github.com/aws/amazon-vpc-cni-k8s/blob/master/config/v1.9/aws-k8s-cni.yaml#L158)中 `WARM_PREFIX_TARGET` 的默认值为 1。在大多数情况下,`WARM_PREFIX_TARGET` 的推荐值 1 将提供快速 Pod 启动时间,同时最小化分配给实例的未使用 IP 地址。

如果您需要进一步节省每个节点的 IPv4 地址,请使用 `WARM_IP_TARGET` 和 `MINIMUM_IP_TARGET` 设置,它们会覆盖配置的 `WARM_PREFIX_TARGET`。通过将 `WARM_IP_TARGET` 设置为小于 16 的值,您可以防止 CNI 保留整个多余的前缀。

### 优先分配新前缀而不是附加新 ENI

将额外的前缀分配给现有 ENI 是一个比创建和附加新 ENI 到实例更快的 EC2 API 操作。使用前缀可以提高性能,同时节省 IPv4 地址分配。分配前缀通常在一秒钟内完成,而附加新 ENI 可能需要 10 秒。对于大多数用例,当运行在前缀模式下时,CNI 每个工作节点只需要一个 ENI。如果您可以承受(在最坏的情况下)每个节点最多 15 个未使用的 IP,我们强烈建议使用这种新的前缀分配网络模式,并实现由此带来的性能和效率收益。

### 使用子网预留来避免子网碎片化(IPv4)

当 EC2 将 /28 IPv4 前缀分配给 ENI 时,它必须是来自您子网的一个连续的 IP 地址块。如果生成前缀的子网碎片化严重(大量使用的子网,二级 IP 地址分散),前缀附加可能会失败,您将在 VPC CNI 日志中看到以下错误消息:

```
failed to allocate a private IP/Prefix address: InsufficientCidrBlocks: There are not enough free cidr blocks in the specified subnet to satisfy the request.
```

为了避免碎片化并为创建前缀提供足够的连续空间,您可以使用 [VPC 子网 CIDR 预留](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-cidr-reservation.html#work-with-subnet-cidr-reservations)在子网内预留 IP 空间供前缀专用。创建预留后,VPC CNI 插件将调用 EC2 API 从预留空间分配前缀。

建议创建一个新的子网,为前缀预留空间,并为该子网中运行的工作节点启用 VPC CNI 前缀分配。如果新子网专门用于在 EKS 集群中运行使用 VPC CNI 前缀分配的 Pod,则可以跳过前缀预留步骤。

### 避免降级 VPC CNI

前缀模式适用于 VPC CNI 1.9.0 及更高版本。一旦启用了前缀模式并将前缀分配给 ENI,就必须避免将 Amazon VPC CNI 加载项降级到 1.9.0 以下版本。如果决定降级 VPC CNI,您必须删除并重新创建节点。

### 在过渡到前缀委派期间替换所有节点

我们强烈建议您创建新的节点组来增加可用 IP 地址的数量,而不是对现有工作节点进行滚动替换。将所有现有节点隔离并排空,以安全地驱逐所有现有 Pod。为了防止服务中断,我们建议在生产集群上为关键工作负载实施 [Pod 中断预算](https://kubernetes.io/docs/tasks/run-application/configure-pdb)。新节点上的 Pod 将从分配给 ENI 的前缀中获得 IP 地址。确认 Pod 正在运行后,您可以删除旧节点和节点组。如果您使用托管节点组,请按照此处提到的步骤安全[删除节点组](https://docs.aws.amazon.com/eks/latest/userguide/delete-managed-node-group.html)。