!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

# 成本优化 - 计算和自动扩缩

作为开发人员,您将对应用程序的资源需求(如 CPU 和内存)进行估计,但如果您不持续调整它们,它们可能会过时,从而增加成本并降低性能和可靠性。持续调整应用程序的资源需求比第一次就正确设置更为重要。

下面提到的最佳实践将帮助您构建和运营成本意识的工作负载,在最小化成本的同时实现业务目标,并使您的组织最大化投资回报。优化集群计算成本的重要性顺序如下:

1. 合理调整工作负载
2. 减少未使用的容量
3. 优化计算容量类型(如 Spot)和加速器(如 GPU)

## 合理调整您的工作负载

在大多数 EKS 集群中,大部分成本来自用于运行容器化工作负载的 EC2 实例。如果不了解您的工作负载需求,您将无法合理调整计算资源。这就是为什么使用适当的请求和限制并根据需要调整这些设置非常重要。此外,实例大小和存储选择等依赖项可能会影响工作负载性能,从而对成本和可靠性产生各种意想不到的后果。

*请求*应与实际利用率保持一致。如果容器的请求过高,将会有未使用的容量,这是总集群成本的一个重要因素。每个 pod 中的容器(如应用程序和 sidecar)都应设置自己的请求和限制,以确保 pod 限制的总和尽可能准确。

利用诸如 [Goldilocks](https://www.youtube.com/watch?v=DfmQWYiwFDk)、[KRR](https://www.youtube.com/watch?v=uITOzpf82RY) 和 [Kubecost](https://aws.amazon.com/blogs/containers/aws-and-kubecost-collaborate-to-deliver-cost-monitoring-for-eks-customers/) 等工具来估算容器的资源请求和限制。根据应用程序的性质、性能/成本要求和复杂性,您需要评估哪些指标最适合扩展,应用程序性能在哪个饱和点开始下降,以及如何相应调整请求和限制。请参考[应用程序合理调整](https://aws.github.io/aws-eks-best-practices/scalability/docs/node_efficiency/#application-right-sizing)以获得更多指导。

我们建议使用水平 Pod 自动缩放器 (HPA) 来控制应用程序应该运行的副本数量,使用垂直 Pod 自动缩放器 (VPA) 来调整每个副本所需的请求和限制,以及使用 [Karpenter](http://karpenter.sh/) 或 [Cluster Autoscaler](https://github.com/kubernetes/autoscaler) 等节点自动缩放器来持续调整集群中的总节点数。本文档后续部分记录了使用 Karpenter 和 Cluster Autoscaler 进行成本优化的技术。

垂直 Pod 自动缩放器可以调整分配给容器的请求和限制,使工作负载能够最佳运行。您应该以审核模式运行 VPA,这样它就不会自动进行更改并重新启动您的 Pod。它将根据观察到的指标提出建议。对于任何会影响生产工作负载的更改,您应该先在非生产环境中进行审查和测试,因为这些更改可能会影响应用程序的可靠性和性能。

## 减少消耗

节省成本的最佳方式是配置更少的资源。一种方法是根据当前需求调整工作负载。您应该从确保您的工作负载定义了其需求并动态扩展开始任何成本优化工作。这将需要从您的应用程序获取指标,并设置诸如 [`PodDisruptionBudgets`](https://kubernetes.io/docs/tasks/run-application/configure-pdb/) 和 [Pod Readiness Gates](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.5/deploy/pod_readiness_gate/) 等配置,以确保您的应用程序可以安全地动态扩展。

水平 Pod 自动缩放器是一个灵活的工作负载自动缩放器,可以根据应用程序的性能和可靠性需求调整所需的副本数量。它有一个灵活的模型,可以根据 CPU、内存或自定义指标(如队列深度、连接到 Pod 的数量等)来定义何时进行扩缩容。

Kubernetes Metrics Server 可以根据内置的 CPU 和内存使用指标进行扩缩容,但如果您想根据其他指标(如 Amazon CloudWatch 或 SQS 队列深度)进行扩缩容,您应该考虑使用事件驱动的自动缩放项目,如 [KEDA](https://keda.sh/)。请参考[此博客文章](https://aws.amazon.com/blogs/mt/proactive-autoscaling-of-kubernetes-workloads-with-keda-using-metrics-ingested-into-amazon-cloudwatch/)了解如何将 KEDA 与 CloudWatch 指标一起使用。如果您不确定应该监控和缩放哪些指标,请查看[关于监控重要指标的最佳实践](https://aws-observability.github.io/observability-best-practices/guides/#monitor-what-matters)。

减少工作负载消耗会在集群中创造出剩余容量,并且通过适当的自动扩缩配置,可以自动缩减节点并降低总体支出。我们建议您不要尝试手动优化计算能力。Kubernetes调度程序和节点自动缩放器被设计用于处理这个过程。

## 减少未使用的容量

确定应用程序的正确大小并减少多余的请求后,您可以开始减少供应的计算容量。如果您已经正确地调整了上述部分的工作负载大小,您应该能够动态地完成这项工作。AWS中使用Kubernetes的两个主要节点自动缩放器如下:

### Karpenter和集群自动缩放器

Karpenter和Kubernetes集群自动缩放器都会根据Pod的创建或删除以及计算需求的变化来调整集群中的节点数量。它们的主要目标是相同的,但Karpenter采取了不同的节点管理供应和取消供应方法,这可以帮助降低成本并优化整个集群的使用情况。

随着集群规模的增大和工作负载种类的增加,预先配置节点组和实例变得更加困难。就像工作负载请求一样,设置一个初始基线并根据需要不断调整很重要。

### 集群自动缩放器优先级扩展器

Kubernetes集群自动缩放器通过扩展和缩减节点组来实现集群的扩缩。如果您没有动态扩缩工作负载,集群自动缩放器将无法帮助您节省资金。集群自动缩放器要求集群管理员事先为工作负载创建节点组。节点组需要配置为使用具有相同"配置文件"的实例,即大致相同的CPU和内存。

您可以拥有多个节点组,集群自动缩放器可以配置为设置优先级缩放级别,每个节点组可以包含不同大小的节点。节点组可以具有不同的容量类型,优先级扩展器可用于首先扩展较便宜的组。

下面是一个使用`ConfigMap`设置预留容量优先级于按需实例的集群配置示例片段。您可以使用相同的技术来优先使用Graviton或Spot实例而不是其他类型。
```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-cluster
managedNodeGroups:
  - name: managed-ondemand
    minSize: 1
    maxSize: 7
    instanceType: m5.xlarge
  - name: managed-reserved
    minSize: 2
    maxSize: 10
    instanceType: c5.2xlarge
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-priority-expander
  namespace: kube-system
data:
  priorities: |-
    10:
      - .*ondemand.*
    50:
      - .*reserved.*
```


使用节点组可以帮助底层计算资源默认执行预期的操作,例如跨可用区分散节点,但并非所有工作负载都有相同的要求或期望,最好让应用程序明确声明其要求。有关集群自动缩放器的更多信息,请参见[最佳实践部分](https://aws.github.io/aws-eks-best-practices/cluster-autoscaling/)。

### 去调度器

集群自动缩放器可以根据需要调度新 pod 或节点利用率低而向集群添加或删除节点容量。它不会在 pod 被调度到节点后全面考虑 pod 放置。如果您使用集群自动缩放器,您还应该查看[Kubernetes 去调度器](https://github.com/kubernetes-sigs/descheduler),以避免浪费集群中的容量。

如果集群中有 10 个节点,每个节点的利用率为 60%,则集群中有 40% 的配置容量未被使用。使用集群自动缩放器,您可以将每个节点的利用率阈值设置为 60%,但这只会在某个节点的利用率降至 60% 以下时尝试缩减该节点。

使用去调度器,它可以在 pod 被调度或节点被添加到集群后查看集群容量和利用率。它试图保持集群的总容量高于指定的阈值。它还可以根据节点污点或新加入集群的节点删除 pod,以确保 pod 在其最佳计算环境中运行。请注意,去调度器不会安排被驱逐 pod 的替换,而是依赖默认调度器来完成此操作。

### Karpenter 整合

Karpenter 采取了一种"无组"的节点管理方法。这种方法对于不同类型的工作负载更加灵活,并且需要集群管理员进行更少的初始配置。Karpenter 不是预先定义组并根据工作负载需求对每个组进行扩缩,而是使用供应商和节点模板来定义可以创建的 EC2 实例类型以及实例创建时的设置。

装箱优化是利用更多实例资源的做法,通过将更多工作负载打包到更少的最佳规模实例上。虽然这有助于通过只配置工作负载使用的资源来降低计算成本,但也有权衡。在大规模扩展事件期间,因为需要向集群添加容量,启动新工作负载可能需要更长时间。在设置装箱优化时,请权衡成本优化、性能和可用性之间的平衡。
Karpenter可以持续监控和装箱以提高实例资源利用率并降低您的计算成本。Karpenter还可以为您的工作负载选择更具成本效益的工作节点。这可以通过在配置器中将"consolidation"标志设置为true来实现(下面是示例代码片段)。下面的示例显示了一个启用了consolidation的配置器。在撰写本指南时,Karpenter不会用更便宜的Spot实例替换正在运行的Spot实例。有关Karpenter consolidation的更多详细信息,请参阅[此博客](https://aws.amazon.com/blogs/containers/optimizing-your-kubernetes-compute-costs-with-karpenter-consolidation/)。
```yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: enable-binpacking
spec:
  consolidation:
    enabled: true
```

对于可能无法中断的工作负载(例如没有检查点的长时间批处理作业)，请考虑使用 `do-not-evict` 注解标注 pod。通过选择不驱逐 pod，您正在告诉 Karpenter 不应自愿删除包含此 pod 的节点。但是，如果在节点正在排空时添加了 `do-not-evict` pod，其他 pod 仍将被驱逐，但该 pod 将阻止终止，直到将其删除。在任何一种情况下，该节点都将被隔离以防止在该节点上安排其他工作。下面是一个示例,展示如何设置注解:
```yaml hl_lines="8"
apiVersion: v1
kind: Pod
metadata:
  name: label-demo
  labels:
    environment: production
  annotations:  
    "karpenter.sh/do-not-evict": "true"
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

通过调整集群自动缩放器参数来删除利用不足的节点

节点利用率定义为请求资源之和除以容量。默认情况下，`scale-down-utilization-threshold`设置为50%。此参数可与`scale-down-unneeded-time`一起使用,后者确定在节点有资格缩减之前应该保持未使用状态的时间 - 默认为10分钟。缩减节点上仍在运行的Pod将由kube-scheduler调度到其他节点上。调整这些设置可以帮助删除利用不足的节点,但重要的是您要先测试这些值,以免迫使集群过早缩减。

您可以通过确保使用集群自动缩放器识别的标签来保护需要驱逐的昂贵Pod,从而防止缩减发生。为此,请确保需要驱逐的昂贵Pod具有注释`cluster-autoscaler.kubernetes.io/safe-to-evict=false`。下面是设置注释的示例yaml:
```yaml hl_lines="8"
apiVersion: v1
kind: Pod
metadata:
  name: label-demo
  labels:
    environment: production
  annotations:  
    "cluster-autoscaler.kubernetes.io/safe-to-evict": "false"
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```


### 使用集群自动缩放器和 Karpenter 标记节点

AWS 资源[标签](https://docs.aws.amazon.com/tag-editor/latest/userguide/tagging.html)用于组织您的资源,并在详细级别跟踪您的 AWS 成本。它们与 Kubernetes 标签用于成本跟踪没有直接关系。建议从 Kubernetes 资源标签开始,并利用[Kubecost](https://aws.amazon.com/blogs/containers/aws-and-kubecost-collaborate-to-deliver-cost-monitoring-for-eks-customers/)等工具根据 Kubernetes 标签在 pod、命名空间等上获得基础设施成本报告。

工作节点需要有标签以在 AWS Cost Explorer 中显示计费信息。使用集群自动缩放器时,请使用[启动模板](https://docs.aws.amazon.com/eks/latest/userguide/launch-templates.html)为托管节点组中的工作节点添加标签。对于自管理节点组,请使用[EC2 自动缩放组](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-tagging.html)为实例添加标签。对于由 Karpenter 配置的实例,请使用[节点模板中的 spec.tags](https://karpenter.sh/docs/concepts/nodeclasses/#spectags)为其添加标签。

### 多租户集群

在与不同团队共享的集群上工作时,您可能无法查看在同一节点上运行的其他工作负载。虽然资源请求可以帮助隔离一些"吵闹邻居"问题,例如 CPU 共享,但它们可能无法隔离所有资源边界,如磁盘 I/O 耗尽。工作负载消耗的每个可共享资源都无法隔离或限制。消耗共享资源速度高于其他工作负载的工作负载应通过节点[污点和容忍](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)进行隔离。这种工作负载的另一种高级技术是[CPU 固定](https://kubernetes.io/docs/tasks/administer-cluster/cpu-management-policies/#static-policy),它确保容器获得独占 CPU 而不是共享 CPU。

在节点级别隔离工作负载可能更昂贵,但可以调度[最佳努力](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/#besteffort)作业,或利用使用[预留实例](https://aws.amazon.com/ec2/pricing/reserved-instances/)、[Graviton 处理器](https://aws.amazon.com/ec2/graviton/)或[Spot](https://aws.amazon.com/ec2/spot/)实现额外节省。

共享集群还可能存在集群级资源约束,如 IP 耗尽、Kubernetes 服务限制或 API 缩放请求。您应该查看[可扩展性最佳实践指南](https://aws.github.io/aws-eks-best-practices/scalability/docs/control-plane/)以确保您的集群避免这些限制。

您可以在命名空间或 Karpenter 供应商级别隔离资源。[资源配额](https://kubernetes.io/docs/concepts/policy/resource-quotas/)提供了一种设置工作负载在命名空间中可以消耗的资源限制的方式。这可以作为一个良好的初始防护措施,但需要持续评估,以确保它不会人为地限制工作负载的扩展。

Karpenter 供应商可以[设置集群中某些可消耗资源(如 CPU、GPU)的限制](https://karpenter.sh/docs/concepts/nodepools/#speclimitsresources),但您需要配置租户应用程序以使用适当的供应商。这可以防止单个供应商在集群中创建太多节点,但也需要持续评估,以确保限制没有设置得太低,从而阻碍工作负载的扩展。

### 计划的自动扩缩

您可能需要在周末和非工作时间缩小集群规模。这对于测试和非生产集群特别相关,在它们不使用时可以缩小到零。[cluster-turndown](https://github.com/kubecost/cluster-turndown) 和 [kube-downscaler](https://codeberg.org/hjacobs/kube-downscaler) 等解决方案可以根据 cron 计划将副本缩小到零。

## 优化计算能力类型

在优化集群中计算总容量并利用装箱优化后,您应该关注所配置的计算类型以及为这些资源的付费方式。AWS 提供[计算节省计划](https://aws.amazon.com/savingsplans/compute-pricing/),可以降低您的计算成本,我们将其划分为以下容量类型:

* 竞价实例
* 节省计划
* 按需
* Fargate

每种容量类型在管理开销、可用性和长期承诺方面都有不同的权衡,您需要决定哪种类型最适合您的环境。任何环境都不应该依赖于单一的容量类型,您可以在单个集群中混合使用多种运行类型,以优化特定工作负载的需求和成本。

### 竞价实例

[竞价实例](https://aws.amazon.com/ec2/spot/)从可用区的剩余容量中配置 EC2 实例。竞价实例提供最大的折扣 - 高达 90%,但这些实例可能会在需要时被中断。此外,可能无法始终配置新的竞价实例,现有的竞价实例也可能在[2 分钟中断通知](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-interruptions.html)后被收回。如果您的应用程序有较长的启动或关闭过程,竞价实例可能不是最佳选择。
使用各种各样的实例类型进行临时计算可以降低无法获得临时容量的可能性。需要处理实例中断以安全地关闭节点。使用 Karpenter 或托管节点组部署的节点会自动支持[实例中断通知](https://aws.github.io/aws-eks-best-practices/karpenter/#enable-interruption-handling-when-using-spot)。如果您使用自管理节点,则需要单独运行[节点终止处理程序](https://github.com/aws/aws-node-termination-handler)来优雅地关闭临时实例。

可以在单个集群中平衡临时实例和按需实例。使用 Karpenter,您可以创建[加权配置器](https://karpenter.sh/docs/concepts/scheduling/#on-demandspot-ratio-split)来实现不同容量类型的平衡。使用集群自动缩放器,您可以创建[混合节点组,包含临时和按需或预留实例](https://aws.amazon.com/blogs/containers/amazon-eks-now-supports-provisioning-and-managing-ec2-spot-instances-in-managed-node-groups/)。

以下是使用 Karpenter 优先考虑临时实例而不是按需实例的示例。在创建配置器时,您可以指定临时、按需或两者(如下所示)。当您同时指定两者,并且 pod 没有明确指定是否需要使用临时或按需时,Karpenter 会优先考虑使用[价格容量优化分配策略](https://aws.amazon.com/blogs/compute/introducing-price-capacity-optimized-allocation-strategy-for-ec2-spot-instances/)来配置节点。
```yaml hl_lines="9"
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: spot-prioritized
spec:
  requirements:
    - key: "karpenter.sh/capacity-type" 
      operator: In
        values: ["spot", "on-demand"]
```


### 节省计划、预留实例和 AWS EDP

您可以通过使用[计算节省计划](https://aws.amazon.com/savingsplans/compute-pricing/)来降低计算支出。节省计划提供 1 年或 3 年计算使用承诺的优惠价格。使用范围可以包括 EKS 集群中的 EC2 实例,也可以包括 Lambda 和 Fargate 等任何计算使用。使用节省计划,您可以在承诺期内降低成本,并仍可选择任何 EC2 实例类型。

计算节省计划可将您的 EC2 成本降低高达 66%,无需承诺使用特定的实例类型、系列或区域。使用时会自动应用节省。

EC2 实例节省计划可在特定区域和 EC2 系列(如 C 系列实例)中提供高达 72% 的计算成本节省。您可以在该区域内的任何可用区之间转移使用,使用该系列的任何一代实例(如 c5 或 c6),并使用该系列内的任何大小实例。折扣将自动应用于您帐户中符合节省计划标准的任何实例。

[预留实例](https://aws.amazon.com/ec2/pricing/reserved-instances/)类似于 EC2 实例节省计划,但它们还可保证可用区或区域的容量,并相比按需实例减少成本 - 高达 72%。一旦您计算出所需的预留容量,就可以选择预留期(1 年或 3 年)。折扣将自动应用于您帐户中运行的这些 EC2 实例。

客户还可选择与 AWS 签订企业协议。企业协议使客户能够定制最适合其需求的协议。客户可根据 AWS EDP(企业折扣计划)享受价格折扣。有关企业协议的更多信息,请联系您的 AWS 销售代表。

### 按需

按需 EC2 实例具有无中断可用性(与点实例相比)和无长期承诺(与节省计划相比)的优势。如果您希望降低集群的成本,应该减少使用按需 EC2 实例。

优化您的工作负载要求后,您应该能够计算出集群的最小和最大容量。这个数字可能会随时间而变化,但很少会下降。考虑对最小容量使用节省计划,对不会影响应用程序可用性的容量使用点实例。其余可能不会持续使用或需要可用性的部分可使用按需。
正如本节所述，减少使用量的最佳方式是减少资源消耗,并尽可能充分利用您配置的资源。使用集群自动缩放器,您可以通过`scale-down-utilization-threshold`设置删除利用率较低的节点。对于 Karpenter,建议启用合并功能。

要手动识别可与您的工作负载一起使用的 EC2 实例类型,您应该使用 [ec2-instance-selector](https://github.com/aws/amazon-ec2-instance-selector),它可以显示每个区域可用的实例以及与 EKS 兼容的实例。以下是一个示例用法,用于查找具有 x86 处理器架构、4 GB 内存、2 个 vCPU 且在 us-east-1 区域可用的实例。
```bash
ec2-instance-selector --memory 4 --vcpus 2 --cpu-architecture x86_64 \
  -r us-east-1 --service eks
c5.large
c5a.large
c5ad.large
c5d.large
c6a.large
c6i.large
t2.medium
t3.medium
t3a.medium
```


对于非生产环境,您可以在夜间和周末等闲置时间自动缩小集群规模。kubecost 项目 [cluster-turndown](https://github.com/kubecost/cluster-turndown) 是一个可以根据预定时间表自动缩小集群规模的控制器示例。

### Fargate 计算

Fargate 计算是 EKS 集群的一种完全托管的计算选项。它通过在 Kubernetes 集群中每个节点上调度一个 pod 来提供 pod 隔离。它允许您根据工作负载的 CPU 和内存需求调整计算节点的大小,从而更好地控制集群中的工作负载使用情况。

Fargate 可以缩放到最小 0.25 vCPU 和 0.5 GB 内存,最大 16 vCPU 和 120 GB 内存的工作负载。可用的 [pod 大小变化](https://docs.aws.amazon.com/eks/latest/userguide/fargate-pod-configuration.html) 有限,您需要了解如何最好地将您的工作负载配置到 Fargate 中。例如,如果您的工作负载需要 1 vCPU 和 0.5 GB 内存,最小的 Fargate pod 将是 1 vCPU 和 2 GB 内存。

虽然 Fargate 有许多优点,如无需管理 EC2 实例或操作系统,但由于每个部署的 pod 都被隔离为集群中的单独节点,它可能需要比传统 EC2 实例更多的计算容量。这需要对诸如 Kubelet、日志代理和通常部署到节点的任何 DaemonSet 进行更多复制。Fargate 不支持 DaemonSet,它们需要转换为 pod "sidecar"并与应用程序一起运行。

Fargate 无法从 bin packing 或 CPU 过度配置中获益,因为工作负载的边界是一个不可突发或共享的节点。Fargate 将为您节省 EC2 实例管理时间,这本身就有成本,但 CPU 和内存成本可能比其他 EC2 容量类型更昂贵。Fargate pod 可以利用计算节省计划来降低按需成本。

## 优化计算使用

节省计算基础设施成本的另一种方式是使用更高效的计算资源。这可以来自更高性能的通用计算,如 [Graviton 处理器](https://aws.amazon.com/ec2/graviton/)，它比 x86 便宜 20% 并且能源效率高 60%,或者是针对特定工作负载的加速器,如 GPU 和 [FPGA](https://aws.amazon.com/ec2/instance-types/f1/)。您需要构建可以[在 arm 架构上运行](https://aws.amazon.com/blogs/containers/how-to-build-your-containers-for-arm-and-save-with-graviton-and-spot-instances-on-amazon-ecs/)的容器,并[设置具有合适加速器](https://aws.amazon.com/blogs/compute/running-gpu-accelerated-kubernetes-workloads-on-p3-and-p2-ec2-instances-with-amazon-eks/)的节点来满足您的工作负载需求。
EKS 具有运行混合架构集群(例如 amd64 和 arm64)的能力,如果您的容器编译支持多种架构,您可以通过在配置器中允许使用两种架构来利用 Graviton 处理器与 Karpenter。但为了保持一致的性能,建议您将每个工作负载保持在单一计算架构上,仅在没有额外容量可用时才使用不同的架构。

配置器可以配置多种架构,工作负载规范也可以请求特定的架构。
```yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
  - key: "kubernetes.io/arch"
    operator: In
    values: ["arm64", "amd64"]
```

使用集群自动缩放器，您需要为 Graviton 实例创建一个节点组，并在您的工作负载上设置[节点容忍度](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)以利用新的容量。

GPU 和 FPGA 可以大大提高您工作负载的性能，但工作负载需要进行优化以使用加速器。许多机器学习和人工智能的工作负载类型可以使用 GPU 进行计算，并且可以将实例添加到集群中并挂载到工作负载中，使用资源请求。
```yaml
spec:
  template:
    spec:
    - containers:
      ...
      resources:
          limits:
            nvidia.com/gpu: "1"
```

一些 GPU 硬件可以跨多个工作负载共享,因此单个 GPU 可以被配置和使用。要了解如何配置工作负载 GPU 共享,请参见[虚拟 GPU 设备插件](https://aws.amazon.com/blogs/opensource/virtual-gpu-device-plugin-for-inference-workload-in-kubernetes/)以获取更多信息。您还可以参考以下博客:

* [在 Amazon EKS 上使用 NVIDIA 时间切片和加速的 EC2 实例进行 GPU 共享](https://aws.amazon.com/blogs/containers/gpu-sharing-on-amazon-eks-with-nvidia-time-slicing-and-accelerated-ec2-instances/)
* [利用 NVIDIA 的多实例 GPU (MIG) 在 Amazon EKS 上最大化 GPU 利用率:为提高性能而每个 GPU 运行更多 pod](https://aws.amazon.com/blogs/containers/maximizing-gpu-utilization-with-nvidias-multi-instance-gpu-mig-on-amazon-eks-running-more-pods-per-gpu-for-enhanced-performance/)
