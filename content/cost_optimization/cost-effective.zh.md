# 成本效益资源

成本效益资源意味着使用适当的服务、资源和配置来运行Kubernetes集群上的工作负载,从而实现成本节省。

## 建议

### 确保用于部署容器化服务的基础设施与应用程序配置文件和扩展需求相匹配

Amazon EKS支持几种类型的Kubernetes自动扩展 - [集群自动扩展器](https://docs.aws.amazon.com/eks/latest/userguide/cluster-autoscaler.html)、[水平Pod自动扩展器](https://docs.aws.amazon.com/eks/latest/userguide/horizontal-pod-autoscaler.html)和[垂直Pod自动扩展器](https://docs.aws.amazon.com/eks/latest/userguide/vertical-pod-autoscaler.html)。本节涵盖了其中两个,即集群自动扩展器和水平Pod自动扩展器。

### 使用集群自动扩展器调整Kubernetes集群的大小以满足当前需求

[Kubernetes集群自动扩展器](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)会自动调整EKS集群中的节点数量,以满足由于资源不足而无法启动Pod或集群中节点利用率低且其Pod可以重新调度到集群中其他节点的需求。集群自动扩展器会在指定的Auto Scaling组内扩展工作节点,并作为部署在您的EKS集群中运行。

Amazon EKS与EC2托管节点组自动化了Amazon EKS Kubernetes集群节点(Amazon EC2实例)的配置和生命周期管理。所有托管节点都作为Amazon EC2自动扩展组的一部分进行配置,由Amazon EKS进行管理,所有资源包括Amazon EC2实例和自动扩展组都在您的AWS帐户中运行。Amazon EKS为托管节点组资源打标签,以便Kubernetes集群自动扩展器进行发现。

https://docs.aws.amazon.com/eks/latest/userguide/cluster-autoscaler.html 中的文档提供了有关设置托管节点组并部署Kubernetes集群自动扩展器的详细指导。如果您正在跨多个可用区运行有状态应用程序,并使用Amazon EBS卷作为后端,同时使用Kubernetes集群自动扩展器,您应该配置多个节点组,每个节点组范围限定在单个可用区。

*基于EC2的工作节点的集群自动扩展器日志-*
![Kubernetes集群自动扩展器日志](../images/cluster-auto-scaler.png)

当由于资源不足而无法调度Pod时,集群自动扩展器会确定集群必须扩展,并增加节点组的大小。当使用多个节点组时,集群自动扩展器会根据Expander配置选择一个节点组。目前,EKS中支持以下策略:
+ **random** - 默认扩展器,随机选择实例组
+ **most-pods** - 选择调度最多Pod的实例组
+ **least-waste** - 选择扩展后将产生最少闲置CPU(如果相等,则为未使用的内存)的节点组。当您有不同类型的节点(例如,高CPU或高内存节点)时,这很有用,因为您只想在有需要大量这些资源的待处理Pod时扩展这些资源。
+ **priority** - 选择用户分配的最高优先级的节点组

如果使用EC2 Spot实例作为工作节点,您可以对集群自动扩展器使用**random**放置策略。这是默认的扩展器,在集群必须扩展时任意选择一个节点组。random扩展器最大限度地利用了多个Spot容量池。

[**Priority**](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/expander/priority/readme.md)扩展器基于用户分配的优先级选择扩展选项。示例优先级可以是让自动扩展器首先尝试扩展Spot实例节点组,然后在无法扩展时回退到扩展按需节点组。

**most-pods**扩展器在您使用nodeSelector确保某些Pod落在某些节点上时很有用。

根据[文档](https://docs.aws.amazon.com/eks/latest/userguide/cluster-autoscaler.html),为集群自动扩展配置指定**least-waste**作为扩展器类型:

```
    spec:
      containers:
      - command:
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/<YOUR CLUSTER NAME>
        - --balance-similar-node-groups
        - --skip-nodes-with-system-pods=false
```

### 部署水平Pod自动扩展,以根据资源的CPU利用率或其他应用程序相关指标自动扩展部署、复制控制器或副本集中的Pod数量

[Kubernetes水平Pod自动扩展器](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)会根据资源指标(如CPU利用率)或自定义指标支持,根据某些其他应用程序提供的指标自动扩展部署、复制控制器或副本集中的Pod数量。这可以帮助您的应用程序在需求增加时扩展,或在资源不需要时缩小,从而释放您的工作节点供其他应用程序使用。当您设置目标指标利用率百分比时,水平Pod自动扩展器会尝试扩展或缩小您的应用程序以满足该目标。

[k8s-cloudwatch-adapter](https://github.com/awslabs/k8s-cloudwatch-adapter)是Kubernetes自定义指标API和外部指标API的一种实现,集成了CloudWatch指标。它允许您使用CloudWatch指标通过水平Pod自动扩展器(HPA)扩展Kubernetes部署。

对于使用资源指标(如CPU)进行扩展的示例,请遵循https://eksworkshop.com/beginner/080_scaling/test_hpa/部署示例应用程序,执行简单的负载测试以测试Pod自动扩展,并模拟Pod自动扩展。

请参阅[博客](https://aws.amazon.com/blogs/compute/scaling-kubernetes-deployments-with-amazon-cloudwatch-metrics/)了解根据Amazon SQS(Simple Queue Service)队列中消息数量进行扩展的自定义指标示例。

来自博客的Amazon SQS外部指标示例:

```yaml
apiVersion: metrics.aws/v1alpha1
kind: ExternalMetric:
  metadata:
    name: hello-queue-length
  spec:
    name: hello-queue-length
    resource:
      resource: "deployment"
    queries:
      - id: sqs_helloworld
        metricStat:
          metric:
            namespace: "AWS/SQS"
            metricName: "ApproximateNumberOfMessagesVisible"
            dimensions:
              - name: QueueName
                value: "helloworld"
          period: 300
          stat: Average
          unit: Count
        returnData: true
```

利用此外部指标的HPA示例:

``` yaml 
kind: HorizontalPodAutoscaler
apiVersion: autoscaling/v2beta1
metadata:
  name: sqs-consumer-scaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1beta1
    kind: Deployment
    name: sqs-consumer
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: External
    external:
      metricName: hello-queue-length
      targetAverageValue: 30
```

集群自动扩展器用于Kubernetes工作节点,以及水平Pod自动扩展器用于Pod,将确保配置的资源尽可能接近实际利用率。

![Kubernetes集群自动扩展器和HPA](../images/ClusterAS-HPA.png)
***(图片来源: https://aws.amazon.com/blogs/containers/cost-optimization-for-kubernetes-on-aws/)***

***Amazon EKS with Fargate***

****Fargate上的水平Pod自动扩展****

可以使用以下机制对EKS on Fargate进行自动扩展:

1. 使用Kubernetes指标服务器,并根据CPU和/或内存使用情况配置自动扩展。
2. 使用Prometheus和Prometheus指标适配器根据自定义指标(如HTTP流量)配置自动扩展
3. 根据App Mesh流量配置自动扩展

上述场景在["使用自定义指标自动扩展EKS on Fargate"](https://aws.amazon.com/blogs/containers/autoscaling-eks-on-fargate-with-custom-metrics/)的实践博客中有解释。

****垂直Pod自动扩展****

对于在Fargate上运行的Pod,请使用[垂直Pod自动扩展器](https://docs.aws.amazon.com/eks/latest/userguide/vertical-pod-autoscaler.html)来优化应用程序使用的CPU和内存。但是,由于更改Pod的资源分配需要重新启动Pod,因此您必须将Pod更新策略设置为Auto或Recreate,以确保正确的功能。

## 建议

### 使用下扩展在非工作时间缩小Kubernetes部署、StatefulSets和/或HorizontalPodAutoscalers。

作为控制成本的一部分,缩小未使用的资源也会对总体成本产生巨大影响。有工具如[kube-downscaler](https://github.com/hjacobs/kube-downscaler)和[Kubernetes的Descheduler](https://github.com/kubernetes-sigs/descheduler)。

**Kube-descaler**可用于在下班时间或设定的时间段内缩小Kubernetes部署。

**Kubernetes的Descheduler**根据其策略,可以找到可以移动的Pod并驱逐它们。在当前的实现中,Kubernetes的Descheduler不会重新调度被驱逐的Pod,而是依赖默认的调度器来完成这项工作。

**Kube-descaler**

*安装kube-downscaler*:
```
git clone https://github.com/hjacobs/kube-downscaler
cd kube-downscaler
kubectl apply -k deploy/
```

示例配置使用--dry-run作为安全标志来防止下扩缩放 - 删除它以启用下扩缩放器,例如通过编辑部署:
```
$ kubectl edit deploy kube-downscaler
```

部署一个nginx Pod,并将其计划在时区 - 周一至周五 09:00-17:00 Asia/Kolkata运行:
```
$ kubectl run nginx1 --image=nginx
$ kubectl annotate deploy nginx1 'downscaler/uptime=Mon-Fri 09:00-17:00 Asia/Kolkata'
```
!!! note 
    新的nginx部署适用默认宽限期15分钟,即如果当前时间不在周一至周五9-17点(亚洲/加尔各答时区),它不会立即缩小,而是在15分钟后缩小。

![Kube-down-scaler for nginx](../images/kube-down-scaler.png)

更高级的下扩缩放部署场景可在[kube-down-scaler github项目](https://github.com/hjacobs/kube-downscaler)中找到。

**Kubernetes的Descheduler**

Descheduler可以作为Job或CronJob在k8s集群内部运行。Descheduler的策略是可配置的,包括可以启用或禁用的策略。目前实现了七种策略*RemoveDuplicates*、*LowNodeUtilization*、*RemovePodsViolatingInterPodAntiAffinity*、*RemovePodsViolatingNodeAffinity*、*RemovePodsViolatingNodeTaints*、*RemovePodsHavingTooManyRestarts*和*PodLifeTime*。更多详细信息可以在[文档](https://github.com/kubernetes-sigs/descheduler)中找到。

一个示例策略,它针对节点的低CPU利用率启用了Descheduler(涵盖了欠利用和过度利用的场景),删除重启次数过多的Pod等:

``` yaml
apiVersion: "descheduler/v1alpha1"
kind: "DeschedulerPolicy"
strategies:
  "RemoveDuplicates":
     enabled: true
  "RemovePodsViolatingInterPodAntiAffinity":
     enabled: true
  "LowNodeUtilization":
     enabled: true
     params:
       nodeResourceUtilizationThresholds:
         thresholds:
           "cpu" : 20
           "memory": 20
           "pods": 20
         targetThresholds:
           "cpu" : 50
           "memory": 50
           "pods": 50
  "RemovePodsHavingTooManyRestarts":
     enabled: true
     params:
       podsHavingTooManyRestarts:
         podRestartThresholds: 100
         includingInitContainers: true
```

**集群关闭**

[集群关闭](https://github.com/kubecost/cluster-turndown)是根据自定义计划和关闭标准自动缩小和扩大Kubernetes集群支持节点的功能。此功能可用于在非工作时间减少支出和/或减少安全面。最常见的用例是在非工作时间将非生产环境(例如开发集群)缩小到零。集群关闭目前处于ALPHA版本。

集群关闭使用Kubernetes自定义资源定义来创建计划。以下计划将创建一个计划,该计划从指定的开始日期时间开始关闭,并在指定的结束日期时间重新启动(时间应采用RFC3339格式,即基于UTC偏移量的时间)。

```yaml
apiVersion: kubecost.k8s.io/v1alpha1
kind: TurndownSchedule
metadata:
  name: example-schedule
  finalizers:
  - "finalizer.kubecost.k8s.io"
spec:
  start: 2020-03-12T00:00:00Z
  end: 2020-03-12T12:00:00Z
  repeat: daily
```

### 使用LimitRanges和ResourceQuotas来通过限制在命名空间级别分配的资源量来帮助管理成本

默认情况下,容器在Kubernetes集群上运行时,计算资源是无限的。使用资源配额,集群管理员可以限制命名空间级别的资源消耗和创建。在命名空间内,Pod或容器可以消耗由命名空间的资源配额定义的CPU和内存。有一个担心是,一个Pod或容器可能会占用所有可用资源。

Kubernetes使用资源配额和限制范围来控制CPU、内存、PersistentVolumeClaims等资源的分配。ResourceQuota是在命名空间级别,而LimitRange则适用于容器级别。

***限制范围***

LimitRange是一个策略,用于限制命名空间中的资源分配(到Pod或容器)。

以下是使用限制范围设置默认内存请求和默认内存限制的示例。

``` yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
  - default:
      memory: 512Mi
    defaultRequest:
      memory: 256Mi
    type: Container
```

更多示例可在[Kubernetes文档](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/)中找到。

***资源配额***

当多个用户或团队共享一个具有固定节点数的集群时,有一个担心是一个团队可能会使用超过其公平份额的资源。资源配额是管理员解决这一问题的一个工具。

以下是如何设置命名空间中所有容器可以使用的内存和CPU总量的配额示例,方法是在ResourceQuota对象中指定配额。这指定容器必须有内存请求、内存限制、CPU请求和CPU限制,并且不应超过ResourceQuota中设置的阈值。

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mem-cpu-demo
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
```

更多示例可在[Kubernetes文档](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/)中找到。

### 使用定价模型实现有效利用

Amazon EKS的定价详细信息在[定价页面](https://aws.amazon.com/eks/pricing/)上给出。Amazon EKS on Fargate和EC2都有共同的控制平面成本。

如果您使用AWS Fargate,定价是根据从下载容器镜像开始到Amazon EKS Pod终止的时间使用的vCPU和内存资源计算的,并向上舍入到最接近的秒。最小收费为1分钟。请参阅[AWS Fargate定价页面](https://aws.amazon.com/fargate/pricing/)上的详细定价信息。

***Amazon EKS on EC2:***

Amazon EC2提供了一系列[实例类型](https://aws.amazon.com/ec2/instance-types/)来满足不同的使用场景。实例类型包括不同组合的CPU、内存、存储和网络容量,让您可以灵活地选择适合您应用程序需求的资源组合。每种实例类型包括一个或多个实例大小,允许您根据目标工作负载的要求扩展资源。

除了CPU数量、内存、处理器系列类型之外,另一个关键决策参数是[弹性网络接口(ENI)的数量](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html),这反过来会影响您可以在该EC2实例上运行的最大Pod数量。[每种EC2实例类型的最大Pod数](https://github.com/awslabs/amazon-eks-ami/blob/master/files/eni-max-pods.txt)列表保存在github上。

****按需EC2实例:****

使用[按需实例](https://aws.amazon.com/ec2/pricing/),您可以根据运行的实例按小时或秒支付计算容量费用。不需要长期承诺或预付款。

Amazon EC2 A1实例提供了显著的成本节省,非常适合受Arm生态系统广泛支持的扩展和基于Arm的工作负载。您现在可以使用Amazon Elastic Container Service for Kubernetes (EKS)在Amazon EC2 A1实例上运行容器,作为[公开开发者预览](https://github.com/aws/containers-roadmap/tree/master/preview-programs/eks-arm-preview)的一部分。Amazon ECR现在支持[多架构容器镜像](https://aws.amazon.com/blogs/containers/introducing-multi-architecture-container-images-for-amazon-ecr/),这使得从同一镜像存储库部署不同架构和操作系统的容器镜像变得更加简单。

您可以使用[AWS Simple Monthly Calculator](https://calculator.s3.amazonaws.com/index.html)或新的[定价计算器](https://calculator.aws/)获取EKS工作节点的按需EC2实例的定价。

### 使用Spot EC2实例:

Amazon [EC2 Spot实例](https://aws.amazon.com/ec2/pricing/)允许您以低于按需价格最高90%的价格请求Amazon EC2备用计算容量。

Spot实例通常非常适合无状态的容器化工作负载,因为容器和Spot实例的方法是相似的;临时和自动扩展的容量。这意味着它们都可以在不影响应用程序性能或可用性的情况下添加和删除。

您可以创建多个节点组,混合使用按需实例类型和EC2 Spot实例,以利用这两种实例类型之间的定价优势。

![按需和Spot节点组](../images/spot_diagram.png)
***(图片来源: https://ec2spotworkshops.com/using_ec2_spot_instances_with_eks/spotworkers/workers_eksctl.html)***

以下是使用eksctl创建一个使用EC2 Spot实例的节点组的示例yaml文件。在创建节点组期间,我们配置了一个节点标签,以便Kubernetes知道我们配置了什么类型的节点。我们将节点的生命周期设置为Ec2Spot。我们还使用PreferNoSchedule进行污点,以更喜欢不在Spot实例上调度Pod。这是NoSchedule的"首选"或"软"版本,即系统将尝试避免将不容忍污点的Pod放在节点上,但不是必需的。我们使用这种技术来确保只有合适的工作负载被调度到Spot实例上。

``` yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: my-cluster-testscaling 
  region: us-west-2
nodeGroups:
  - name: ng-spot
    labels:
      lifecycle: Ec2Spot
    taints:
      spotInstance: true:PreferNoSchedule
    minSize: 2
    maxSize: 5
    instancesDistribution: # 至少应指定两种实例类型
      instanceTypes:
        - m4.large
        - c4.large
        - c5.large
      onDemandBaseCapacity: 0
      onDemandPercentageAboveBaseCapacity: 0 # 所有实例都将是Spot实例
      spotInstancePools: 2
```
使用节点标签来识别节点的生命周期。
```
$ kubectl get nodes --label-columns=lifecycle --selector=lifecycle=Ec2Spot
```

我们还应该在每个Spot实例上部署[AWS Node Termination Handler](https://github.com/aws/aws-node-termination-handler)。这将监视实例上的EC2元数据服务,以获取中断通知。终止处理程序包括ServiceAccount、ClusterRole、ClusterRoleBinding和DaemonSet。AWS Node Termination Handler不仅适用于Spot实例,它还可以捕获一般的EC2维护事件,因此可以在集群的所有工作节点上使用。

如果客户多样化且使用容量优化分配策略,Spot实例将可用。您可以在清单文件中使用节点亲和性来配置这一点,以更喜欢Spot实例,但不要求它们。这将允许Pod在没有可用或正确标记的Spot实例的情况下调度到按需节点上。

``` yaml

affinity:
nodeAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 1
    preference:
      matchExpressions:
      - key: lifecycle
        operator: In
        values:
        - Ec2Spot
tolerations:
- key: "spotInstance"
operator: "Equal"
value: "true"
effect: "PreferNoSchedule"

```

您可以在[在线EC2 Spot研讨会](https://ec2spotworkshops.com/using_ec2_spot_instances_with_eks.html)上完成一个关于EC2 Spot实例的完整研讨会。

### 使用计算节省计划

计算节省计划是一种灵活的折扣模型,它提供与预留实例相同的折扣,作为在一年或三年期内使用特定金额(以美元/小时为单位)的计算能力的承诺。详细信息在[节省计划启动FAQ](https://aws.amazon.com/savingsplans/faq/)中介绍。该计划会自动应用于任何EC2工作节点,无论区域、实例系列、操作系统还是租赁,包括EKS集群的一部分。例如,您可以从C4切换到C5实例,将工作负载从都柏林移到伦敦,同时一路享受节省计划价格,而无需做任何事情。

AWS Cost Explorer将帮助您选择节省计划,并指导您完成购买过程。
![计算节省计划](../images/Compute-savings-plan.png)

注意 - 计算节省计划现在也适用于[AWS Fargate for AWS Elastic Kubernetes Service (EKS)](https://aws.amazon.com/about-aws/whats-new/2020/08/amazon-fargate-aws-eks-included-compute-savings-plan/)。

注意 - 上述定价不包括数据传输费用、CloudWatch、Elastic Load Balancer等其他AWS服务,以及Kubernetes应用程序可能使用的其他AWS服务。

## 资源
请参考以下资源,了解有关成本优化最佳实践的更多信息。

### 视频
+	[AWS re:Invent 2019: 节省高达90%并在Spot实例上运行生产工作负载(CMP331-R1)](https://www.youtube.com/watch?v=7q5AeoKsGJw)

### 文档和博客
+	[Kubernetes在AWS上的成本优化](https://aws.amazon.com/blogs/containers/cost-optimization-for-kubernetes-on-aws/)
+	[使用Spot实例为EKS构建成本优化和弹性](https://aws.amazon.com/blogs/compute/cost-optimization-and-resilience-eks-with-spot-instances/)
+ [使用自定义指标自动扩展EKS on Fargate](https://aws.amazon.com/blogs/containers/autoscaling-eks-on-fargate-with-custom-metrics/)
+ [AWS Fargate注意事项](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html)
+	[将Spot实例与EKS一起使用](https://ec2spotworkshops.com/using_ec2_spot_instances_with_eks.html)
+   [扩展EKS API:托管节点组](https://aws.amazon.com/blogs/containers/eks-managed-node-groups/)
+	[使用Amazon EKS进行自动扩展](https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html) 
+	[Amazon EKS定价](https://aws.amazon.com/eks/pricing/)
+	[AWS Fargate定价](https://aws.amazon.com/fargate/pricing/)
+   [节省计划](https://docs.aws.amazon.com/savingsplans/latest/userguide/what-is-savings-plans.html)
+   [使用Kubernetes在AWS上节省云成本](https://srcco.de/posts/saving-cloud-costs-kubernetes-aws.html) 

### 工具
+  [Kube downscaler](https://github.com/hjacobs/kube-downscaler)
+  [Kubernetes Descheduler](https://github.com/kubernetes-sigs/descheduler)
+  [集群关闭](https://github.com/kubecost/cluster-turndown)