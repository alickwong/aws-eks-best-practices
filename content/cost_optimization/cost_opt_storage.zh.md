---
日期: 2023-10-31
作者:
  - Chance Lee
---
# 成本优化 - 存储

## 概述

在某些情况下,您可能需要运行需要短期或长期保存数据的应用程序。对于此类用例,可以定义卷并由 Pod 挂载,以便其容器可以利用不同的存储机制。Kubernetes 支持不同类型的[卷](https://kubernetes.io/docs/concepts/storage/volumes/)用于临时和持久性存储。存储的选择主要取决于应用程序的要求。对于每种方法,都会产生成本影响,下面详述的做法将有助于您在 EKS 环境中实现需要某种形式存储的工作负载的成本效率。

## 临时卷

临时卷用于需要临时本地卷但不需要在重启后保留数据的应用程序。这包括对临时空间、缓存和只读输入数据(如配置数据和机密)的要求。您可以在此处找到更多关于 Kubernetes 临时卷的[详细信息](https://kubernetes.io/docs/concepts/storage/ephemeral-volumes/)。大多数临时卷(例如 emptyDir、configMap、downwardAPI、secret、hostpath)都由本地附加的可写设备(通常为根磁盘)或 RAM 支持,因此选择最具成本效益和性能的主机卷非常重要。

### 使用 EBS 卷

*我们建议从 [gp3](https://aws.amazon.com/ebs/general-purpose/) 作为主机根卷开始。*它是亚马逊 EBS 提供的最新一代通用 SSD 卷,每 GB 价格也比 gp2 卷低(高达 20%)。

### 使用亚马逊 EC2 实例存储

[亚马逊 EC2 实例存储](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/InstanceStorage.html)为您的 EC2 实例提供临时块级存储。EC2 实例存储提供的存储通过物理连接到主机的磁盘访问。与亚马逊 EBS 不同,您只能在启动实例时附加实例存储卷,这些卷仅在实例的生命周期内存在。它们无法与其他实例分离和重新附加。您可以在此处了解更多关于亚马逊 EC2 实例存储的[信息](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/InstanceStorage.html)。*使用实例存储卷不会产生任何额外费用。*这使它们(实例存储卷)比具有大型 EBS 卷的常规 EC2 实例更具成本效益。

要在 Kubernetes 中使用本地存储卷,您应该使用亚马逊 EC2 用户数据[对磁盘进行分区、配置和格式化](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-add-user-data.html),以便可以将卷挂载为 pod 规范中的[HostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath)。或者,您可以利用[本地持久卷静态供应商](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner)来简化本地存储管理。本地持久卷静态供应商允许您通过标准的 Kubernetes PersistentVolumeClaim (PVC) 接口访问本地实例存储卷。此外,它还将提供包含节点亲和性信息的 PersistentVolumes (PVs),以将 Pod 调度到正确的节点。尽管它使用 Kubernetes PersistentVolumes,但 EC2 实例存储卷本质上是临时的。写入临时磁盘的数据仅在实例的生命周期内可用。当实例终止时,数据也会消失。更多详细信息请参考此[博客](https://aws.amazon.com/blogs/containers/eks-persistent-volumes-for-instance-store/)。

请记住,在使用亚马逊 EC2 实例存储卷时,总 IOPS 限制与主机共享,并且将 Pod 绑定到特定主机。在采用亚马逊 EC2 实例存储卷之前,您应该仔细审查您的工作负载要求。

## 持久卷

Kubernetes 通常与运行无状态应用程序相关联。但是,也有一些情况下您可能希望运行需要保留持久数据或从一个请求到下一个请求保留信息的微服务。数据库是此类用例的一个常见示例。但是,Pod 及其内部的容器或进程都是短暂的。为了在 Pod 生命周期之外保持数据,您可以使用 PV 来定义对独立于 Pod 的特定位置的存储的访问。*PV 相关的成本高度依赖于所使用的存储类型以及应用程序的使用方式。*

在亚马逊 EKS 上,有不同类型的存储选项支持 Kubernetes PV,列在[此处](https://docs.aws.amazon.com/eks/latest/userguide/storage.html)。下面介绍的存储选项包括亚马逊 EBS、亚马逊 EFS、亚马逊 FSx for Lustre 和亚马逊 FSx for NetApp ONTAP。

### 亚马逊弹性块存储 (EBS) 卷

亚马逊 EBS 卷可以作为 Kubernetes PV 使用,提供块级存储卷。这些非常适合依赖随机读写和吞吐量密集型应用程序的数据库,这些应用程序执行长时间的连续读写。[亚马逊弹性块存储容器存储接口 (CSI) 驱动程序](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html)允许亚马逊 EKS 集群管理亚马逊 EBS 卷的生命周期以用作持久卷。容器存储接口使 Kubernetes 与存储系统之间的交互更加容易和便捷。当 CSI 驱动程序部署到您的 EKS 集群时,您可以通过原生的 Kubernetes 存储资源(如持久卷 (PV)、持久卷声明 (PVC) 和存储类 (SC))访问其功能。此[链接](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/tree/master/examples/kubernetes)提供了如何使用亚马逊 EBS CSI 驱动程序与亚马逊 EBS 卷交互的实际示例。

#### 选择合适的卷

*我们建议使用最新一代的块存储(gp3),因为它在价格和性能之间提供了合适的平衡*。它还允许您独立于卷大小扩展卷 IOPS 和吞吐量,无需预配额外的块存储容量。如果您目前正在使用 gp2 卷,我们强烈建议您迁移到 gp3 卷。此[博客](https://aws.amazon.com/blogs/containers/migrating-amazon-eks-clusters-from-gp2-to-gp3-ebs-volumes/)解释了如何在亚马逊 EKS 集群上从 *gp2* 迁移到 *gp3*。

当您有需要更高性能且需要的卷大于单个 [gp3 卷支持的容量](https://aws.amazon.com/ebs/general-purpose/)时,您应该考虑使用 [io2 block express](https://aws.amazon.com/ebs/provisioned-iops/)。这种存储类型非常适合您最大、最 I/O 密集和最关键的部署,例如 SAP HANA 或其他具有低延迟要求的大型数据库。请记住,实例的 EBS 性能受实例性能限制的约束,因此并非所有实例都支持 io2 block express 卷。您可以在此[文档](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/provisioned-iops.html)中检查支持的实例类型和其他注意事项。

*单个 gp3 卷最多可支持 16,000 个最大 IOPS、1,000 MiB/s 最大吞吐量、最大 16TiB。最新一代的 Provisioned IOPS SSD 卷可提供高达 256,000 IOPS、4,000 MiB/s 吞吐量和 64TiB。*

在这些选项中,您应该根据应用程序的需求最佳匹配存储性能和成本。

#### 随时监控和优化

了解应用程序的基线性能并监控所选卷以检查是否满足您的要求/期望,或者是否过度配置(例如,配置的 IOPS 未被充分利用)非常重要。

您可以逐步增加卷的大小,而不是一开始就分配一个大卷。您可以使用亚马逊弹性块存储 CSI 驱动程序(aws-ebs-csi-driver)中的[卷调整](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/tree/master/examples/kubernetes/resizing)功能动态调整卷大小。*请记住,您只能增加 EBS 卷大小。*

要识别和删除任何悬挂的 EBS 卷,您可以使用 [AWS 信任顾问的成本优化类别](https://docs.aws.amazon.com/awssupport/latest/user/cost-optimization-checks.html)。此功能可帮助您识别未附加的卷或在一段时间内写入活动非常低的卷。还有一个名为 [Popeye](https://github.com/derailed/popeye) 的云原生只读工具,可扫描实时 Kubernetes 集群并报告已部署资源和配置的潜在问题。例如,它可以扫描未使用的 PV 和 PVC,并检查它们是否已绑定或是否存在任何卷挂载错误。

有关深入监控的更多信息,请参考[EKS 成本优化可观察性指南](https://aws.github.io/aws-eks-best-practices/cost_optimization/cost_opt_observability/)。

您可以考虑的另一个选项是[AWS Compute Optimizer 亚马逊 EBS 卷建议](https://docs.aws.amazon.com/compute-optimizer/latest/ug/view-ebs-recommendations.html)。该工具可自动识别最佳卷配置和所需的性能水平。例如,它可用于确定与过去 14 天最大利用率相关的最佳设置,包括配置的 IOPS、卷大小和 EBS 卷类型。它还量化了其建议带来的潜在每月成本节省。您可以查看此[博客](https://aws.amazon.com/blogs/storage/cost-optimizing-amazon-ebs-volumes-using-aws-compute-optimizer/)了解更多详细信息。

#### 备份保留策略

您可以通过拍摄时间点快照来备份亚马逊 EBS 卷上的数据。亚马逊 EBS CSI 驱动程序支持卷快照。您可以按照此处概述的步骤[创建快照并使用 EBS PV 还原](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/master/examples/kubernetes/snapshot/README.md)。

后续快照是增量备份,这意味着只保存您最近一次快照后设备上发生变化的块。这可最大限度地减少创建快照所需的时间,并通过不重复数据来节省存储成本。但是,在没有适当保留策略的情况下,旧的 EBS 快照数量不断增加会导致在大规模运营时产生意外成本。如果您通过 AWS API 直接备份亚马逊 EBS 卷,可以利用[亚马逊数据生命周期管理器](https://aws.amazon.com/ebs/data-lifecycle-manager/)(DLM),它提供了一个自动化、基于策略的生命周期管理解决方案,用于管理亚马逊弹性块存储 (EBS) 快照和基于 EBS 的亚马逊机器映像 (AMI)。控制台使管理 EBS 快照和 AMI 的创建、保留和删除变得更加容易。

!!! note
    目前无法通过亚马逊 EBS CSI 驱动程序利用亚马逊数据生命周期管理器。

在 Kubernetes 环境中,您可以利用名为 [Velero](https://velero.io/) 的开源工具来备份您的 EBS 持久卷。您可以在计划作业时设置 TTL 标志来过期备份。Velero 提供了一个[指南](https://velero.io/docs/v1.12/how-velero-works/#set-a-backup-to-expire)作为示例。

### 亚马逊弹性文件系统 (EFS)

[亚马逊弹性文件系统 (EFS)](https://aws.amazon.com/efs/)是一个无服务器、完全弹性的文件系统,让您可以使用标准的文件系统接口和文件系统语义共享文件数据,适用于广泛的工作负载和应用程序。工作负载和应用程序的示例包括 Wordpress 和 Drupal、JIRA 和 Git 等开发工具,以及 Jupyter 等共享笔记本系统以及家目录。

亚马逊 EFS 的主要优势之一是它可以被多个节点和多个可用区域上的多个容器挂载。另一个优势是您只需为使用的存储付费。EFS 文件系统将自动根据您添加和删除文件的情况进行扩展和收缩,从而消除了容量规划的需求。

要在 Kubernetes 中使用亚马逊 EFS,您需要使用亚马逊弹性文件系统容器存储接口 (CSI) 驱动程序 [aws-efs-csi-driver](https://github.com/kubernetes-sigs/aws-efs-csi-driver)。目前,该驱动程序可以动态创建[访问点](https://docs.aws.amazon.com/efs/latest/ug/efs-access-points.html)。但是,必须先配置亚马逊 EFS 文件系统,并将其作为输入提供给 Kubernetes 存储类参数。

#### 选择合适的 EFS 存储类

亚马逊 EFS 提供[四种存储类](https://docs.aws.amazon.com/efs/latest/ug/storage-classes.html)。

两个标准存储类:

* 亚马逊 EFS 标准
* [亚马逊 EFS 标准-不频繁访问](https://aws.amazon.com/blogs/aws/optimize-storage-cost-with-reduced-pricing-for-amazon-efs-infrequent-access/)(EFS 标准-IA)

两个单区域存储类:

* [亚马逊 EFS 单区域](https://aws.amazon.com/blogs/aws/new-lower-cost-one-zone-storage-classes-for-amazon-elastic-file-system/)
* 亚马逊 EFS 单区域-不频繁访问(EFS 单区域-IA)

不频繁访问 (IA) 存储类针对的是每天访问不频繁的文件。通过使用亚马逊 EFS 生命周期管理,您可以将在生命周期策略(7、14、30、60 或 90 天)期间未被访问的文件移动到 IA 存储类,*这可将存储成本降低高达 92% 与 EFS 标准和 EFS 单区域存储类相比*。

通过 EFS 智能分层,生命周期管理会监控您的文件系统的访问模式,并自动将文件移动到最合适的存储类。

!!! note
    aws-efs-csi-driver 目前无法控制更改存储类、生命周期管理或智能分层。这些应该在 AWS 控制台或通过 EFS API 手动设置。

!!! note
    aws-efs-csi-driver 与基于 Windows 的容器映像不兼容。

!!! note
    当启用 *vol-metrics-opt-in*(发出卷指标)时,存在内存问题,因为 [DiskUsage](https://github.com/kubernetes/kubernetes/blob/ee265c92fec40cd69d1de010b477717e4c142492/pkg/volume/util/fs/fs.go#L66) 函数消耗的内存量与您的文件系统大小成正比。*目前,我们建议在大型文件系统上禁用 `--vol-metrics-opt-in` 选项,以避免消耗过多内存。有关更多详细信息,请参见 github 问题[链接](https://github.com/kubernetes-sigs/aws-efs-csi-driver/issues/1104)。*

### 亚马逊 FSx for Lustre

Lustre 是一个高性能的并行文件系统,通常用于需要高达数百 GB/s 的吞吐量和亚毫秒级延迟的工作负载。它用于机器学习训练、金融建模、HPC 和视频处理等场景。[亚马逊 FSx for Lustre](https://aws.amazon.com/fsx/lustre/)提供了一个完全托管的共享存储,具有可扩展性和性能,并与亚马逊 S3 无缝集成。

您可以使用亚马逊 EKS 或您在 AWS 上自管理的 Kubernetes 集群上的 [FSx for Lustre CSI 驱动程序](https://github.com/kubernetes-sigs/aws-fsx-csi-driver)来使用 FSx for Lustre 支持的 Kubernetes 持久存储卷。请参阅[亚马逊 EKS 文档](https://docs.aws.amazon.com/eks/latest/userguide/fsx-csi.html)了解更多详细信息和示例。

#### 链接到亚马逊 S3

建议将位于亚马逊 S3 上的高度耐用的长期数据存储库与您的 FSx for Lustre 文件系统链接。链接后,大型数据集会根据需要从亚马逊 S3 延迟加载到 FSx for Lustre 文件系统。您还可以将分析和结果运行回 S3,然后删除您的 [Lustre] 文件系统。

#### 选择合适的部署和存储选项

FSx for Lustre 提供不同的部署选项。第一个选项称为 *scratch*,它不会复制数据,而第二个选项称为 *persistent*,顾名思义,它会持久保存数据。

第一个选项(*scratch*)可用于*降低临时较短期数据处理的成本*。持久部署选项*专为需要长期存储而设计*,它会自动在 AWS 可用区内复制数据。它还支持 SSD 和 HDD 存储。

您可以在 FSx for lustre 文件系统的 Kubernetes StorageClass 的参数中配置所需的部署类型。此[链接](https://github.com/kubernetes-sigs/aws-fsx-csi-driver/tree/master/examples/kubernetes/dynamic_provisioning#edit-storageclass)提供了示例模板。

!!! note
    对于对延迟敏感的工作负载或需要最高 IOPS/吞吐量的工作负载,您应该选择 SSD 存储。对于不太关注延迟但需要高吞吐量的工作负载,您应该选择 HDD 存储。

#### 启用数据压缩

您还可以通过在文件系统参数中指定"LZ4"作为数据压缩类型来启用数据压缩。启用后,所有新写入的文件在写入磁盘之前都会自动压缩在 FSx for Lustre 上,在读取时会自动解压缩。LZ4 数据压缩算法是无损的,因此可以完全从压缩数据中重建原始数据。

您可以在 FSx for lustre 文件系统的 Kubernetes StorageClass 的参数中配置数据压缩类型为 LZ4。当值设置为 NONE(默认)时,压缩将被禁用。此[链接](https://github.com/kubernetes-sigs/aws-fsx-csi-driver/tree/master/examples/kubernetes/dynamic_provisioning#edit-storageclass)提供了示例模板。

!!! note
    亚马逊 FSx for Lustre 与基于 Windows 的容器映像不兼容。

### 亚马逊 FSx for NetApp ONTAP

[亚马逊 FSx for NetApp ONTAP](https://aws.amazon.com/fsx/netapp-ontap/)是一个完全托管的共享存储,建立在 NetApp 的 ONTAP 文件系统之上。FSx for ONTAP 提供功能丰富、快速和灵活的共享文件存储,可广泛从在 AWS 或本地运行的 Linux、Windows 和 macOS 计算实例访问。

亚马逊 FSx for NetApp ONTAP 支持两层存储:*1/主存储层*和*2/容量池层*。

*主存储层*是一个配置的高性能 SSD 基础层,用于活跃的、对延迟敏感的数据。完全弹性的*容量池层*针对不经常访问的数据进行了成本优化,随着数据被分层到该层而自动扩展,并提供虚乎无限的数 PB 容量。您可以在容量池存储上启用数据压缩和重复数据删除,进一步减少存储容量的使用。NetApp 的原生、基于策略的 FabricPool 功能会持续监控数据访问模式,自动双向地在存储层之间传输数据,以优化性能和成本。

NetApp 的 Astra Trident 提供了使用 CSI 驱动程序的动态存储编排,允许亚马逊 EKS 集群管理由亚马逊 FSx for NetApp ONTAP 文件系统支持的持久卷 PV 的生命周期。要开始使用,请参阅 Astra Trident 文档中的[将 Astra Trident 与亚马逊 FSx for NetApp ONTAP 一起使用](https://docs.netapp.com/us-en/trident/trident-use/trident-fsx.html)。

## 其他注意事项

### 最小化容器镜像大小

部署容器后,容器镜像会被缓存在主机上的多个层中。通过减小镜像大小,可以减少主机上所需的存储空间。

通过使用精简的基础镜像,如 [scratch](https://hub.docker.com/_/scratch) 镜像或 [distroless](https://github.com/GoogleContainerTools/distroless) 容器镜像(仅包含您的应用程序及其运行时依赖项),*您不仅可以降低存储成本,还可以获得其他附带好处,如减少攻击面和更短的镜像拉取时间。*

您还应该考虑使用开源工具,如 [Slim.ai](https://www.slim.ai/docs/quickstart),它提供了一种简单、安全的方式来创建最小化的镜像。

软件包、工具、应用程序依赖项和库的多个层会轻易地使容器镜像膨胀。通过使用多阶段构建,您可以选择性地从一个阶段复制制品到另一个阶段,从最终镜像中排除不必要的所有内容。您可以在此处查看更多镜像构建最佳实践[链接](https://docs.docker.com/get-started/09_image_best/)。

另一个需要考虑的是缓存镜像的保留时间。当磁盘使用量达到一定程度时,您可能需要清理过时的镜像。这样做将确保为主机操作留出足够的空间。默认情况下,[kubelet](https://kubernetes.io/docs/reference/generated/kubelet)每五分钟对未使用的镜像进行一次垃圾回收,每分钟对未使用的容器进行一次垃圾回收。

*要配置未使用的容器和镜像垃圾回收的选项,请使用[配置文件](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/)调整 kubelet,并使用 [`KubeletConfiguration`](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)资源类型更改与垃圾回收相关的参数。*

您可以在 Kubernetes [文档](https://kubernetes.io/docs/concepts/architecture/garbage-collection/#containers-images)中了解更多信息。