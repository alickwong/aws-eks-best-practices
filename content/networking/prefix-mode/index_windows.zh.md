!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

# Windows 前缀模式
在 Amazon EKS 中,在 Windows 主机上运行的每个 Pod 默认情况下都会被 [VPC 资源控制器](https://github.com/aws/amazon-vpc-resource-controller-k8s)分配一个辅助 IP 地址。这个 IP 地址是一个可路由的 VPC 地址,它是从主机子网中分配的。在 Linux 上,附加到实例的每个 ENI 都有多个插槽,可以由辅助 IP 地址或 /28 CIDR（前缀）填充。但是,Windows 主机只支持单个 ENI 及其可用的插槽。仅使用辅助 IP 地址可能会人为地限制您在 Windows 主机上可以运行的 Pod 数量,即使有大量可用于分配的 IP 地址。

为了提高 Windows 主机上的 Pod 密度,特别是在使用较小的实例类型时,您可以为 Windows 节点启用**前缀委派**。启用前缀委派后,/28 IPv4 前缀将分配给 ENI 插槽,而不是辅助 IP 地址。可以通过在 `amazon-vpc-cni` 配置映射中添加 `enable-windows-prefix-delegation: "true"` 条目来启用前缀委派。这是您需要设置 `enable-windows-ipam: "true"` 条目以启用 Windows 支持的同一配置映射。

请按照 [EKS 用户指南](https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html)中提到的说明为 Windows 节点启用前缀委派模式。

![两个工作者子网的插图,比较 ENI 辅助 IP 与 ENI 委派前缀](./windows-1.jpg)

图：辅助 IP 模式与前缀委派模式的比较

可分配给网络接口的最大 IP 地址数量取决于实例类型及其大小。分配给网络接口的每个前缀都会消耗一个可用插槽。例如,`c5.large` 实例每个网络接口有 `10` 个插槽的限制。网络接口的第一个插槽始终由接口的主 IP 地址占用,这样您就剩下 9 个插槽用于前缀和/或辅助 IP 地址。如果这些插槽被分配给前缀,则节点可以支持 (9 * 16) 144 个 IP 地址,而如果它们被分配给辅助 IP 地址,则只能支持 9 个 IP 地址。有关[每个实例类型的网络接口 IP 地址数量](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI)和[将前缀分配给网络接口](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-prefix-eni.html)的更多信息,请参阅文档。

在工作节点初始化期间,VPC 资源控制器会将一个或多个前缀分配给主 ENI,以便通过维护 IP 地址的预热池来加快 Pod 启动。可以通过在 `amazon-vpc-cni` 配置映射中设置以下配置参数来控制要保留在预热池中的前缀数量。

* `warm-prefix-target`，超出当前需求的前缀数量。

* `warm-ip-target`，超出当前需求的 IP 地址数量。
* `minimum-ip-target`，任何时候都要保持可用的最小 IP 地址数量。
* 如果设置了 `warm-ip-target` 和/或 `minimum-ip-target`，将会覆盖 `warm-prefix-target`。

随着更多 Pod 被调度到节点上，将会请求为现有 ENI 分配更多前缀。当 Pod 被调度到节点上时，VPC 资源控制器会首先尝试从节点上现有前缀中分配一个 IPv4 地址。如果无法完成，只要子网有足够的容量，就会请求一个新的 IPv4 前缀。

![流程图显示了将 IP 地址分配给 Pod 的过程](./windows-2.jpg)

图：将 IPv4 地址分配给 Pod 的工作流程

## 建议
### 在以下情况下使用前缀委派
如果您在工作节点上遇到 Pod 密度问题，请使用前缀委派。为了避免错误,我们建议在迁移到前缀模式之前,先检查子网是否有连续的 /28 前缀地址块。请参考"[使用子网预留来避免子网碎片化(IPv4)](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-cidr-reservation.html)"部分了解子网预留的详细信息。

默认情况下,Windows 节点上的 `max-pods` 设置为 `110`。对于大多数实例类型,这应该足够了。如果您想增加或减少此限制,请在用户数据中的引导命令中添加以下内容:
```
-KubeletExtraArgs '--max-pods=example-value'
```

关于 Windows 节点的 bootstrap 配置参数的更多详细信息,请访问文档[此处](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-windows-ami.html#bootstrap-script-configuration-parameters)。

### 避免前缀委派
如果您的子网非常碎片化,并且没有足够的可用 IP 地址来创建 /28 前缀,请避免使用前缀模式。如果生成前缀的子网碎片化(一个使用率很高且二级 IP 地址分散的子网),前缀附加可能会失败。通过创建一个新的子网并保留一个前缀,可以避免这个问题。

### 配置前缀委派参数以节省 IPv4 地址
可以使用 `warm-prefix-target`、`warm-ip-target` 和 `minimum-ip-target` 来微调预缩放和动态缩放与前缀相关的行为。默认情况下,使用以下值:
```
warm-ip-target: "1"
minimum-ip-target: "3"
```

通过微调这些配置参数，您可以实现保护 IP 地址和确保由于 IP 地址分配而导致的 Pod 延迟降低之间的最佳平衡。有关这些配置参数的更多信息，请访问[此处](https://github.com/aws/amazon-vpc-resource-controller-k8s/blob/master/docs/windows/prefix_delegation_config_options.md)的文档。

### 使用子网预留来避免子网碎片化(IPv4)
当 EC2 向 ENI 分配 /28 IPv4 前缀时，它必须是来自您子网的连续 IP 地址块。如果生成前缀的子网已碎片化(一个高度使用的子网,具有分散的辅助 IP 地址)，前缀附加可能会失败,您将看到以下节点事件:
```
InsufficientCidrBlocks: The specified subnet does not have enough free cidr blocks to satisfy the request
```

为了避免碎片化并为创建前缀提供足够的连续空间,请使用[VPC子网CIDR预留](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-cidr-reservation.html#work-with-subnet-cidr-reservations)来预留子网内的IP空间,供前缀专用。创建预留后,预留块中的IP地址将不会分配给其他资源。这样,VPC资源控制器在向节点ENI分配前缀时就能获得可用的前缀。

建议创建一个新的子网,为前缀预留空间,并为在该子网中运行的工作节点启用前缀分配。如果新子网仅用于运行您的EKS集群中的Pod,并启用了前缀委派,则可以跳过前缀预留步骤。

### 从辅助IP模式迁移到前缀委派模式或反之时,请全部替换节点
强烈建议您创建新的节点组,以增加可用IP地址的数量,而不是对现有工作节点进行滚动替换。

使用自管理节点组时,过渡步骤如下:

* 增加集群容量,使新节点能够容纳您的工作负载
* 为Windows启用/禁用前缀委派功能
* 隔离并排空所有现有节点,安全地驱逐所有现有Pod。为了防止服务中断,我们建议在生产集群的关键工作负载上实施[Pod中断预算](https://kubernetes.io/docs/tasks/run-application/configure-pdb)。
* 确认Pod正在运行后,您可以删除旧节点和节点组。新节点上的Pod将从分配给节点ENI的前缀中获得IPv4地址。

使用托管节点组时,过渡步骤如下:

* 为Windows启用/禁用前缀委派功能
* 按照[此处](https://docs.aws.amazon.com/eks/latest/userguide/update-managed-node-group.html)所述的步骤更新节点组。这执行与上述类似的步骤,但由EKS管理。

!!! warning
    在同一模式下运行节点上的所有Pod

对于Windows,我们建议您避免同时在辅助IP模式和前缀委派模式下运行Pod。当您从辅助IP模式迁移到前缀委派模式或反之时,可能会出现这种情况。
在这种情况下,您的正在运行的 Pods 不会受到影响,但节点的 IP 地址容量可能会存在不一致。例如,考虑一个 t3.xlarge 节点,它有 14 个用于辅助 IPv4 地址的插槽。如果您正在运行 10 个 Pods,那么 ENI 上的 10 个插槽将被辅助 IP 地址所占用。在启用前缀委派后,向 kube-api 服务器公布的容量将是 (14 个插槽 * 每个前缀 16 个 IP 地址) 244,但实际容量在该时刻将是 (4 个剩余插槽 * 每个前缀 16 个地址) 64。这种公布容量和实际剩余容量之间的不一致可能会导致在可分配的 IP 地址不足的情况下运行更多 Pods 时出现问题。

也就是说,您可以使用上述迁移策略安全地将 Pods 从辅助 IP 地址转换为从前缀获得的地址。在切换模式时,Pods 将继续正常运行,并且:

* 从辅助 IP 模式切换到前缀委派模式时,分配给正在运行的 Pods 的辅助 IP 地址不会被释放。前缀将被分配到空闲的插槽。一旦 Pod 被终止,使用的辅助 IP 和插槽将被释放。
* 从前缀委派模式切换到辅助 IP 模式时,当前缀范围内的所有 IP 地址不再分配给 Pods 时,该前缀将被释放。如果前缀中的任何 IP 地址被分配给 Pod,则该前缀将一直保留,直到 Pods 被终止。

### 调试前缀委派问题
您可以使用我们的调试指南[此处](https://github.com/aws/amazon-vpc-resource-controller-k8s/blob/master/docs/troubleshooting.md)深入了解您在 Windows 上使用前缀委派时遇到的问题。
