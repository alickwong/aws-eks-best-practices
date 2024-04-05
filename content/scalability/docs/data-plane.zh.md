
# Kubernetes 数据平面

Kubernetes 数据平面包括 EC2 实例、负载均衡器、存储和 Kubernetes 控制平面使用的其他 API。为了组织目的,我们将[集群服务](./cluster-services.md)分组到一个单独的页面,并将负载均衡器缩放放在[工作负载部分](./workloads.md)。本节将重点关注计算资源的缩放。

选择 EC2 实例类型可能是客户面临的最困难的决策之一,因为在具有多个工作负载的集群中没有一种通用的解决方案。以下是一些建议,可帮助您避免在缩放计算时遇到常见的陷阱。

## 自动节点自动缩放

我们建议您使用节点自动缩放,它可以减少人工操作,并与 Kubernetes 深度集成。[托管节点组](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html)和 [Karpenter](https://karpenter.sh/) 被推荐用于大规模集群。

托管节点组将为您提供 Amazon EC2 Auto Scaling 组的灵活性,并增加托管升级和配置的好处。它可以通过 [Kubernetes 集群自动缩放器](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)进行缩放,是具有各种计算需求的集群的常见选择。

Karpenter 是由 AWS 创建的开源、工作负载原生节点自动缩放器。它根据资源需求(如 GPU)和污点和容忍度(如区域分散)来缩放集群中的节点,而无需管理节点组。节点直接从 EC2 创建,避免了默认节点组配额(每组 450 个节点)的限制,并提供更大的实例选择灵活性和更少的操作开销。我们建议客户尽可能使用 Karpenter。

## 使用多种不同的 EC2 实例类型

每个 AWS 区域都有有限数量的可用实例类型。如果您创建一个只使用一种实例类型的集群,并将节点数量扩展到超出该区域容量,您将收到没有可用实例的错误。为了避免这个问题,您不应该任意限制集群中可以使用的实例类型。

Karpenter 默认将使用一组广泛的兼容实例类型,并将根据待处理工作负载的需求、可用性和成本在配置时选择实例。您可以通过 `karpenter.k8s.aws/instance-category` 键在 [NodePools](https://karpenter.sh/docs/concepts/nodepools/#instance-types) 中扩展使用的实例类型列表。

Kubernetes 集群自动缩放器要求节点组具有类似的大小,以便能够一致地进行缩放。您应该根据 CPU 和内存大小创建多个组,并独立地对它们进行缩放。使用 [ec2-instance-selector](https://github.com/aws/amazon-ec2-instance-selector) 来识别适合您节点组的类似大小的实例。
```
ec2-instance-selector --service eks --vcpus-min 8 --memory-min 16
a1.2xlarge
a1.4xlarge
a1.metal
c4.4xlarge
c4.8xlarge
c5.12xlarge
c5.18xlarge
c5.24xlarge
c5.2xlarge
c5.4xlarge
c5.9xlarge
c5.metal
```


## 选择较大的节点以减轻 API 服务器负载

在决定使用哪种实例类型时,较少的大型节点将对 Kubernetes 控制平面施加较小的负载,因为运行的 kubelet 和 DaemonSet 较少。但是,大型节点可能无法像较小的节点那样得到充分利用。应根据您的工作负载可用性和扩展需求来评估节点大小。

一个由三个 u-24tb1.metal 实例(24 TB 内存和 448 个内核)组成的集群有 3 个 kubelet,默认情况下每个节点最多可以运行 110 个 pod。如果您的 pod 每个使用 4 个内核,那么这可能是预期的(4 个内核 x 110 = 440 个内核/节点)。在一个由 3 个节点组成的集群中,处理实例事故的能力较低,因为 1 个实例故障可能会影响集群的 1/3。您应该在工作负载中指定节点要求和 pod 分布,以便 Kubernetes 调度程序可以正确地放置工作负载。

工作负载应该定义它们所需的资源和所需的可用性,通过污点、容忍和 [PodTopologySpread](https://kubernetes.io/blog/2020/05/introducing-podtopologyspread/) 来实现。它们应该优先选择可以充分利用并满足可用性目标的最大节点,以减少控制平面负载、降低运营成本和降低成本。

Kubernetes 调度程序将自动尝试在可用资源的情况下将工作负载分散到可用性区域和主机上。如果没有可用容量,Kubernetes 集群自动缩放器将尝试在每个可用性区域均匀地添加节点。Karpenter 将尝试尽快且尽可能便宜地添加节点,除非工作负载指定了其他要求。

要强制工作负载使用调度程序进行分散,并在可用性区域内创建新节点,您应该使用 topologySpreadConstraints:
```
spec:
  topologySpreadConstraints:
    - maxSkew: 3
      topologyKey: "topology.kubernetes.io/zone"
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels:
          dev: my-deployment
    - maxSkew: 2
      topologyKey: "kubernetes.io/hostname"
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels:
          dev: my-deployment
```

使用相似的节点大小以保持一致的工作负载性能

工作负载应该定义它们需要在什么大小的节点上运行,以实现一致的性能和可预测的扩展。请求500m CPU的工作负载在4核实例和16核实例上的性能会有所不同。避免使用可突发CPU的实例类型,如T系列实例。

为确保您的工作负载获得一致的性能,工作负载可以使用[支持的Karpenter标签](https://karpenter.sh/docs/concepts/scheduling/#labels)来定位特定的实例大小。
```
kind: deployment
...
spec:
  template:
    spec:
    containers:
    nodeSelector:
      karpenter.k8s.aws/instance-size: 8xlarge
```

在使用 Kubernetes 集群自动扩缩器调度工作负载的集群中，工作负载应该根据标签匹配与节点组的节点选择器相匹配。
```
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: eks.amazonaws.com/nodegroup
            operator: In
            values:
            - 8-core-node-group    # match your node group name
```


## 有效利用计算资源

计算资源包括 EC2 实例和可用区。有效利用计算资源将提高您的可扩展性、可用性、性能,并降低总成本。在具有多个应用程序的自动扩展环境中,高效利用资源极其困难。[Karpenter](https://karpenter.sh/) 被创建用于根据工作负载需求按需配置实例,以最大化利用率和灵活性。

Karpenter 允许工作负载声明所需的计算资源类型,而无需先创建节点组或为特定节点配置标签污点。请参阅 [Karpenter 最佳实践](https://aws.github.io/aws-eks-best-practices/karpenter/) 以了解更多信息。考虑在 Karpenter 配置程序中启用[合并](https://aws.github.io/aws-eks-best-practices/karpenter/#configure-requestslimits-for-all-non-cpu-resources-when-using-consolidation),以替换利用率较低的节点。

## 自动化 Amazon 机器镜像 (AMI) 更新

保持工作节点组件最新将确保您拥有最新的安全补丁和与 Kubernetes API 兼容的功能。更新 kubelet 是 Kubernetes 功能最重要的组件,但自动化操作系统、内核和本地安装的应用程序补丁将在您扩展时减少维护工作。

建议您使用最新的 [Amazon EKS 优化 Amazon Linux 2](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html) 或 [Amazon EKS 优化 Bottlerocket AMI](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami-bottlerocket.html) 作为节点镜像。Karpenter 将自动使用[最新可用的 AMI](https://karpenter.sh/docs/concepts/nodepools/#instance-types) 来配置集群中的新节点。托管节点组将在节点组更新期间更新 AMI,但在节点配置时不会更新 AMI ID。

对于托管节点组,您需要在新的 AMI ID 可用时更新自动扩展组 (ASG) 启动模板。AMI 次版本(例如从 1.23.5 到 1.24.3)将作为[节点组升级](https://docs.aws.amazon.com/eks/latest/userguide/update-managed-node-group.html)在 EKS 控制台和 API 中提供。补丁版本(例如从 1.23.5 到 1.23.6)不会作为节点组升级提供。如果您希望使用最新的 AMI 补丁版本保持节点组最新,则需要创建新的启动模板版本,并让节点组使用新的 AMI 版本替换实例。

您可以从[此页面](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html)或使用 AWS CLI 找到最新可用的 AMI。
```
aws ssm get-parameter \
  --name /aws/service/eks/optimized-ami/1.24/amazon-linux-2/recommended/image_id \
  --query "Parameter.Value" \
  --output text
```

使用多个 EBS 卷进行容器部署

EBS 卷的输入/输出 (I/O) 配额取决于卷类型(如 gp3)和磁盘大小。如果您的应用程序共享主机的单个 EBS 根卷,这可能会耗尽整个主机的磁盘配额,导致其他应用程序等待可用容量。应用程序会在写入文件到其覆盖分区、从主机挂载本地卷以及根据所使用的日志代理将日志写入标准输出 (STDOUT) 时写入磁盘。

为了避免磁盘 I/O 耗尽,您应该将第二个卷挂载到容器状态文件夹(例如 /run/containerd)、为工作负载存储使用单独的 EBS 卷,并禁用不必要的本地日志记录。

要使用 [eksctl](https://eksctl.io/) 在您的 EC2 实例上挂载第二个卷,您可以使用具有以下配置的节点组:
```
managedNodeGroups:
  - name: al2-workers
    amiFamily: AmazonLinux2
    desiredCapacity: 2
    volumeSize: 80
    additionalVolumes:
      - volumeName: '/dev/sdz'
        volumeSize: 100
    preBootstrapCommands:
    - |
      "systemctl stop containerd"
      "mkfs -t ext4 /dev/nvme1n1"
      "rm -rf /var/lib/containerd/*"
      "mount /dev/nvme1n1 /var/lib/containerd/"
      "systemctl start containerd"
```

如果您使用 Terraform 来配置节点组,请参见 [EKS Blueprints for Terraform](https://aws-ia.github.io/terraform-aws-eks-blueprints/patterns/stateful/#eks-managed-nodegroup-w-multiple-volumes) 中的示例。如果您使用 Karpenter 来配置节点,您可以使用 [`blockDeviceMappings`](https://karpenter.sh/docs/concepts/nodeclasses/#specblockdevicemappings) 和节点用户数据来添加额外的卷。

要直接将 EBS 卷挂载到您的 pod 上,您应该使用 [AWS EBS CSI 驱动程序](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)并使用存储类来使用卷。
```
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 4Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: public.ecr.aws/docker/library/nginx
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: ebs-claim
```

避免使用低 EBS 附加限制的实例，如果工作负载使用 EBS 卷

EBS 是工作负载拥有持久性存储的最简单方式之一,但它也存在可扩展性限制。每种实例类型都有最大的 [EBS 卷可附加数量](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/volume_limits.html)。工作负载需要声明它们应该在哪些实例类型上运行,并使用 Kubernetes 污点限制单个实例上的副本数量。

禁用不必要的磁盘日志记录

避免通过不在生产环境中以调试日志记录运行应用程序,以及禁用频繁读写磁盘的日志记录来避免不必要的本地日志记录。Journald 是本地日志记录服务,它将日志缓冲区保存在内存中,并定期刷新到磁盘。Journald 优于每行立即记录到磁盘的 syslog。禁用 syslog 还可以降低所需的总存储量,并避免需要复杂的日志轮换规则。要禁用 syslog,可以将以下代码段添加到您的 cloud-init 配置中:
```
runcmd:
  - [ systemctl, disable, --now, syslog.service ]
```


## 在操作系统更新速度是必要时就地修补实例

!!! 注意
    只有在必要时才应该就地修补实例。Amazon 建议将基础设施视为不可变的,并以与应用程序相同的方式彻底测试通过较低环境推广的更新。本节适用于无法做到这一点的情况。

在不中断容器化工作负载的情况下,在现有 Linux 主机上安装软件包只需几秒钟。可以安装并验证该软件包,而无需隔离、排空或替换实例。

要替换实例,首先需要创建、验证和分发新的 AMI。需要为实例创建替换,并隔离和排空旧实例。然后需要在新实例上创建工作负载,进行验证,并对所有需要修补的实例重复此过程。在不中断工作负载的情况下安全地替换实例需要数小时、数天或数周的时间。

Amazon 建议使用由自动化、声明式系统构建、测试和推广的不可变基础设施,但如果您需要快速修补系统,则需要就地修补系统并在新 AMI 可用时替换它们。由于修补和替换系统之间存在巨大的时间差异,我们建议使用 [AWS Systems Manager Patch Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-patch.html) 来自动修补节点。

修补节点将允许您快速推出安全更新,并在 AMI 更新后定期替换实例。如果您使用的操作系统具有只读根文件系统,如 [Flatcar Container Linux](https://flatcar-linux.org/) 或 [Bottlerocket OS](https://github.com/bottlerocket-os/bottlerocket),我们建议使用适用于这些操作系统的更新操作员。[Flatcar Linux 更新操作员](https://github.com/flatcar/flatcar-linux-update-operator)和 [Bottlerocket 更新操作员](https://github.com/bottlerocket-os/bottlerocket-update-operator)将自动重启实例以保持节点最新。
