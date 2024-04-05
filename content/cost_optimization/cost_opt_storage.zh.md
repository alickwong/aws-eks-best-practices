
# 成本优化 - 存储

## 概述

在某些情况下,您可能需要运行需要短期或长期保存数据的应用程序。对于这种用例,可以定义卷并由 Pod 挂载,以便其容器可以利用不同的存储机制。Kubernetes 支持不同类型的[卷](https://kubernetes.io/docs/concepts/storage/volumes/)用于临时和持久性存储。存储的选择主要取决于应用程序的要求。对于每种方法,都会产生成本影响,下面详述的做法将有助于您在 EKS 环境中实现需要某种形式存储的工作负载的成本效率。

## 临时卷

临时卷用于需要临时本地卷但不需要在重启后保留数据的应用程序。例子包括对临时空间、缓存和只读输入数据(如配置数据和密钥)的需求。您可以在[此处](https://kubernetes.io/docs/concepts/storage/ephemeral-volumes/)找到更多关于 Kubernetes 临时卷的详细信息。大多数临时卷(如 emptyDir、configMap、downwardAPI、secret、hostpath)都由本地附加的可写设备(通常为根磁盘)或 RAM 支持,因此选择最具成本效益和性能的主机卷非常重要。

### 使用 EBS 卷

*我们建议从 [gp3](https://aws.amazon.com/ebs/general-purpose/) 作为主机根卷开始。*它是 Amazon EBS 提供的最新通用型 SSD 卷,每 GB 价格也比 gp2 卷低(高达 20%)。

### 使用 Amazon EC2 实例存储

[Amazon EC2 实例存储](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/InstanceStorage.html)为您的 EC2 实例提供临时块级存储。EC2 实例存储提供的存储通过物理连接到主机的磁盘访问。与 Amazon EBS 不同,您只能在启动实例时附加实例存储卷,这些卷仅在实例的生命周期内存在。它们无法与其他实例分离和重新附加。您可以在[此处](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/InstanceStorage.html)了解更多关于 Amazon EC2 实例存储的信息。*实例存储卷没有任何额外费用。*这使得它们(实例存储卷)比带有大型 EBS 卷的常规 EC2 实例更具成本效益。

在 Kubernetes 中使用本地存储卷，您应该对磁盘进行分区、配置和格式化[使用 Amazon EC2 用户数据](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-add-user-data.html)，以便将卷挂载为 pod 规范中的[HostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath)。或者，您可以利用[本地持久卷静态供应器](https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner)来简化本地存储管理。本地持久卷静态供应器允许您通过标准的 Kubernetes PersistentVolumeClaim (PVC) 接口访问本地实例存储卷。此外，它将提供包含节点亲和性信息的 PersistentVolumes (PVs)，以将 Pod 调度到正确的节点。尽管它使用 Kubernetes PersistentVolumes，但 EC2 实例存储卷本质上是临时的。写入临时磁盘的数据只在实例的生命周期内可用。当实例终止时，数据也会消失。有关更多详细信息，请参阅[此博客](https://aws.amazon.com/blogs/containers/eks-persistent-volumes-for-instance-store/)。

请记住，在使用 Amazon EC2 实例存储卷时，总 IOPS 限制与主机共享，并将 Pod 绑定到特定主机。在采用 Amazon EC2 实例存储卷之前，您应该仔细审查您的工作负载要求。

## 持久卷

Kubernetes 通常与运行无状态应用程序相关联。但是,也有一些场景需要运行需要保留从一个请求到下一个请求的持久数据或信息的微服务。数据库是这种用例的一个常见示例。但是,Pod 及其内部的容器或进程都是临时性的。为了在 Pod 生命周期之外持久化数据,您可以使用 PV 来定义对特定位置存储的访问,该位置独立于 Pod。*PV 的成本高度依赖于所使用的存储类型以及应用程序的使用方式。*

在 Amazon EKS 上支持 Kubernetes PV 的不同存储选项列在[此处](https://docs.aws.amazon.com/eks/latest/userguide/storage.html)。下面介绍的存储选项包括 Amazon EBS、Amazon EFS、Amazon FSx for Lustre 和 Amazon FSx for NetApp ONTAP。

### Amazon Elastic Block Store (EBS) 卷

亚马逊 EBS 卷可以作为 Kubernetes PV 使用,提供块级存储卷。这些非常适合依赖随机读写和吞吐量密集型应用程序的数据库,这些应用程序执行长时间的连续读写。[亚马逊弹性块存储容器存储接口 (CSI) 驱动程序](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html)允许亚马逊 EKS 集群管理亚马逊 EBS 卷的生命周期以用作持久卷。容器存储接口使 Kubernetes 与存储系统之间的交互成为可能和便利。当在您的 EKS 集群中部署 CSI 驱动程序时,您可以通过原生 Kubernetes 存储资源(如持久卷 (PV)、持久卷声明 (PVC) 和存储类 (SC))访问其功能。[此链接](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/tree/master/examples/kubernetes)提供了如何使用亚马逊 EBS CSI 驱动程序与亚马逊 EBS 卷交互的实际示例。

#### 选择合适的卷

*我们建议使用最新一代的块存储 (gp3),因为它在价格和性能之间提供了合适的平衡*。它还允许您独立于卷大小扩展卷 IOPS 和吞吐量,无需预置额外的块存储容量。如果您目前正在使用 gp2 卷,我们强烈建议您迁移到 gp3 卷。[此博客](https://aws.amazon.com/blogs/containers/migrating-amazon-eks-clusters-from-gp2-to-gp3-ebs-volumes/)解释了如何在亚马逊 EKS 集群上从 *gp2* 迁移到 *gp3*。

当您有需要更高性能且需要超过单个 [gp3 卷支持的卷大小](https://aws.amazon.com/ebs/general-purpose/)的应用程序时,您应该考虑使用 [io2 块快车](https://aws.amazon.com/ebs/provisioned-iops/)。这种存储类型非常适合您最大、最 I/O 密集和关键任务部署,如 SAP HANA 或其他具有低延迟要求的大型数据库。请记住,实例的 EBS 性能受实例性能限制的约束,因此并非所有实例都支持 io2 块快车卷。您可以在[此文档](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/provisioned-iops.html)中查看支持的实例类型和其他注意事项。

*单个 gp3 卷最多可支持 16,000 个最大 IOPS、1,000 MiB/s 最大吞吐量和 16TiB 最大容量。最新一代的配置 IOPS SSD 卷最多可提供 256,000 IOPS、4,000 MiB/s 吞吐量和 64TiB 容量。*

在这些选项中,您应该根据应用程序的需求最佳匹配存储性能和成本。

#### 随时监控和优化

了解应用程序的基线性能并监控所选卷是很重要的,以检查是否满足您的要求/期望,或者是否过度配置(例如,配置的 IOPS 未被充分利用)。

在一开始就分配大量存储空间的做法不如逐步增加存储空间。您可以使用亚马逊弹性块存储 CSI 驱动程序 (aws-ebs-csi-driver) 中的[卷调整](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/tree/master/examples/kubernetes/resizing)功能动态调整卷大小。*请注意,您只能增加 EBS 卷大小。*

要识别和删除任何悬挂的 EBS 卷,您可以使用 [AWS 可信顾问的成本优化类别](https://docs.aws.amazon.com/awssupport/latest/user/cost-optimization-checks.html)。此功能可帮助您识别未连接的卷或在一段时间内写入活动非常低的卷。还有一个名为 [Popeye](https://github.com/derailed/popeye) 的云原生开源只读工具,可扫描实时 Kubernetes 集群并报告已部署资源和配置的潜在问题。例如,它可以扫描未使用的 PV 和 PVC,并检查它们是否已绑定或是否存在任何卷挂载错误。

有关深入监控的信息,请参考 [EKS 成本优化可观察性指南](https://aws.github.io/aws-eks-best-practices/cost_optimization/cost_opt_observability/)。

您可以考虑的另一个选择是 [AWS Compute Optimizer Amazon EBS 卷建议](https://docs.aws.amazon.com/compute-optimizer/latest/ug/view-ebs-recommendations.html)。此工具可自动识别所需的最佳卷配置和性能级别。例如,它可用于根据过去 14 天的最大利用率优化配置,如预置 IOPS、卷大小和 EBS 卷类型。它还量化了其建议带来的潜在每月成本节省。您可以查看此[博客](https://aws.amazon.com/blogs/storage/cost-optimizing-amazon-ebs-volumes-using-aws-compute-optimizer/)了解更多详细信息。

#### 备份保留策略

您可以通过创建时间点快照来备份 Amazon EBS 卷上的数据。Amazon EBS CSI 驱动程序支持卷快照。您可以按照[此处](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/master/examples/kubernetes/snapshot/README.md)概述的步骤了解如何创建快照和恢复 EBS PV。

后续的快照是增量备份,这意味着只有在最近一次快照后设备上发生变化的块才会被保存。这最大限度地减少了创建快照所需的时间,并通过避免重复数据来节省存储成本。但是,如果没有适当的保留策略,旧的 EBS 快照数量的增加会在大规模运营时造成意外成本。如果您通过 AWS API 直接备份 Amazon EBS 卷,可以利用 [Amazon Data Lifecycle Manager](https://aws.amazon.com/ebs/data-lifecycle-manager/) (DLM),它提供了一个自动化、基于策略的生命周期管理解决方案,用于管理 Amazon Elastic Block Store (EBS) 快照和基于 EBS 的 Amazon Machine Images (AMIs)。控制台使得自动化创建、保留和删除 EBS 快照和 AMIs 变得更加容易。

!!! note
    目前还没有办法通过 Amazon EBS CSI 驱动程序利用 Amazon DLM。

在 Kubernetes 环境中,您可以利用一个名为 [Velero](https://velero.io/) 的开源工具来备份您的 EBS 持久卷。您可以在安排作业时设置 TTL 标志来过期备份。这里有一个来自 Velero 的[指南](https://velero.io/docs/v1.12/how-velero-works/#set-a-backup-to-expire)作为示例。

### Amazon Elastic File System (EFS)

[Amazon Elastic File System (EFS)](https://aws.amazon.com/efs/) 是一个无服务器、完全弹性的文件系统,它允许您使用标准的文件系统接口和文件系统语义共享文件数据,适用于广泛的工作负载和应用程序。工作负载和应用程序的示例包括 Wordpress 和 Drupal、JIRA 和 Git 等开发工具,以及 Jupyter 等共享笔记本系统以及家目录。

Amazon EFS 的主要优势之一是它可以被多个节点和多个可用区域中的多个容器挂载。另一个优势是您只需为使用的存储付费。EFS 文件系统将自动根据您添加和删除文件而增长和收缩,从而消除了容量规划的需要。

要在 Kubernetes 中使用 Amazon EFS,您需要使用 Amazon Elastic File System Container Storage Interface (CSI) 驱动程序 [aws-efs-csi-driver](https://github.com/kubernetes-sigs/aws-efs-csi-driver)。目前,该驱动程序可以动态创建[访问点](https://docs.aws.amazon.com/efs/latest/ug/efs-access-points.html)。但是,必须先配置 Amazon EFS 文件系统,并将其作为输入提供给 Kubernetes 存储类参数。

#### 选择合适的 EFS 存储类

Amazon EFS 提供了[四种存储类](https://docs.aws.amazon.com/efs/latest/ug/storage-classes.html)。

两种标准存储类:

* Amazon EFS Standard
* [Amazon EFS Standard-Infrequent Access](https://aws.amazon.com/blogs/aws/optimize-storage-cost-with-reduced-pricing-for-amazon-efs-infrequent-access/) (EFS Standard-IA)

两种单区域存储类:

[亚马逊 EFS 单区域]
亚马逊 EFS 单区域-不频繁访问 (EFS 单区域-IA)

不频繁访问 (IA) 存储类针对的是不是每天都会访问的文件。通过亚马逊 EFS 生命周期管理,您可以将在生命周期策略期限内(7、14、30、60 或 90 天)未被访问的文件移动到 IA 存储类,*这可以将存储成本降低高达 92% 与 EFS 标准和 EFS 单区域存储类相比*。

通过 EFS 智能分层,生命周期管理会监控您文件系统的访问模式,并自动将文件移动到最优的存储类。

!!! note 
    aws-efs-csi-driver 目前没有控制更改存储类、生命周期管理或智能分层的功能。这些应该在 AWS 控制台或通过 EFS API 手动设置。

!!! note
    aws-efs-csi-driver 与基于 Windows 的容器映像不兼容。

!!! note
    当启用 *vol-metrics-opt-in* (发出卷指标)时,由于 [DiskUsage](https://github.com/kubernetes/kubernetes/blob/ee265c92fec40cd69d1de010b477717e4c142492/pkg/volume/util/fs/fs.go#L66) 函数会消耗与文件系统大小成比例的内存量,从而存在已知的内存问题。*目前,我们建议在大型文件系统上禁用 `--vol-metrics-opt-in` 选项,以避免消耗过多内存。有关更多详细信息,请查看 github 问题 [link](https://github.com/kubernetes-sigs/aws-efs-csi-driver/issues/1104)。*

### 亚马逊 FSx for Lustre

Lustre 是一种高性能并行文件系统,通常用于需要高达数百 GB/s 的吞吐量和亚毫秒级的单个操作延迟的工作负载。它用于机器学习训练、金融建模、高性能计算和视频处理等场景。[亚马逊 FSx for Lustre](https://aws.amazon.com/fsx/lustre/) 提供了一个完全托管的共享存储,具有可扩展性和性能,并与亚马逊 S3 无缝集成。

您可以使用 [FSx for Lustre CSI 驱动程序](https://github.com/kubernetes-sigs/aws-fsx-csi-driver)在 Amazon EKS 或您在 AWS 上自管理的 Kubernetes 集群上,通过 Kubernetes 持久存储卷使用 FSx for Lustre。请参阅 [Amazon EKS 文档](https://docs.aws.amazon.com/eks/latest/userguide/fsx-csi.html)了解更多详细信息和示例。

#### 与亚马逊 S3 的链接

建议将位于亚马逊 S3 上的高耐用性长期数据存储库与您的 FSx for Lustre 文件系统链接。链接后,大型数据集会根据需要从亚马逊 S3 延迟加载到 FSx for Lustre 文件系统。您还可以将分析结果写回 S3,然后删除您的 [Lustre] 文件系统。

#### 选择合适的部署和存储选项

FSx for Lustre提供了不同的部署选项。第一个选项称为*scratch*,它不会复制数据,而第二个选项称为*persistent*,顾名思义,它会持久保存数据。

第一个选项(*scratch*)可用于*降低临时短期数据处理的成本。*持久部署选项_旨在用于长期存储_,它会自动在AWS可用区内复制数据。它还支持SSD和HDD存储。

您可以在FSx for lustre文件系统的Kubernetes StorageClass的参数中配置所需的部署类型。这里有一个[链接](https://github.com/kubernetes-sigs/aws-fsx-csi-driver/tree/master/examples/kubernetes/dynamic_provisioning#edit-storageclass)提供了示例模板。

!!! note
    对于对延迟敏感或需要最高IOPS/吞吐量的工作负载,您应该选择SSD存储。对于不太关注延迟但注重吞吐量的工作负载,您应该选择HDD存储。


#### 启用数据压缩

您还可以通过将"LZ4"指定为数据压缩类型来启用文件系统的数据压缩。启用后,所有新写入的文件在写入磁盘之前都会自动在FSx for Lustre上进行压缩,读取时会自动解压缩。LZ4数据压缩算法是无损的,因此可以完全从压缩数据中重建原始数据。

您可以在FSx for lustre文件系统的Kubernetes StorageClass的参数中配置数据压缩类型为LZ4。当值设置为NONE(默认值)时,压缩功能将被禁用。这个[链接](https://github.com/kubernetes-sigs/aws-fsx-csi-driver/tree/master/examples/kubernetes/dynamic_provisioning#edit-storageclass)提供了示例模板。

!!! note
    Amazon FSx for Lustre与基于Windows的容器映像不兼容。


### Amazon FSx for NetApp ONTAP

[Amazon FSx for NetApp ONTAP](https://aws.amazon.com/fsx/netapp-ontap/)是一种完全托管的共享存储,它基于NetApp的ONTAP文件系统构建。FSx for ONTAP提供功能丰富、快速和灵活的共享文件存储,可广泛从在AWS或本地运行的Linux、Windows和macOS计算实例访问。

Amazon FSx for NetApp ONTAP支持两层存储:*1/主存储层*和*2/容量池层。*

主要层是一个配置的、高性能的基于SSD的层,用于活跃的、对延迟敏感的数据。完全弹性的容量池层针对不频繁访问的数据进行成本优化,随着数据分层到其中而自动扩展,并提供几乎无限的PB级容量。您可以在容量池存储上启用数据压缩和重复数据删除,进一步减少数据占用的存储容量。NetApp的本地、基于策略的FabricPool功能持续监控数据访问模式,自动双向地在存储层之间传输数据,以优化性能和成本。

NetApp的Astra Trident提供了使用CSI驱动程序的动态存储编排,允许Amazon EKS集群管理由Amazon FSx for NetApp ONTAP文件系统支持的持久卷(PV)的生命周期。要开始使用,请参见Astra Trident文档中的[在Amazon FSx for NetApp ONTAP上使用Astra Trident](https://docs.netapp.com/us-en/trident/trident-use/trident-fsx.html)。

## 其他注意事项

### 最小化容器镜像的大小

一旦容器部署,容器镜像就会被缓存在主机上的多个层中。通过减小镜像大小,可以减少主机上所需的存储空间。

通过使用精简的基础镜像,如[scratch](https://hub.docker.com/_/scratch)镜像或[distroless](https://github.com/GoogleContainerTools/distroless)容器镜像(仅包含您的应用程序及其运行时依赖项),从一开始就可以*减少存储成本,并获得其他附带好处,如减少攻击面和缩短镜像拉取时间*。

您还应该考虑使用开源工具,如[Slim.ai](https://www.slim.ai/docs/quickstart),它提供了一种简单、安全的方式来创建最小化的镜像。

多层的软件包、工具、应用程序依赖项和库很容易使容器镜像膨胀。通过使用多阶段构建,您可以选择性地从一个阶段复制制品到另一个阶段,从最终镜像中排除所有不必要的内容。您可以在[这里](https://docs.docker.com/get-started/09_image_best/)查看更多镜像构建最佳实践。

另一个需要考虑的是缓存镜像的保留时间。当磁盘使用率达到一定程度时,您可能需要清理缓存中的过时镜像。这样做可以确保为主机操作保留足够的空间。默认情况下,[kubelet](https://kubernetes.io/docs/reference/generated/kubelet)每5分钟对未使用的镜像进行一次垃圾回收,每分钟对未使用的容器进行一次垃圾回收。
为了配置未使用的容器和镜像垃圾收集的选项,请使用[配置文件](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-config-file/)调整kubelet,并使用[`KubeletConfiguration`](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)资源类型更改与垃圾收集相关的参数。

您可以在Kubernetes[文档](https://kubernetes.io/docs/concepts/architecture/garbage-collection/#containers-images)中了解更多相关信息。
