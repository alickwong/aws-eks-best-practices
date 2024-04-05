!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

# 成本效益资源
成本效益资源意味着使用适当的服务、资源和配置来运行您的 Kubernetes 集群工作负载,从而实现成本节省。

## 建议
### 确保用于部署容器化服务的基础设施与应用程序配置文件和扩展需求相匹配

Amazon EKS 支持几种类型的 Kubernetes 自动扩展 - [集群自动扩展器](https://docs.aws.amazon.com/eks/latest/userguide/cluster-autoscaler.html)、[水平 Pod 自动扩展器](https://docs.aws.amazon.com/eks/latest/userguide/horizontal-pod-autoscaler.html)和[垂直 Pod 自动扩展器](https://docs.aws.amazon.com/eks/latest/userguide/vertical-pod-autoscaler.html)。本节涵盖了其中两个,即集群自动扩展器和水平 Pod 自动扩展器。

### 使用集群自动扩展器调整 Kubernetes 集群的大小以满足当前需求

[Kubernetes 集群自动扩展器](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)会自动调整 EKS 集群中的节点数量,以满足由于资源不足而无法启动 pod 或集群中节点利用率低且其 pod 可以重新调度到集群中其他节点的需求。集群自动扩展器会在指定的自动扩展组内扩展工作节点,并作为部署在您的 EKS 集群中运行。

Amazon EKS 与 EC2 托管节点组自动执行 Amazon EKS Kubernetes 集群节点(Amazon EC2 实例)的配置和生命周期管理。所有托管节点都作为 Amazon EC2 自动扩展组的一部分进行配置,由 Amazon EKS 为您管理,所有资源包括 Amazon EC2 实例和自动扩展组都在您的 AWS 帐户中运行。Amazon EKS 为托管节点组资源打标签,以便 Kubernetes 集群自动扩展器进行发现。

https://docs.aws.amazon.com/eks/latest/userguide/cluster-autoscaler.html 上的文档提供了有关设置托管节点组并部署 Kubernetes 集群自动扩展器的详细指导。如果您正在跨多个可用区运行有状态应用程序,该应用程序由 Amazon EBS 卷支持,并使用 Kubernetes 集群自动扩展器,您应该配置多个节点组,每个节点组范围限定在单个可用区。

*基于 EC2 的工作节点的集群自动扩展器日志 -*
![Kubernetes 集群自动扩展器日志](../images/cluster-auto-scaler.png)

当由于资源不足而无法调度 pod 时,集群自动扩展器会确定集群必须扩展,并增加节点组的大小。当使用多个节点组时,集群自动扩展器会根据扩展器配置选择一个。目前,EKS 支持以下策略:
+ **random** - 默认扩展器,随机选择实例组
+ **most-pods** - 选择可调度最多 pod 的实例组
[**最少浪费**] - 选择将在扩容后拥有最少空闲 CPU（如果并列，则选择未使用内存最少的）的节点组。当您有不同类型的节点（例如高 CPU 或高内存节点）时,这很有用,您只想在有需要大量这些资源的待处理 pod 时扩展这些节点。
[**优先级**] - 选择用户分配的最高优先级的节点组

您可以对集群自动缩放器中的扩展器使用[**随机**]放置策略,如果工作节点使用的是 EC2 Spot 实例。这是默认的扩展器,当集群必须扩展时,它会任意选择一个节点组。随机扩展器可以最大限度地利用多个 Spot 容量池。

[**优先级**]扩展器根据用户分配给扩展组的优先级来选择扩展选项。示例优先级可以是让自动缩放器先尝试扩展 Spot 实例节点组,然后如果无法扩展,则退回到扩展按需节点组。

[**最多 pod**]扩展器在您使用 nodeSelector 确保某些 pod 落在某些节点上时很有用。

根据[文档](https://docs.aws.amazon.com/eks/latest/userguide/cluster-autoscaler.html),为集群自动缩放配置指定[**最少浪费**]作为扩展器类型:
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


### 部署水平 Pod 自动缩放以自动扩展部署、副本控制器或副本集中的 Pod 数量，具体取决于该资源的 CPU 利用率或其他应用程序相关指标

[Kubernetes 水平 Pod 自动缩放器](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)会自动根据 CPU 利用率等资源指标或某些其他应用程序提供的自定义指标来扩展部署、副本控制器或副本集中的 Pod 数量。这可以帮助您的应用程序扩展以满足不断增加的需求,或在不需要资源时缩小,从而释放您的工作节点供其他应用程序使用。当您设置目标指标利用率百分比时,水平 Pod 自动缩放器会尝试扩展或缩小您的应用程序以达到该目标。

[k8s-cloudwatch-adapter](https://github.com/awslabs/k8s-cloudwatch-adapter)是 Kubernetes 自定义指标 API 和外部指标 API 的一种实现,并与 CloudWatch 指标集成。它允许您使用 CloudWatch 指标通过水平 Pod 自动缩放器(HPA)扩展 Kubernetes 部署。

要了解如何使用资源指标(如 CPU)进行扩展,请遵循 https://eksworkshop.com/beginner/080_scaling/test_hpa/ 部署示例应用程序,执行简单的负载测试以测试 Pod 自动缩放,并模拟 Pod 自动缩放。

请参考此[博客](https://aws.amazon.com/blogs/compute/scaling-kubernetes-deployments-with-amazon-cloudwatch-metrics/)了解如何使用 Amazon SQS(简单队列服务)队列中的消息数量作为自定义指标来扩展应用程序的示例。

博客中的 Amazon SQS 外部指标示例:
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

利用此外部指标的 HPA 示例：

![any text]
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


使用 Kubernetes 工作节点的集群自动缩放器和 pod 的水平 Pod 自动缩放器的组合,将确保提供的资源尽可能接近实际利用率。

![Kubernetes 集群自动缩放器和 HPA](../images/ClusterAS-HPA.png)
**（图片来源：https://aws.amazon.com/blogs/containers/cost-optimization-for-kubernetes-on-aws/）**

***Amazon EKS with Fargate***

****Pods 的水平 Pod 自动缩放****

可以使用以下机制对 Fargate 上的 EKS 进行自动缩放:

1. 使用 Kubernetes 指标服务器,并根据 CPU 和/或内存使用情况配置自动缩放。
2. 使用 Prometheus 和 Prometheus 指标适配器,根据自定义指标(如 HTTP 流量)配置自动缩放
3. 根据 App Mesh 流量配置自动缩放

上述场景在["使用自定义指标对 Fargate 上的 EKS 进行自动缩放"](https://aws.amazon.com/blogs/containers/autoscaling-eks-on-fargate-with-custom-metrics/)这篇实践性博客中有详细解释。

****垂直 Pod 自动缩放****

对于在 Fargate 上运行的 pod,请使用[垂直 Pod 自动缩放器](https://docs.aws.amazon.com/eks/latest/userguide/vertical-pod-autoscaler.html)来优化 CPU 和内存的使用。但是,由于更改 pod 的资源分配需要重新启动 pod,因此必须将 pod 更新策略设置为 Auto 或 Recreate,以确保正确的功能。

## 建议

### 在非工作时间使用向下缩放来缩小 Kubernetes 部署、StatefulSets 和/或 HorizontalPodAutoscalers。

作为控制成本的一部分,缩小未使用的资源也可以对总体成本产生巨大影响。有工具如 [kube-downscaler](https://github.com/hjacobs/kube-downscaler) 和 [Kubernetes 的 Descheduler](https://github.com/kubernetes-sigs/descheduler)。

**kube-downscaler** 可用于在工作时间之外或在设定的时间段内缩小 Kubernetes 部署。

**Kubernetes 的 Descheduler** 根据其策略,可以找到可以移动的 pod 并驱逐它们。在当前的实现中,Kubernetes 的 Descheduler 不会重新调度被驱逐的 pod,而是依赖默认的调度器来完成这项工作。

**kube-downscaler**

*kube-downscaler 的安装*:
```
git clone https://github.com/hjacobs/kube-downscaler
cd kube-downscaler
kubectl apply -k deploy/
```

该示例配置使用 --dry-run 作为安全标志以防止缩小 --- 删除它以启用缩小器,例如通过编辑部署:
```
$ kubectl edit deploy kube-downscaler
```

部署一个 nginx pod 并将其安排在时区 - 周一至周五 09:00-17:00 亚洲/加尔各答中运行:
```
$ kubectl run nginx1 --image=nginx
$ kubectl annotate deploy nginx1 'downscaler/uptime=Mon-Fri 09:00-17:00 Asia/Kolkata'
```


!!! note 
    新部署的 nginx 默认宽限期为 15 分钟,即如果当前时间不在周一至周五 9-17 点(亚洲/加尔各答时区),它不会立即缩小,而是在 15 分钟后缩小。

![Kube-down-scaler for nginx](../images/kube-down-scaler.png)

更高级的缩小部署场景可在 [kube-down-scaler github 项目](https://github.com/hjacobs/kube-downscaler)中找到。

**Kubernetes 调度器**

调度器可以作为 Job 或 CronJob 在 k8s 集群内运行。调度器的策略是可配置的,包括可以启用或禁用的策略。目前实现了七种策略:*RemoveDuplicates*、*LowNodeUtilization*、*RemovePodsViolatingInterPodAntiAffinity*、*RemovePodsViolatingNodeAffinity*、*RemovePodsViolatingNodeTaints*、*RemovePodsHavingTooManyRestarts* 和 *PodLifeTime*。更多详细信息可以在他们的[文档](https://github.com/kubernetes-sigs/descheduler)中找到。

一个示例策略,它启用了调度器对节点低 CPU 利用率的处理(涵盖了欠利用和过度利用的场景),删除重启次数过多的 pod 等:
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

[集群关闭](https://github.com/kubecost/cluster-turndown)是一种基于自定义计划和关闭标准自动缩小和扩大Kubernetes集群支持节点的功能。此功能可用于在非工作时间减少开支和/或减少安全面积。最常见的用例是在非工作时间将非生产环境(例如开发集群)缩小到零。集群关闭目前处于ALPHA版本发布。

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

使用 LimitRanges 和资源配额来帮助管理成本,通过限制在命名空间级别分配的资源量

默认情况下,容器在 Kubernetes 集群上运行时计算资源是无限的。通过资源配额,集群管理员可以限制命名空间级别的资源消耗和创建。在一个命名空间中,Pod 或容器可以消耗由该命名空间的资源配额定义的 CPU 和内存。有一个担心是,一个 Pod 或容器可能会垄断所有可用的资源。

Kubernetes 使用资源配额和限制范围来控制 CPU、内存、PersistentVolumeClaims 等资源的分配。ResourceQuota 位于命名空间级别,而 LimitRange 应用于容器级别。

***限制范围***

LimitRange 是一个策略,用于限制命名空间中 Pod 或容器的资源分配。

以下是使用 Limit Range 设置默认内存请求和默认内存限制的示例。
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

更多示例可在 [Kubernetes 文档](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/)中找到。

***资源配额***

当多个用户或团队共享一个具有固定节点数的集群时,可能会出现一个团队使用超过其公平份额资源的问题。资源配额是管理员解决此问题的一种工具。

以下是如何设置命名空间中所有容器可使用的总内存和 CPU 配额的示例,方法是在 ResourceQuota 对象中指定配额。这指定容器必须具有内存请求、内存限制、CPU 请求和 CPU 限制,并且不应超过 ResourceQuota 中设置的阈值。
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


更多示例可在 [Kubernetes 文档](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/) 中找到。

### 使用定价模型实现有效利用

Amazon EKS 的定价详情在 [定价页面](https://aws.amazon.com/eks/pricing/) 中给出。Amazon EKS on Fargate 和 EC2 都有共同的控制平面成本。

如果您使用 AWS Fargate,定价将根据从下载容器镜像开始到 Amazon EKS pod 终止期间使用的 vCPU 和内存资源计算,并向上舍入到最接近的秒数。最小收费为 1 分钟。请参阅 [AWS Fargate 定价页面](https://aws.amazon.com/fargate/pricing/) 上的详细定价信息。

***Amazon EKS on EC2:***

Amazon EC2 提供了广泛的 [实例类型](https://aws.amazon.com/ec2/instance-types/) 选择,这些实例类型针对不同的用例进行了优化。实例类型包括 CPU、内存、存储和网络容量的各种组合,让您可以灵活地选择适合应用程序的资源组合。每种实例类型包括一个或多个实例大小,使您能够根据目标工作负载的需求扩展资源。

除了 CPU 数量、内存、处理器系列类型之外,实例类型的另一个关键决策参数是 [弹性网络接口(ENI) 的数量](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html),这反过来会影响您可以在该 EC2 实例上运行的 pod 的最大数量。[每种 EC2 实例类型的最大 pod 数](https://github.com/awslabs/amazon-eks-ami/blob/master/files/eni-max-pods.txt) 列表保存在 GitHub 上。

****按需 EC2 实例:****

使用 [按需实例](https://aws.amazon.com/ec2/pricing/),您可以根据运行的实例按小时或秒付费。无需长期承诺或预付款。

Amazon EC2 A1 实例提供了显著的成本节省,非常适合受 Arm 生态系统广泛支持的扩展和基于 Arm 的工作负载。您现在可以使用 Amazon Elastic Container Service for Kubernetes (EKS) 在 Amazon EC2 A1 实例上运行容器,作为 [公开开发者预览版](https://github.com/aws/containers-roadmap/tree/master/preview-programs/eks-arm-preview) 的一部分。Amazon ECR 现在支持 [多架构容器镜像](https://aws.amazon.com/blogs/containers/introducing-multi-architecture-container-images-for-amazon-ecr/),这使得从同一镜像存储库部署不同架构和操作系统的容器镜像变得更加简单。

您可以使用 [AWS 简单月度计算器](https://calculator.s3.amazonaws.com/index.html) 或新的 [定价计算器](https://calculator.aws/) 获取 EKS 工作节点的按需 EC2 实例的定价。

### 使用 Spot EC2 实例:
[简体中文]

亚马逊 [EC2 Spot 实例](https://aws.amazon.com/ec2/pricing/)允许您以高达 On-Demand 价格 90% 的折扣请求备用亚马逊 EC2 计算容量。

Spot 实例通常非常适合无状态的容器化工作负载,因为容器和 Spot 实例的方法类似;临时性和自动扩展容量。这意味着它们都可以在遵守 SLA 的同时添加和删除,而不会影响应用程序的性能或可用性。

您可以创建多个节点组,混合使用按需实例类型和 EC2 Spot 实例,以利用这两种实例类型之间的价格优势。

![按需和 Spot 节点组](../images/spot_diagram.png)
***(图片来源: https://ec2spotworkshops.com/using_ec2_spot_instances_with_eks/spotworkers/workers_eksctl.html)***

下面给出了一个使用 eksctl 创建 EC2 Spot 实例节点组的示例 yaml 文件。在创建节点组时,我们配置了节点标签,以便 Kubernetes 知道我们配置了什么类型的节点。我们将节点的生命周期设置为 Ec2Spot。我们还使用 PreferNoSchedule 进行污点,以更倾向于不将 pod 调度到 Spot 实例上。这是 NoSchedule 的"首选"或"软"版本,即系统将尽量避免将不能容忍污点的 pod 放置在节点上,但这不是必需的。我们使用这种技术来确保只有合适的工作负载被调度到 Spot 实例上。
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
    instancesDistribution: # At least two instance types should be specified
      instanceTypes:
        - m4.large
        - c4.large
        - c5.large
      onDemandBaseCapacity: 0
      onDemandPercentageAboveBaseCapacity: 0 # all the instances will be spot instances
      spotInstancePools: 2
```

使用节点标签来识别节点的生命周期。
```
$ kubectl get nodes --label-columns=lifecycle --selector=lifecycle=Ec2Spot
```

我们还应该在每个Spot实例上部署[AWS Node Termination Handler](https://github.com/aws/aws-node-termination-handler)。这将监控实例上的EC2元数据服务是否有中断通知。终止处理程序包括ServiceAccount、ClusterRole、ClusterRoleBinding和DaemonSet。AWS Node Termination Handler不仅适用于Spot实例,它还可以捕获一般的EC2维护事件,因此可以用于集群中的所有工作节点。

如果客户多元化且使用容量优化分配策略,Spot实例将可用。您可以在清单文件中使用节点亲和性来配置这一点,以优先使用Spot实例,但不要求使用。这将允许在没有可用或正确标记的Spot实例的情况下,将pod调度到按需节点上。
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


您可以在[在线 EC2 Spot 研讨会](https://ec2spotworkshops.com/using_ec2_spot_instances_with_eks.html)上完成一个完整的 EC2 spot 实例研讨会。

### 使用计算节省计划

计算节省计划是一种灵活的折扣模式,它提供与预留实例相同的折扣,但需要您承诺在一年或三年期内使用特定金额(以美元/小时为单位)的计算能力。详情请参见[节省计划启动常见问题解答](https://aws.amazon.com/savingsplans/faq/)。这些计划会自动应用于任何 EC2 工作节点,无论区域、实例系列、操作系统还是租用类型,包括 EKS 集群的一部分。例如,您可以从 C4 实例切换到 C5 实例,将工作负载从都柏林迁移到伦敦,并在此过程中享受节省计划价格,而无需进行任何操作。

AWS Cost Explorer 将帮助您选择节省计划,并指导您完成购买过程。

![计算节省计划](../images/Compute-savings-plan.png)

注意 - 计算节省计划现在也适用于[AWS Fargate for AWS Elastic Kubernetes Service (EKS)](https://aws.amazon.com/about-aws/whats-new/2020/08/amazon-fargate-aws-eks-included-compute-savings-plan/)。

注意 - 上述定价不包括数据传输费用、CloudWatch、弹性负载均衡和 Kubernetes 应用程序可能使用的其他 AWS 服务。

## 资源
请参考以下资源,了解成本优化的最佳实践。

### 视频
+	[AWS re:Invent 2019: 在 Spot 实例上节省高达 90% 并运行生产工作负载 (CMP331-R1)](https://www.youtube.com/watch?v=7q5AeoKsGJw)

### 文档和博客
+	[Kubernetes 在 AWS 上的成本优化](https://aws.amazon.com/blogs/containers/cost-optimization-for-kubernetes-on-aws/)
+	[使用 Spot 实例为 EKS 构建成本优化和弹性](https://aws.amazon.com/blogs/compute/cost-optimization-and-resilience-eks-with-spot-instances/)
+ [使用自定义指标自动扩展 Fargate 上的 EKS](https://aws.amazon.com/blogs/containers/autoscaling-eks-on-fargate-with-custom-metrics/)
+ [AWS Fargate 注意事项](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html)
+	[在 EKS 上使用 Spot 实例](https://ec2spotworkshops.com/using_ec2_spot_instances_with_eks.html)
+   [扩展 EKS API: 托管节点组](https://aws.amazon.com/blogs/containers/eks-managed-node-groups/)
+	[Amazon EKS 自动扩展](https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html) 
+	[Amazon EKS 定价](https://aws.amazon.com/eks/pricing/)
+	[AWS Fargate 定价](https://aws.amazon.com/fargate/pricing/)
+   [节省计划](https://docs.aws.amazon.com/savingsplans/latest/userguide/what-is-savings-plans.html)
+   [使用 Kubernetes 在 AWS 上节省云成本](https://srcco.de/posts/saving-cloud-costs-kubernetes-aws.html)

### 工具
+ [Kube downscaler](https://github.com/hjacobs/kube-downscaler)
+ [Kubernetes 调度器](https://github.com/kubernetes-sigs/descheduler)
+ [集群关闭](https://github.com/kubecost/cluster-turndown)
