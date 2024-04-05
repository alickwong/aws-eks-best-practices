!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

# EKS 数据平面

为了运行高可用和有弹性的应用程序,您需要一个高可用和有弹性的数据平面。弹性数据平面确保 Kubernetes 可以自动扩展和修复您的应用程序。弹性数据平面由两个或更多工作节点组成,可以根据工作负载的变化而增长和收缩,并能自动从故障中恢复。

您可以选择使用 [EC2 实例](https://docs.aws.amazon.com/eks/latest/userguide/worker.html)或 [Fargate](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html)作为 EKS 的工作节点。如果选择 EC2 实例,您可以自己管理工作节点,也可以使用 [EKS 托管节点组](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html)。您可以拥有一个由托管节点、自管理节点和 Fargate 组成的集群。

EKS on Fargate 提供了最简单的弹性数据平面方案。Fargate 在隔离的计算环境中运行每个 Pod。在 Fargate 上运行的每个 Pod 都有自己的工作节点。Fargate 会根据 Kubernetes 扩展 Pod 的情况自动扩展数据平面。您可以使用[水平 Pod 自动扩缩器](https://docs.aws.amazon.com/eks/latest/userguide/horizontal-pod-autoscaler.html)来扩展数据平面和您的工作负载。

扩展 EC2 工作节点的首选方式是使用 [Kubernetes 集群自动缩放器](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md)、[EC2 Auto Scaling 组](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html)或社区项目如 [Atlassian's Escalator](https://github.com/atlassian/escalator)。

## 建议

### 使用 EC2 Auto Scaling 组创建工作节点

使用 EC2 Auto Scaling 组创建工作节点是最佳实践,而不是创建单个 EC2 实例并将其加入集群。Auto Scaling 组将自动替换任何终止或失败的节点,确保集群始终有足够的容量来运行您的工作负载。

### 使用 Kubernetes 集群自动缩放器扩展节点

集群自动缩放器会在集群资源不足以运行所有 Pod 时调整数据平面的大小,并添加新的工作节点。尽管集群自动缩放器是一个被动的过程,但它会等待 Pod 进入 *Pending* 状态,因为集群容量不足。当出现这种情况时,它会向集群添加 EC2 实例。每当集群耗尽容量时,新的副本或新的 Pod 将处于 *Pending* 状态,直到添加新的工作节点。如果数据平面无法足够快地扩展以满足工作负载的需求,这种延迟可能会影响应用程序的可靠性。如果一个工作节点一直处于低利用率,并且其所有 Pod 都可以调度到其他工作节点上,集群自动缩放器会终止该节点。

### 配置集群自动缩放器的过度配置

集群自动缩放器在集群中的 Pod 已经处于 *Pending* 状态时触发数据平面的扩容。因此,从您的应用程序需要更多副本到实际获得更多副本之间可能会有延迟。一种应对这种可能延迟的方法是增加所需副本数量,即为应用程序增加副本数量。

集群自动缩放器推荐的另一种模式使用 [*暂停* Pod 和优先级抢占特性](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md#how-can-i-configure-overprovisioning-with-cluster-autoscaler)。*暂停 Pod* 运行一个 [暂停容器](https://github.com/kubernetes/kubernetes/tree/master/build/pause),顾名思义,它什么也不做,只是作为集群中其他 Pod 可以使用的计算容量的占位符。由于它被分配了 *非常低的优先级*,当集群没有可用容量时,暂停 Pod 会被从节点上驱逐,Kubernetes 调度器会注意到暂停 Pod 的驱逐并尝试重新调度它。但由于集群已经达到容量上限,暂停 Pod 将保持 *Pending* 状态,这将触发集群自动缩放器添加新节点。

可以使用 Helm 图表安装 [集群过度配置](https://github.com/helm/charts/tree/master/stable/cluster-overprovisioner)。

### 将集群自动缩放器与多个自动缩放组一起使用

运行集群自动缩放器时,启用 `--node-group-auto-discovery` 标志。这样做将允许集群自动缩放器找到包含特定定义标签的所有自动缩放组,并避免在清单中定义和维护每个自动缩放组。

### 将集群自动缩放器与本地存储一起使用

默认情况下,集群自动缩放器不会缩减部署有本地存储的节点。将 `--skip-nodes-with-local-storage` 标志设置为 false,以允许集群自动缩放器缩减这些节点。

### 跨多个 AZ 分散工作节点和工作负载

您可以通过在多个 AZ 中运行工作节点和 Pod 来保护您的工作负载免受单个 AZ 故障的影响。您可以使用创建节点的子网来控制工作节点创建的 AZ。

如果您使用的是 Kubernetes 1.18 及更高版本,推荐的跨 AZ 分散 Pod 的方法是使用 [Pod 拓扑分散约束](https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/#spread-constraints-for-pods)。

下面的部署尽可能将 Pod 分散到不同的 AZ 中,如果无法做到,也允许这些 Pod 继续运行:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-server
  template:
    metadata:
      labels:
        app: web-server
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          whenUnsatisfiable: ScheduleAnyway
          topologyKey: topology.kubernetes.io/zone
          labelSelector:
            matchLabels:
              app: web-server
      containers:
      - name: web-app
        image: nginx
        resources:
          requests:
            cpu: 1
```

您的翻译是:

!!! note
    `kube-scheduler` 仅通过具有这些标签的节点了解拓扑域。 如果上述部署部署到仅在单个区域中具有节点的集群中，则所有 pod 都将在这些节点上调度，因为 `kube-scheduler` 不知道其他区域。 为了使此拓扑传播按预期与调度程序一起工作,必须已经在所有区域中存在节点。 在 Kubernetes 1.24 中添加 `MinDomainsInPodToplogySpread` [功能门](https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/#api) 将解决此问题,该功能允许指定 `minDomains` 属性来告知调度程序合格域的数量。

!!! warning
    将 `whenUnsatisfiable` 设置为 `DoNotSchedule` 将导致 pod 在无法满足拓扑传播约束时无法调度。 只有在更喜欢 pod 不运行而不是违反拓扑传播约束时,才应该设置它。

在较旧版本的 Kubernetes 中,您可以使用 pod 反亲和性规则在多个 AZ 中调度 pod。 下面的清单告知 Kubernetes 调度程序*首选*在不同 AZ 中调度 pod。
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
  labels:
    app: web-server
spec:
  replicas: 4
  selector:
    matchLabels:
      app: web-server
  template:
    metadata:
      labels:
        app: web-server
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - web-server
              topologyKey: failure-domain.beta.kubernetes.io/zone
            weight: 100
      containers:
      - name: web-app
        image: nginx
```


不要要求 pods 跨不同的可用区进行调度,否则,部署中的 pods 数量永远不会超过可用区的数量。

### 在使用 EBS 卷时确保每个可用区的容量

如果您使用[Amazon EBS 提供持久卷](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html),那么您需要确保 pods 和相关的 EBS 卷位于同一可用区。在撰写本文时,EBS 卷仅在单个可用区内可用。Pod 无法访问位于不同可用区的 EBS 支持的持久卷。Kubernetes [调度程序知道工作节点](https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#topologykubernetesiozone)所在的可用区。Kubernetes 将始终在与卷所在相同的可用区中调度需要 EBS 卷的 Pod。但是,如果在卷所在的可用区内没有可用的工作节点,则无法调度 Pod。

为每个可用区创建自动扩展组,并确保集群始终有足够的容量来在与 EBS 卷相同的可用区中调度 pods。此外,您应该在集群自动缩放器中启用 `--balance-similar-node-groups` 功能。

如果您正在运行使用 EBS 卷的应用程序,但没有高可用性要求,那么您可以将应用程序的部署限制在单个可用区。在 EKS 中,工作节点会自动添加 `failure-domain.beta.kubernetes.io/zone` 标签,其中包含可用区的名称。您可以通过运行 `kubectl get nodes --show-labels` 查看附加到节点的标签。有关内置节点标签的更多信息,请参阅[此处](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#built-in-node-labels)。您可以使用节点选择器将 pod 调度到特定的可用区。

在下面的示例中,pod 将仅在 `us-west-2c` 可用区中调度:
```
apiVersion: v1
kind: Pod
metadata:
  name: single-az-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: failure-domain.beta.kubernetes.io/zone
            operator: In
            values:
            - us-west-2c
  containers:
  - name: single-az-container
    image: kubernetes/pause
```


持久卷（由 EBS 支持）也会自动标记 AZ 的名称；您可以通过运行 `kubectl get pv -L topology.ebs.csi.aws.com/zone` 来查看您的持久卷属于哪个 AZ。当创建 Pod 并声明一个卷时，Kubernetes 会将 Pod 调度到与该卷相同 AZ 的节点上。

考虑以下场景：您有一个 EKS 集群,其中有一个节点组。该节点组有三个工作节点,分布在三个 AZ 中。您有一个使用 EBS 支持的持久卷的应用程序。当您创建此应用程序和相应的卷时,其 Pod 会在三个 AZ 中的第一个 AZ 中创建。然后,运行此 Pod 的工作节点变得不健康并最终无法使用。集群自动缩放器将用新的工作节点替换不健康的节点;但是,由于自动缩放组跨越三个 AZ,新的工作节点可能会在第二个或第三个 AZ 中启动,而不是在第一个 AZ 中,这并不符合需求。由于 AZ 受限的 EBS 卷只存在于第一个 AZ 中,但该 AZ 没有可用的工作节点,因此无法调度 Pod。因此,您应该在每个 AZ 中创建一个节点组,以确保始终有足够的容量来运行无法在其他 AZ 中调度的 Pod。

另外,[EFS](https://github.com/kubernetes-sigs/aws-efs-csi-driver)可以简化运行需要持久存储的应用程序的集群自动缩放。客户端可以从该区域的所有 AZ 并发访问 EFS 文件系统。即使使用 EFS 支持的持久卷的 Pod 被终止并在不同的 AZ 中重新调度,它也能够挂载该卷。

### 运行节点问题检测器

工作节点的故障可能会影响应用程序的可用性。[节点问题检测器](https://github.com/kubernetes/node-problem-detector)是一个 Kubernetes 插件,您可以在集群中安装它来检测工作节点问题。您可以使用 [npd 的补救系统](https://github.com/kubernetes/node-problem-detector#remedy-systems)自动排空和终止节点。

### 为系统和 Kubernetes 守护进程保留资源

您可以通过[为操作系统和 Kubernetes 守护进程保留计算容量](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/)来提高工作节点的稳定性。 Pod - 特别是那些没有声明 `limits` 的 Pod - 可能会占用系统资源,使节点陷入操作系统进程和 Kubernetes 守护进程(`kubelet`、容器运行时等)与 Pod 争夺系统资源的困境。您可以使用 `kubelet` 标志 `--system-reserved` 和 `--kube-reserved` 分别为系统进程(`udev`、`sshd`等)和 Kubernetes 守护进程保留资源。

如果您使用 [EKS 优化的 Linux AMI](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html)，CPU、内存和存储空间默认情况下会为系统和 Kubernetes 守护进程保留。当基于此 AMI 启动工作节点时，EC2 用户数据会被配置为触发 [`bootstrap.sh` 脚本](https://github.com/awslabs/amazon-eks-ami/blob/master/files/bootstrap.sh)。该脚本根据 EC2 实例上可用的 CPU 内核数量和总内存计算 CPU 和内存预留。计算出的值将写入位于 `/etc/kubernetes/kubelet/kubelet-config.json` 的 `KubeletConfiguration` 文件。

如果您在节点上运行自定义守护进程,并且默认预留的 CPU 和内存量不足,您可能需要增加系统资源预留。

`eksctl` 提供了最简单的方式来自定义[系统和 Kubernetes 守护进程的资源预留](https://eksctl.io/usage/customizing-the-kubelet/)。

### 实施 QoS

对于关键应用程序,请考虑为 Pod 中的容器定义 `requests`=`limits`。这将确保即使其他 Pod 请求资源,该容器也不会被终止。

为所有容器实施 CPU 和内存限制是最佳实践,因为它可以防止容器无意中消耗系统资源,从而影响其他共同位置进程的可用性。

### 为所有工作负载配置和调整资源请求/限制

可以应用一些一般性指导来调整资源请求和限制的大小:

- 不要在 CPU 上指定资源限制。在没有限制的情况下,请求充当[容器获得相对 CPU 时间的权重](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#how-pods-with-resource-limits-are-run)。这允许您的工作负载使用全部 CPU,而不会受到人为限制或饥饿。

- 对于非 CPU 资源,配置 `requests`=`limits` 提供最可预测的行为。如果 `requests`!=`limits`,容器的 [QOS](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/#qos-classes) 也会从 Guaranteed 降低到 Burstable,这使其在[节点压力](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)事件中更有可能被驱逐。

- 对于非 CPU 资源,不要指定远大于请求的限制。`limits` 相对于 `requests` 配置得越大,节点被过度承诺的可能性就越大,从而导致工作负载中断的可能性更高。

- 在使用像 [Karpenter](https://aws.github.io/aws-eks-best-practices/karpenter/) 或 [Cluster AutoScaler](https://aws.github.io/aws-eks-best-practices/cluster-autoscaling/) 这样的节点自动扩缩解决方案时,正确设置资源请求尤为重要。这些工具会根据您的工作负载请求来确定要配置的节点数量和大小。如果您的请求过小但限制较大,您可能会发现您的工作负载被驱逐或内存溢出,因为它们被紧密地打包在一个节点上。

确定资源请求可能很困难,但像 [Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) 这样的工具可以帮助您通过观察容器在运行时的资源使用情况来"调整"请求大小。其他可能有助于确定请求大小的工具包括:

- [Goldilocks](https://github.com/FairwindsOps/goldilocks)
- [Parca](https://www.parca.dev/)
- [Prodfiler](https://prodfiler.com/)
- [rsg](https://mhausenblas.info/right-size-guide/)


### 为命名空间配置资源配额

命名空间旨在用于拥有多个用户分布在多个团队或项目的环境中。它们为名称提供了作用域,是在多个团队、项目和工作负载之间划分集群资源的一种方式。您可以限制命名空间中的总体资源消耗。[`ResourceQuota`](https://kubernetes.io/docs/concepts/policy/resource-quotas/) 对象可以限制在该命名空间中可以创建的对象的数量,以及该项目中的资源可以消耗的总计算资源。您可以限制在给定命名空间中可以请求的存储和/或计算(CPU 和内存)资源的总和。

> 如果为命名空间启用了资源配额,用户必须为该命名空间中的每个容器指定请求或限制。

考虑为每个命名空间配置配额。考虑使用 `LimitRanges` 自动将预配置的限制应用于命名空间内的容器。

### 限制命名空间内容器资源使用

资源配额有助于限制命名空间可以使用的资源量。[`LimitRange` 对象](https://kubernetes.io/docs/concepts/policy/limit-range/)可以帮助您实施容器可以请求的最小和最大资源。使用 `LimitRange`，您可以为容器设置默认请求和限制,如果在您的组织中设置计算资源限制不是标准做法,这将很有帮助。顾名思义,`LimitRange` 可以强制执行命名空间中每个 Pod 或容器的最小和最大计算资源使用量。同时,它还可以强制执行命名空间中每个 PersistentVolumeClaim 的最小和最大存储请求。
[考虑结合使用 `LimitRange` 和 `ResourceQuota` 来执行容器和命名空间级别的限制。设置这些限制将确保容器或命名空间不会占用集群中其他租户使用的资源。

## CoreDNS

CoreDNS 在 Kubernetes 中提供名称解析和服务发现功能。它默认安装在 EKS 集群中。为了实现互操作性,CoreDNS 的 Kubernetes 服务仍然命名为 [kube-dns](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/)。CoreDNS Pod 作为 `kube-system` 命名空间中的 Deployment 的一部分运行,在 EKS 中,默认情况下它运行两个副本,并声明了请求和限制。DNS 查询被发送到 `kube-system` 命名空间中运行的 `kube-dns` 服务。

## 建议
### 监控 CoreDNS 指标
CoreDNS 内置了 [Prometheus](https://github.com/coredns/coredns/tree/master/plugin/metrics) 支持。您应该特别考虑监控 CoreDNS 延迟 (`coredns_dns_request_duration_seconds_sum`, 在 [1.7.0](https://github.com/coredns/coredns/blob/master/notes/coredns-1.7.0.md) 版本之前,该指标被称为 `core_dns_response_rcode_count_total`)、错误 (`coredns_dns_responses_total`, NXDOMAIN, SERVFAIL, FormErr) 和 CoreDNS Pod 的内存消耗。

出于故障排查目的,您可以使用 kubectl 查看 CoreDNS 日志:

]
```shell
for p in $(kubectl get pods -n kube-system -l k8s-app=kube-dns -o jsonpath='{.items[*].metadata.name}'); do kubectl logs $p -n kube-system; done
```


### 使用 NodeLocal DNSCache
您可以通过运行 [NodeLocal DNSCache](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/) 来提高集群 DNS 性能。此功能以 DaemonSet 的形式在集群节点上运行 DNS 缓存代理。所有 pod 都使用节点上运行的 DNS 缓存代理进行名称解析,而不是使用 `kube-dns` 服务。

### 为 CoreDNS 配置 cluster-proportional-scaler

提高集群 DNS 性能的另一种方法是[根据集群中的节点数和 CPU 核心数自动水平扩展 CoreDNS 部署](https://kubernetes.io/docs/tasks/administer-cluster/dns-horizontal-autoscaling/#enablng-dns-horizontal-autoscaling)。[水平集群比例自动缩放器](https://github.com/kubernetes-sigs/cluster-proportional-autoscaler/blob/master/README.md)是一个容器,它根据可调度数据平面的大小调整部署的副本数量。

节点和节点中 CPU 核心的总和是您可以用来扩展 CoreDNS 的两个指标。您可以同时使用这两个指标。如果您使用较大的节点,CoreDNS 缩放将基于 CPU 核心数。而如果您使用较小的节点,CoreDNS 副本的数量将取决于数据平面中的 CPU 核心数。比例自动缩放器配置如下所示:
```
linear: '{"coresPerReplica":256,"min":1,"nodesPerReplica":16}'
```

### 选择具有节点组的 AMI
EKS 提供了经过优化的 EC2 AMI，客户可以使用这些 AMI 来创建自管理和托管的节点组。这些 AMI 在每个区域的每个受支持的 Kubernetes 版本中都有发布。当发现任何 CVE 或错误时，EKS 会将这些 AMI 标记为已弃用。因此,在为节点组选择 AMI 时,建议不要使用已弃用的 AMI。

可以使用以下命令通过 Ec2 describe-images api 过滤已弃用的 AMI:
```
aws ec2 describe-images --image-id ami-0d551c4f633e7679c --no-include-deprecated
```

您也可以通过验证 describe-image 输出是否包含 DeprecationTime 字段来识别已弃用的 AMI。例如:
```
aws ec2 describe-images --image-id ami-xxx --no-include-deprecated
{
    "Images": [
        {
            "Architecture": "x86_64",
            "CreationDate": "2022-07-13T15:54:06.000Z",
            "ImageId": "ami-xxx",
            "ImageLocation": "123456789012/eks_xxx",
            "ImageType": "machine",
            "Public": false,
            "OwnerId": "123456789012",
            "PlatformDetails": "Linux/UNIX",
            "UsageOperation": "RunInstances",
            "State": "available",
            "BlockDeviceMappings": [
                {
                    "DeviceName": "/dev/xvda",
                    "Ebs": {
                        "DeleteOnTermination": true,
                        "SnapshotId": "snap-0993a2fc4bbf4f7f4",
                        "VolumeSize": 20,
                        "VolumeType": "gp2",
                        "Encrypted": false
                    }
                }
            ],
            "Description": "EKS Kubernetes Worker AMI with AmazonLinux2 image, (k8s: 1.19.15, docker: 20.10.13-2.amzn2, containerd: 1.4.13-3.amzn2)",
            "EnaSupport": true,
            "Hypervisor": "xen",
            "Name": "aws_eks_optimized_xxx",
            "RootDeviceName": "/dev/xvda",
            "RootDeviceType": "ebs",
            "SriovNetSupport": "simple",
            "VirtualizationType": "hvm",
            "DeprecationTime": "2023-02-09T19:41:00.000Z"
        }
    ]
}
```

