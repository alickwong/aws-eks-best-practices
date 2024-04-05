!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

# Kubernetes 集群自动缩放器

<iframe width="560" height="315" src="https://www.youtube.com/embed/FIBc8GkjFU0" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

## 概述

[Kubernetes 集群自动缩放器](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)是由 [SIG Autoscaling](https://github.com/kubernetes/community/tree/master/sig-autoscaling) 维护的一种流行的集群自动缩放解决方案。它负责确保您的集群有足够的节点来调度您的 pod,而不会浪费资源。它会监视无法调度的 pod 和利用不足的节点。然后它模拟添加或删除节点,然后再将更改应用到您的集群。集群自动缩放器中的 AWS 云提供商实现控制您的 EC2 Auto Scaling 组的 `.DesiredReplicas` 字段。

![](./architecture.png)

本指南将提供一个配置集群自动缩放器的心智模型,并选择最佳的权衡方案以满足您组织的需求。虽然没有单一最佳配置,但有一组配置选项可以让您权衡性能、可扩展性、成本和可用性。此外,本指南还将提供针对 AWS 的优化配置的技巧和最佳实践。

### 术语表

在本文档中,将频繁使用以下术语。这些术语可以有广泛的含义,但在本文档中仅限于以下定义。

**可扩展性**指集群自动缩放器在 Kubernetes 集群中的 pod 和节点数量增加时的性能。随着可扩展性限制的达到,集群自动缩放器的性能和功能会下降。当集群自动缩放器超过其可扩展性限制时,它可能无法再在集群中添加或删除节点。

**性能**指集群自动缩放器做出和执行缩放决策的速度。性能完美的集群自动缩放器将立即做出决定并触发缩放操作,以响应诸如 pod 无法调度等刺激。

**可用性**意味着 pod 可以快速调度并且不会中断。这包括当新创建的 pod 需要调度以及当缩小节点时终止分配到该节点的任何剩余 pod。

**成本**由扩展和收缩事件背后的决策决定。如果现有节点利用率低或添加了太大的新节点,资源就会浪费。根据用例的不同,由于过于激进的缩减决策而提前终止 pod 也可能会产生成本。

**节点组**是 Kubernetes 集群中节点组的抽象概念。它不是真正的 Kubernetes 资源,而是在集群自动缩放器、集群 API 和其他组件中作为抽象存在的。节点组内的节点共享标签和污点等属性,但可能由多个可用区或实例类型组成。

**EC2 Auto Scaling 组**可用作 EC2 上节点组的实现。EC2 Auto Scaling 组配置为启动自动加入其 Kubernetes 集群并将标签和污点应用于 Kubernetes API 中相应节点资源的实例。

**EC2 托管节点组**是 EC2 上节点组的另一种实现。它抽象了手动配置 EC2 自动缩放组的复杂性,并提供了节点版本升级和优雅节点终止等额外管理功能。

### 操作集群自动缩放器

集群自动缩放器通常作为[部署](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler/cloudprovider/aws/examples)安装在您的集群中。它使用[领导者选举](https://en.wikipedia.org/wiki/Leader_election)来确保高可用性,但工作由单个副本执行。它不具有水平可扩展性。对于基本设置,默认情况下应该可以使用提供的[安装说明](https://docs.aws.amazon.com/eks/latest/userguide/cluster-autoscaler.html)正常工作,但需要注意一些事项。

确保:

* 集群自动缩放器的版本与集群版本匹配。跨版本兼容性[未经测试或支持](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/README.md#releases)。
* 启用[自动发现](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler/cloudprovider/aws#auto-discovery-setup),除非您有特定的高级用例阻止使用此模式。

### 采用最小权限访问 IAM 角色

当使用[自动发现](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md#Auto-discovery-setup)时,我们强烈建议您通过将 `autoscaling:SetDesiredCapacity` 和 `autoscaling:TerminateInstanceInAutoScalingGroup` 操作限制到当前集群范围内的自动缩放组来采用最小权限访问。

这将防止在一个集群中运行的集群自动缩放器修改另一个集群中的节点组,即使 `--node-group-auto-discovery` 参数没有使用标签(例如 `k8s.io/cluster-autoscaler/<cluster-name>`)将其范围缩小到集群的节点组。
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup"
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "aws:ResourceTag/k8s.io/cluster-autoscaler/enabled": "true",
                    "aws:ResourceTag/k8s.io/cluster-autoscaler/<my-cluster>": "owned"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeScalingActivities",
                "autoscaling:DescribeTags",
                "ec2:DescribeImages",
                "ec2:DescribeInstanceTypes",
                "ec2:DescribeLaunchTemplateVersions",
                "ec2:GetInstanceTypesFromInstanceRequirements",
                "eks:DescribeNodegroup"
            ],
            "Resource": "*"
        }
    ]
}
```


### 配置您的节点组

有效的自动扩展从正确配置集群的一组节点组开始。选择正确的节点组集合是最大化可用性和降低成本的关键。AWS使用EC2自动缩放组实现节点组,这对于大量用例来说都很灵活。但是,集群自动缩放器对您的节点组做出了一些假设。保持您的EC2自动缩放组配置与这些假设一致将最大限度地减少不需要的行为。

确保:

* 每个节点组中的每个节点都具有相同的调度属性,如标签、污点和资源。
  * 对于混合实例策略,实例类型的CPU、内存和GPU形状必须相同。
  * 策略中指定的第一种实例类型将用于模拟调度。
  * 如果您的策略有更多资源的其他实例类型,扩展后可能会浪费资源。
  * 如果您的策略有更少资源的其他实例类型,pod可能无法在实例上调度。
* 相比于较少节点的多个节点组,更喜欢拥有更多节点的节点组。这将对可扩展性产生最大影响。
* 尽可能使用EC2功能,因为两个系统都提供支持(例如区域、混合实例策略)。

*注意:我们建议使用[EKS托管节点组](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html)。托管节点组带有强大的管理功能,包括集群自动缩放器的功能,如自动发现EC2自动缩放组和优雅的节点终止。*

## 优化性能和可扩展性

了解自动缩放算法的运行时复杂度将有助于您调整集群自动缩放器,使其在拥有超过[1,000个节点](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/proposals/scalability_tests.md)的大型集群中继续平稳运行。

调整集群自动缩放器可扩展性的主要参数是提供给该进程的资源、算法的扫描间隔和集群中的节点组数量。还有其他涉及该算法真实运行复杂度的因素,如调度插件复杂度和pod数量。这些被视为不可配置参数,因为它们是集群工作负载的自然属性,无法轻易调整。

集群自动缩放器会将整个集群的状态(包括pod、节点和节点组)加载到内存中。在每个扫描间隔中,该算法识别无法调度的pod,并为每个节点组模拟调度。调整这些因素会带来不同的权衡,应该仔细考虑您的用例。

### 垂直自动扩展集群自动缩放器

缩放集群自动缩放器以处理更大集群的最简单方法是增加其部署的资源请求。对于大型集群,内存和 CPU 都应该增加,但这会因集群大小而有很大差异。自动缩放算法将所有 pod 和节点存储在内存中,这可能会导致内存占用超过 1 GB。通常手动增加资源。如果发现持续调整资源会造成运营负担,可以考虑使用 [Addon Resizer](https://github.com/kubernetes/autoscaler/tree/master/addon-resizer) 或 [Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)。

### 减少节点组的数量

最小化节点组的数量是确保集群自动缩放器在大型集群上继续良好运行的一种方法。对于一些按团队或应用程序构建节点组的组织来说,这可能是一个挑战。虽然这完全支持 Kubernetes API,但这被视为集群自动缩放器的反模式,会对可扩展性产生影响。使用多个节点组有很多原因(例如 Spot 或 GPU),但在许多情况下,存在可以在使用少量组的情况下实现相同效果的替代设计。

确保:

* 使用命名空间而不是节点组进行 Pod 隔离。
  * 在低信任的多租户集群中可能无法实现。
  * 正确设置 Pod ResourceRequests 和 ResourceLimits 以避免资源争用。
  * 使用更大的实例类型将导致更优化的装箱和减少系统 pod 开销。
* 使用 NodeTaints 或 NodeSelectors 作为例外,而不是规则来调度 pod。
* 区域资源被定义为具有多个可用性区域的单个 EC2 Auto Scaling Group。

### 减少扫描间隔

较低的扫描间隔(例如 10 秒)将确保集群自动缩放器在 pod 无法调度时尽快做出响应。但是,每次扫描都会导致对 Kubernetes API 和 EC2 Auto Scaling Group 或 EKS 托管节点组 API 进行大量 API 调用。这些 API 调用可能会导致速率限制或甚至 Kubernetes 控制平面的服务不可用。

默认扫描间隔为 10 秒,但在 AWS 上,启动节点需要显著更长的时间。这意味着可以增加间隔,而不会显著增加整体扩展时间。例如,如果需要 2 分钟才能启动一个节点,将间隔更改为 1 分钟将以 6 倍减少 API 调用为代价,缩放速度降低 38%。

### 跨节点组分片
集群自动缩放器可以配置为在特定的节点组上运作。使用此功能,可以部署多个集群自动缩放器实例,每个实例都配置为在不同的节点组上运作。这种策略使您能够使用任意数量的节点组,以成本换取可扩展性。我们只建议在提高性能的最后一种手段中使用这种方法。

集群自动缩放器最初并未设计用于此配置,因此会产生一些副作用。由于分片之间不会通信,多个自动缩放器可能会尝试调度一个无法调度的 pod。这可能会导致多个节点组不必要地进行扩容。这些额外的节点将在 `scale-down-delay` 后缩回。
```
metadata:
  name: cluster-autoscaler
  namespace: cluster-autoscaler-1

...

--nodes=1:10:k8s-worker-asg-1
--nodes=1:10:k8s-worker-asg-2

---

metadata:
  name: cluster-autoscaler
  namespace: cluster-autoscaler-2

...

--nodes=1:10:k8s-worker-asg-3
--nodes=1:10:k8s-worker-asg-4
```


确保:

* 每个分片都配置为指向一组唯一的 EC2 Auto Scaling 组
* 每个分片都部署到一个单独的命名空间,以避免领导者选举冲突

## 优化成本和可用性

### 竞价实例

您可以在节点组中使用竞价实例,并节省高达 90% 的按需价格,但代价是竞价实例可能随时被中断,当 EC2 需要回收容量时。当您的 EC2 Auto Scaling 组无法由于缺乏可用容量而扩展时,就会发生容量不足错误。通过选择多种实例系列来最大化多样性,可以增加您实现所需规模的机会,并减少竞价实例中断对集群可用性的影响。使用混合实例策略与竞价实例是增加多样性而不增加节点组数量的好方法。请记住,如果您需要有保证的资源,请使用按需实例而不是竞价实例。

在配置混合实例策略时,所有实例类型的资源容量都必须相似。自动扩展器的调度模拟器使用混合实例策略中的第一个实例类型。如果后续实例类型更大,扩容后可能会浪费资源。如果更小,您的 pod 可能无法调度到新实例上,因为容量不足。例如,M4、M5、M5a 和 M5n 实例的 CPU 和内存容量相似,是混合实例策略的良好候选。[EC2 实例选择器](https://github.com/aws/amazon-ec2-instance-selector)工具可以帮助您识别相似的实例类型。

![](./spot_mix_instance_policy.jpg)

建议将按需和竞价容量隔离到单独的 EC2 Auto Scaling 组中。这比使用[基础容量策略](https://docs.aws.amazon.com/autoscaling/ec2/userguide/asg-purchase-options.html#asg-instances-distribution)更可取,因为调度属性根本不同。由于竞价实例可能随时被中断(当 EC2 需要回收容量时),用户通常会污染其可抢占节点,需要对抢占行为进行明确的 pod 容忍。这些污点导致节点的调度属性不同,因此它们应该被分离到多个 EC2 Auto Scaling 组中。

集群自动扩展器有一个[扩展器](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md#what-are-expanders)的概念,提供不同的策略来选择扩展哪个节点组。`--expander=least-waste`策略是一个很好的通用默认值,如果您要使用多个节点组来实现竞价实例多样化(如上图所述),它可以通过扩展最佳利用的节点组来进一步优化成本。

### 优先考虑节点组/ASG
您还可以使用优先级扩展器配置基于优先级的自动缩放。`--expander=priority`使您的集群能够优先考虑一个节点组/ASG,如果由于任何原因无法扩展,它将选择优先列表中的下一个节点组。在某些情况下,这很有用,例如,您希望使用P3实例类型,因为它们的GPU为您的工作负载提供了最佳性能,但作为第二选择,您也可以使用P2实例类型。
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-priority-expander
  namespace: kube-system
data:
  priorities: |-
    10:
      - .*p2-node-group.*
    50:
      - .*p3-node-group.*
```


集群自动缩放器将尝试扩大与名称 *p3-node-group* 匹配的 EC2 Auto Scaling 组。如果此操作在 `--max-node-provision-time` 内未能成功，它将尝试扩大与名称 *p2-node-group* 匹配的 EC2 Auto Scaling 组。
此值默认为 15 分钟,可以减少以获得更快的节点组选择,但如果该值过低,可能会导致不必要的扩展。

### 过度配置

集群自动缩放器通过确保只在需要时添加节点并在未使用时删除节点来最大限度地降低成本。这会显著影响部署延迟,因为许多 pod 将被迫等待节点扩展才能被调度。节点可能需要多分钟才能可用,这可能会将 pod 调度延迟增加一个数量级。

这可以通过使用[过度配置](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md#how-can-i-configure-overprovisioning-with-cluster-autoscaler)来缓解,这会以成本为代价换取调度延迟。过度配置是通过使用负优先级的临时 pod 来实现的,这些 pod 占用集群中的空间。当新创建的 pod 无法调度且具有更高优先级时,临时 pod 将被抢占以腾出空间。临时 pod 然后变为无法调度,触发集群自动缩放器扩展新的过度配置节点。

过度配置还有其他不太明显的好处。没有过度配置,高利用率集群的一个副作用是 pod 将使用 Pod 或节点亲和性的 `preferredDuringSchedulingIgnoredDuringExecution` 规则做出较次优的调度决策。一个常见的用例是使用 AntiAffinity 将高可用应用程序的 pod 跨可用性区域分开。过度配置可以显著提高可用正确区域节点的机会。

过度配置容量的数量是您组织的一个谨慎的业务决策。从根本上说,这是性能和成本之间的权衡。做出这一决定的一种方法是确定您的平均扩展频率,并将其除以新节点的供应时间。例如,如果您平均每 30 秒需要一个新节点,而 EC2 需要 30 秒来配置一个新节点,一个过度配置节点将确保始终有一个额外的节点可用,将调度延迟减少 30 秒,代价是一个额外的 EC2 实例。为了改善区域调度决策,过度配置等同于您 EC2 Auto Scaling 组中可用性区域数量的节点数,以确保调度程序可以为传入的 pod 选择最佳区域。

### 防止缩减驱逐

某些工作负载驱逐起来代价高昂。大数据分析、机器学习任务和测试运行程序最终会完成,但如果中断则必须重新启动。集群自动缩放器将尝试缩减任何低于缩减利用率阈值的节点,这将中断节点上剩余的任何 pod。通过确保被驱逐代价高昂的 pod 受到集群自动缩放器识别的标签的保护,可以防止这种情况发生。

确保:

* 驱逐代价高昂的 pod 具有注解 `cluster-autoscaler.kubernetes.io/safe-to-evict=false`

## 高级用例

### EBS 卷

持久存储对于构建有状态应用程序(如数据库或分布式缓存)至关重要。[EBS 卷](https://aws.amazon.com/premiumsupport/knowledge-center/eks-persistent-storage/)在 Kubernetes 上启用了此用例,但仅限于特定区域。如果跨多个可用区分片,这些应用程序可以实现高可用性,每个可用区使用单独的 EBS 卷。集群自动缩放器然后可以平衡 EC2 自动缩放组的缩放。

确保:

* 通过设置 `balance-similar-node-groups=true` 启用节点组平衡。
* 节点组配置有相同的设置,只是不同的可用区和 EBS 卷。

### 协同调度

机器学习分布式训练作业从同区域节点配置的最小延迟中获益很大。这些工作负载部署多个 pod 到特定区域。这可以通过为所有协同调度的 pod 设置 Pod 亲和性或使用 `topologyKey: failure-domain.beta.kubernetes.io/zone` 设置节点亲和性来实现。集群自动缩放器然后将扩展特定区域以满足需求。您可能希望分配多个 EC2 自动缩放组,每个可用区一个,以实现整个协同调度工作负载的故障转移。

确保:

* 通过设置 `balance-similar-node-groups=false` 启用节点组平衡
* 当集群包含区域和区域节点组时,使用[节点亲和性](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)和/或[Pod 抢占](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/)
  * 使用[节点亲和性](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)来强制或鼓励区域 pod 避免区域节点组,反之亦然。
  * 如果区域 pod 调度到区域节点组,这将导致区域 pod 的容量不平衡。
  * 如果您的区域工作负载可以容忍中断和重新定位,请配置[Pod 抢占](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/)以使区域扩展的 pod 能够强制抢占并重新调度到竞争较少的区域。

### 加速器

某些集群利用了诸如 GPU 等专用硬件加速器。在扩展时,加速器设备插件需要几分钟的时间才能将资源广告到集群中。集群自动缩放器已模拟该节点将拥有加速器,但在加速器准备就绪并更新节点的可用资源之前,待处理的 pod 无法在该节点上进行调度。这可能导致[重复的不必要扩展](https://github.com/kubernetes/kubernetes/issues/54959)。

此外,具有加速器以及高 CPU 或内存利用率的节点将不会被考虑进行缩减,即使加速器未被使用。这种行为可能会由于加速器的相对成本而变得昂贵。相反,集群自动缩放器可以应用特殊规则,以考虑具有未占用加速器的节点进行缩减。

为确保这些情况下的正确行为,您可以配置加速器节点上的 kubelet,以在节点加入集群之前对其进行标记。集群自动缩放器将使用此标签选择器来触发加速器优化行为。

请确保:

* GPU 节点的 Kubelet 配置有 `--node-labels k8s.amazonaws.com/accelerator=$ACCELERATOR_TYPE`
* 具有加速器的节点遵守上述相同的调度属性规则。

### 从 0 开始扩展

集群自动缩放器能够将节点组扩展到 0 并从 0 开始扩展,这可以带来显著的成本节省。它通过检查 LaunchConfiguration 或 LaunchTemplate 中指定的 InstanceType 来检测自动缩放组的 CPU、内存和 GPU 资源。某些 pod 需要额外的资源,如 `WindowsENI` 或 `PrivateIPv4Address` 或特定的 NodeSelectors 或 Taints,这些无法从 LaunchConfiguration 中发现。集群自动缩放器可以通过从 EC2 自动缩放组的标签中发现这些因素来考虑它们。例如:
```
Key: k8s.io/cluster-autoscaler/node-template/resources/$RESOURCE_NAME
Value: 5
Key: k8s.io/cluster-autoscaler/node-template/label/$LABEL_KEY
Value: $LABEL_VALUE
Key: k8s.io/cluster-autoscaler/node-template/taint/$TAINT_KEY
Value: NoSchedule
```


*注意:请记住,当缩放到零时,您的容量将返回到 EC2,并可能在将来无法使用。*

## 其他参数

可以使用许多配置选项来调整集群自动缩放器的行为和性能。
可在 [GitHub](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md#what-are-the-parameters-to-ca) 上获得完整的参数列表。

|  |  |  |
|-|-|-|
| 参数 | 描述 | 默认值 |
| scan-interval | 集群重新评估进行扩展或缩减的频率 | 10 秒 |
| max-empty-bulk-delete | 可以同时删除的空节点的最大数量 | 10 |
| scale-down-delay-after-add | 扩展后恢复缩减评估的延迟时间 | 10 分钟 |
| scale-down-delay-after-delete | 节点删除后恢复缩减评估的延迟时间,默认为 scan-interval | scan-interval |
| scale-down-delay-after-failure | 缩减失败后恢复缩减评估的延迟时间 | 3 分钟 |
| scale-down-unneeded-time | 节点被视为不需要的时间,才有资格进行缩减 | 10 分钟 |
| scale-down-unready-time | 未就绪节点被视为不需要的时间,才有资格进行缩减 | 20 分钟 |
| scale-down-utilization-threshold | 节点利用率水平,定义为请求资源之和除以容量,低于该水平的节点可被视为缩减候选 | 0.5 |
| scale-down-non-empty-candidates-count | 每次迭代中考虑作为缩减候选的非空节点的最大数量。较低的值意味着更好的 CA 响应能力,但可能会导致较慢的缩减延迟。较高的值可能会影响具有大型集群(数百个节点)的 CA 性能。设置为非正值可关闭此启发式算法 - CA 将不会限制它考虑的节点数量。 | 30 |
| scale-down-candidates-pool-ratio | 当某些候选节点在前一次迭代中不再有效时,被视为额外非空候选节点的比例。较低的值意味着更好的 CA 响应能力,但可能会导致较慢的缩减延迟。较高的值可能会影响具有大型集群(数百个节点)的 CA 性能。设置为 1.0 可关闭此启发式算法 - CA 将把所有节点视为额外候选节点。 | 0.1 |
| scale-down-candidates-pool-min-count | 当某些候选节点在前一次迭代中不再有效时,被视为额外非空候选节点的最小数量。在计算额外候选池的大小时,我们取 `max(#nodes * scale-down-candidates-pool-ratio, scale-down-candidates-pool-min-count)` | 50 |

## 其他资源

本页面包含集群自动缩放器演示文稿和演示。如果您想在此添加演示文稿或演示,请发送拉取请求。

| 演示文稿/演示 | 演讲者 |
| ------------ | ------- |
[自动扩缩容和成本优化 Kubernetes: 从 0 到 100](https://sched.co/Zemi) | Guy Templeton, Skyscanner 和 Jiaxin Shan, Amazon

[SIG-Autoscaling 深入探讨](https://youtu.be/odxPyW_rZNQ) | Maciek Pytel 和 Marcin Wielgus

## 参考资料

* [https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md)
* [https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md)
* [https://github.com/aws/amazon-ec2-instance-selector](https://github.com/aws/amazon-ec2-instance-selector)
* [https://github.com/aws/aws-node-termination-handler](https://github.com/aws/aws-node-termination-handler)
