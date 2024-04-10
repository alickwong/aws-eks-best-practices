# Windows 前缀模式

在 Amazon EKS 中,在 Windows 主机上运行的每个 Pod 默认都会被 [VPC 资源控制器](https://github.com/aws/amazon-vpc-resource-controller-k8s)分配一个辅助 IP 地址。这个 IP 地址是一个可路由的 VPC 地址,是从主机子网中分配的。在 Linux 上,附加到实例的每个 ENI 都有多个插槽,可以填充辅助 IP 地址或 /28 CIDR(前缀)。但是,Windows 主机只支持单个 ENI 及其可用的插槽。仅使用辅助 IP 地址可能会人为地限制您在 Windows 主机上可以运行的 Pod 数量,即使有大量可用于分配的 IP 地址。

为了提高 Windows 主机上的 Pod 密度,特别是在使用较小的实例类型时,您可以为 Windows 节点启用**前缀委派**。启用前缀委派后,/28 IPv4 前缀将分配给 ENI 插槽,而不是辅助 IP 地址。可以通过在 `amazon-vpc-cni` 配置映射中添加 `enable-windows-prefix-delegation: "true"` 条目来启用前缀委派。这是您需要设置 `enable-windows-ipam: "true"` 条目以启用 Windows 支持的同一配置映射。

请按照 [EKS 用户指南](https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html)中提到的说明为 Windows 节点启用前缀委派模式。

![两个工作者子网的插图,比较 ENI 辅助 IP 与 ENI 委派前缀](./windows-1.jpg)

图:辅助 IP 模式与前缀委派模式的比较

可以分配给网络接口的最大 IP 地址数量取决于实例类型及其大小。分配给网络接口的每个前缀都会消耗一个可用插槽。例如,`c5.large` 实例每个网络接口有 `10` 个插槽的限制。网络接口的第一个插槽始终由接口的主 IP 地址占用,这样您就剩下 9 个插槽用于前缀和/或辅助 IP 地址。如果这些插槽被分配给前缀,则节点可以支持 (9 * 16) 144 个 IP 地址,而如果它们被分配给辅助 IP 地址,则只能支持 9 个 IP 地址。有关[每个实例类型的网络接口 IP 地址数量](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI)和[将前缀分配给网络接口](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-prefix-eni.html)的更多信息,请参阅相关文档。

在工作节点初始化期间,VPC 资源控制器会将一个或多个前缀分配给主 ENI,以便通过维护 IP 地址的预热池来加快 Pod 启动。可以通过在 `amazon-vpc-cni` 配置映射中设置以下配置参数来控制预热池中要保留的前缀数量。

* `warm-prefix-target`,要在当前需求之外分配的前缀数量。
* `warm-ip-target`,要在当前需求之外分配的 IP 地址数量。
* `minimum-ip-target`,任何时候都要保持可用的最小 IP 地址数量。
* 如果设置了 `warm-ip-target` 和/或 `minimum-ip-target`,它们将覆盖 `warm-prefix-target`。

随着更多 Pod 被调度到节点上,将为现有 ENI 请求更多前缀。当 Pod 被调度到节点上时,VPC 资源控制器将首先尝试从节点上现有前缀中分配 IPv4 地址。如果这不可能,只要子网有足够的容量,就会请求新的 IPv4 前缀。

![分配 IP 到 Pod 的流程图](./windows-2.jpg)

图:分配 IPv4 地址到 Pod 的工作流程

## 建议

### 何时使用前缀委派
如果您在工作节点上遇到 Pod 密度问题,请使用前缀委派。为了避免错误,我们建议在迁移到前缀模式之前,先检查子网是否有连续的地址块用于 /28 前缀。请参阅"[使用子网预留来避免子网碎片化(IPv4)](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-cidr-reservation.html)"部分了解子网预留的详细信息。

默认情况下,Windows 节点上的 `max-pods` 设置为 `110`。对于大多数实例类型,这应该足够了。如果您想增加或减少此限制,请在用户数据中的引导命令中添加以下内容:
```
-KubeletExtraArgs '--max-pods=example-value'
```
有关 Windows 节点引导配置参数的更多详细信息,请访问[此处](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-windows-ami.html#bootstrap-script-configuration-parameters)的文档。

### 何时避免使用前缀委派
如果您的子网非常碎片化,没有足够的可用 IP 地址来创建 /28 前缀,请避免使用前缀模式。如果生成前缀的子网碎片化严重(一个使用密集的子网,有散布的辅助 IP 地址),前缀附加可能会失败。通过创建一个新的子网并预留一个前缀,可以避免这个问题。

### 配置前缀委派参数以节省 IPv4 地址
`warm-prefix-target`、`warm-ip-target` 和 `minimum-ip-target` 可用于微调预缩放和动态缩放的行为。默认情况下,使用以下值:
```
warm-ip-target: "1"
minimum-ip-target: "3"
```
通过微调这些配置参数,您可以实现在保护 IP 地址和确保由于 IP 地址分配而导致的 Pod 延迟降低之间达到最佳平衡。有关这些配置参数的更多信息,请访问[此处](https://github.com/aws/amazon-vpc-resource-controller-k8s/blob/master/docs/windows/prefix_delegation_config_options.md)的文档。

### 使用子网预留来避免子网碎片化(IPv4)
当 EC2 将 /28 IPv4 前缀分配给 ENI 时,它必须是来自您子网的一个连续的 IP 地址块。如果生成前缀的子网碎片化严重(一个高度使用的子网,有散布的辅助 IP 地址),前缀附加可能会失败,您会看到以下节点事件:
```
InsufficientCidrBlocks: The specified subnet does not have enough free cidr blocks to satisfy the request
```
为了避免碎片化并为创建前缀保留足够的连续空间,请使用 [VPC 子网 CIDR 预留](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-cidr-reservation.html#work-with-subnet-cidr-reservations)在子网内预留 IP 空间,供前缀专用。创建预留后,预留块中的 IP 地址将不会分配给其他资源。这样,VPC 资源控制器在向节点 ENI 分配请求时就能获得可用的前缀。

建议创建一个新的子网,为前缀预留空间,并为在该子网中运行的工作节点启用前缀分配。如果新子网专门用于在您的 EKS 集群中运行启用了前缀委派的 Pod,那么您可以跳过前缀预留步骤。

### 从辅助 IP 模式迁移到前缀委派模式或反之亦然时,请替换所有节点
我们强烈建议您创建新的节点组来增加可用 IP 地址的数量,而不是对现有工作节点进行滚动替换。

使用自管理节点组时,过渡的步骤如下:

* 增加集群的容量,使新节点能够容纳您的工作负载
* 为 Windows 启用/禁用前缀委派功能
* 隔离并排空所有现有节点,以安全地驱逐所有现有 Pod。为了防止服务中断,我们建议在生产集群上为关键工作负载实施 [Pod 中断预算](https://kubernetes.io/docs/tasks/run-application/configure-pdb)。
* 确认 Pod 正在运行后,您可以删除旧节点和节点组。新节点上的 Pod 将从分配给节点 ENI 的前缀中获得 IPv4 地址。

使用托管节点组时,过渡的步骤如下:

* 为 Windows 启用/禁用前缀委派功能
* 按照[此处](https://docs.aws.amazon.com/eks/latest/userguide/update-managed-node-group.html)提到的步骤更新节点组。这执行与上述类似的步骤,但由 EKS 管理。

!!! warning
    在同一节点上运行所有 Pod 都使用相同的模式

对于 Windows,我们建议您避免同时在辅助 IP 模式和前缀委派模式下运行 Pod。当您从辅助 IP 模式迁移到前缀委派模式或反之亦然时,可能会出现这种情况。

虽然这不会影响您正在运行的 Pod,但在节点的 IP 地址容量方面可能会存在不一致。例如,考虑一个 t3.xlarge 节点,它有 14 个用于辅助 IPv4 地址的插槽。如果您正在运行 10 个 Pod,那么 ENI 上的 10 个插槽将被辅助 IP 地址占用。在启用前缀委派后,advertised 到 kube-api 服务器的容量将是 (14 个插槽 * 16 个前缀 IP 地址) 244,但实际容量在那一刻将是 (4 个剩余插槽 * 16 个前缀 IP 地址) 64。这种 advertised 容量和实际可用容量(剩余插槽)之间的不一致可能会导致问题,如果您运行的 Pod 数量超过可用于分配的 IP 地址数量。

也就是说,您可以使用上述迁移策略安全地将 Pod 从辅助 IP 地址迁移到从前缀获得的地址。在切换模式时,Pod 将继续正常运行,并且:

* 从辅助 IP 模式切换到前缀委派模式时,分配给正在运行的 Pod 的辅助 IP 地址不会被释放。将为空闲插槽分配前缀。一旦 Pod 被终止,它使用的辅助 IP 和插槽将被释放。
* 从前缀委派模式切换到辅助 IP 模式时,当前缀范围内的所有 IP 地址不再分配给 Pod 时,该前缀将被释放。如果前缀中的任何 IP 地址分配给 Pod,则该前缀将一直保留,直到 Pod 被终止。

### 调试前缀委派问题
您可以使用我们在[此处](https://github.com/aws/amazon-vpc-resource-controller-k8s/blob/master/docs/troubleshooting.md)提供的调试指南,深入了解您在 Windows 上遇到的前缀委派问题。