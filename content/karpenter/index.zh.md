# Karpenter 最佳实践

## Karpenter

Karpenter 是一个开源集群自动扩缩器,可自动为无法调度的 Pod 提供新节点。Karpenter 评估待处理 Pod 的总体资源需求,并选择最佳实例类型来运行它们。它将自动缩减或终止没有任何非 daemonset Pod 的实例,以减少浪费。它还支持一个整合功能,可主动移动 Pod,并删除或替换为更便宜版本的节点,以降低集群成本。

**使用 Karpenter 的原因**

在 Karpenter 推出之前,Kubernetes 用户主要依赖于 [Amazon EC2 Auto Scaling 组](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html) 和 [Kubernetes 集群自动扩缩器](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler) (CAS) 来动态调整集群的计算能力。使用 Karpenter,您无需创建数十个节点组即可获得 Karpenter 提供的灵活性和多样性。此外,Karpenter 与 Kubernetes 版本的耦合性不太紧密(与 CAS 相比),也不需要您在 AWS 和 Kubernetes API 之间跳转。

Karpenter 将实例编排职责合并到一个系统中,这种方式更简单、更稳定且对集群有意识。Karpenter 旨在克服集群自动扩缩器带来的一些挑战,提供简化的方式:

* 根据工作负载需求提供节点。
* 通过灵活的 NodePool 选项,按实例类型创建不同的节点配置。与管理许多特定的自定义节点组不同,Karpenter 可让您使用单个灵活的 NodePool 管理不同工作负载的容量。
* 通过快速启动节点和调度 Pod,实现大规模 Pod 调度的改进。

有关使用 Karpenter 的信息和文档,请访问 [karpenter.sh](https://karpenter.sh/) 网站。

## 建议

最佳实践分为 Karpenter 本身、NodePool 和 Pod 调度几个部分。

## Karpenter 最佳实践

以下最佳实践涵盖了与 Karpenter 本身相关的主题。

### 对容量需求变化的工作负载使用 Karpenter

与 [Auto Scaling 组](https://aws.amazon.com/blogs/containers/amazon-eks-cluster-multi-zone-auto-scaling-groups/) (ASG) 和 [Managed Node Group](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html) (MNG) 相比,Karpenter 将扩缩管理更接近 Kubernetes 原生 API。ASG 和 MNG 是 AWS 原生抽象,其中扩缩触发基于 AWS 级别的指标,如 EC2 CPU 负载。[集群自动扩缩器](https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html#cluster-autoscaler) 将 Kubernetes 抽象桥接到 AWS 抽象,但由于这种方式失去了一些灵活性,例如为特定可用区域调度。
Karpenter 移除了一层 AWS 抽象层,将一些灵活性直接带入 Kubernetes。Karpenter 最适合用于工作负载遇到高峰期需求或具有不同计算需求的集群。MNG 和 ASG 更适合运行工作负载相对静态和一致的集群。根据您的需求,您可以混合使用动态和静态管理的节点。

### 在以下情况时考虑其他自动扩缩项目...

如果您需要 Karpenter 尚未提供的功能,请暂时考虑其他自动扩缩项目。由于 Karpenter 是一个相对较新的项目,如果您需要 Karpenter 目前尚未包含的功能,请暂时考虑其他自动扩缩项目。

### 在 EKS Fargate 或属于节点组的工作节点上运行 Karpenter 控制器

Karpenter 使用 [Helm 图表](https://karpenter.sh/docs/getting-started/)进行安装。Helm 图表会安装 Karpenter 控制器和一个 Webhook Pod 作为 Deployment,在使用 Karpenter 进行集群扩缩之前,这些组件需要先运行。我们建议至少配置一个小型节点组,并至少包含一个工作节点。或者,您也可以通过为 `karpenter` 命名空间创建 Fargate 配置文件,在 EKS Fargate 上运行这些 Pod。这样做会导致部署到该命名空间的所有 Pod 都在 EKS Fargate 上运行。不要在由 Karpenter 管理的节点上运行 Karpenter。

### 避免使用自定义启动模板与 Karpenter

Karpenter 强烈建议不要使用自定义启动模板。使用自定义启动模板会阻止多架构支持、自动升级节点的能力以及安全组发现。使用启动模板也可能导致混淆,因为 Karpenter 的 NodePool 中某些字段是重复的,而其他字段(如子网和实例类型)则会被 Karpenter 忽略。

您通常可以通过使用自定义用户数据和/或直接在 AWS 节点模板中指定自定义 AMI 来避免使用启动模板。如何执行此操作的更多信息,请参阅 [NodeClasses](https://karpenter.sh/docs/concepts/nodeclasses/)。

### 排除不适合您工作负载的实例类型

如果您的集群中运行的工作负载不需要某些特定实例类型,请考虑使用 [node.kubernetes.io/instance-type](http://node.kubernetes.io/instance-type) 键将其排除。

以下示例展示了如何避免配置大型 Graviton 实例。
```yaml

- key: node.kubernetes.io/instance-type

  operator: NotIn

  values:

  - m6g.16xlarge

  - m6gd.16xlarge

  - r6g.16xlarge

  - r6gd.16xlarge

  - c6g.16xlarge

```

### 启用使用 Spot 实例时的中断处理

Karpenter 支持 [原生中断处理](https://karpenter.sh/docs/concepts/disruption/#interruption)，通过 `--interruption-queue-name` CLI 参数启用，并提供 SQS 队列名称。中断处理会监视即将发生的非自愿中断事件,这些事件会导致您的工作负载中断,例如:

* Spot 实例中断警告
* 计划的变更健康事件(维护事件)
* 实例终止事件
* 实例停止事件

当 Karpenter 检测到您的节点将发生上述事件之一时,它会自动封锁、排空和终止节点,以在中断事件发生前为工作负载清理提供最大时间。不建议与 Karpenter 一起使用 AWS Node Termination Handler,原因如 [此处](https://karpenter.sh/docs/faq/#interruption-handling) 所述。

需要检查点或其他形式的优雅排空(在关闭前需要 2 分钟)的 Pod 应在其集群中启用 Karpenter 中断处理。

### **无法访问互联网的 Amazon EKS 私有集群**

在将 EKS 集群配置到没有路由到互联网的 VPC 时,您必须确保已根据 EKS 文档中的私有集群 [要求](https://docs.aws.amazon.com/eks/latest/userguide/private-clusters.html#private-cluster-requirements) 配置了环境。此外,您还需要确保在 VPC 中创建了 STS VPC 区域端点。否则,您将看到类似于下面显示的错误。
```console

{"level":"FATAL","time":"2024-02-29T14:28:34.392Z","logger":"controller","message":"Checking EC2 API connectivity, WebIdentityErr: failed to retrieve credentials\ncaused by: RequestError: send request failed\ncaused by: Post \"https://sts.<region>.amazonaws.com/\": dial tcp 54.239.32.126:443: i/o timeout","commit":"596ea97"}

```

以下是翻译后的文本:

这些更改在私有集群中是必需的,因为 Karpenter 控制器使用 IAM 角色服务账户 (IRSA)。配置了 IRSA 的 Pod 通过调用 AWS 安全令牌服务 (AWS STS) API 获取凭证。如果没有出站互联网访问权限,您必须在 VPC 中创建并使用 ***AWS STS VPC 终端节点***。

私有集群还需要您创建 ***SSM 的 VPC 终端节点***。当 Karpenter 尝试供应新节点时,它会查询启动模板配置和 SSM 参数。如果您的 VPC 中没有 SSM VPC 终端节点,将导致以下错误:
```console

{"level":"ERROR","time":"2024-02-29T14:28:12.889Z","logger":"controller","message":"Unable to hydrate the AWS launch template cache, RequestCanceled: request context canceled\ncaused by: context canceled","commit":"596ea97","tag-key":"karpenter.k8s.aws/cluster","tag-value":"eks-workshop"}

...

{"level":"ERROR","time":"2024-02-29T15:08:58.869Z","logger":"controller.nodeclass","message":"discovering amis from ssm, getting ssm parameter \"/aws/service/eks/optimized-ami/1.27/amazon-linux-2/recommended/image_id\", RequestError: send request failed\ncaused by: Post \"https://ssm.<region>.amazonaws.com/\": dial tcp 67.220.228.252:443: i/o timeout","commit":"596ea97","ec2nodeclass":"default","query":"/aws/service/eks/optimized-ami/1.27/amazon-linux-2/recommended/image_id"}

```

没有 ***针对 [价格列表查询 API](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/using-pelong.html) 的 VPC 终端节点***。

因此,定价数据会随着时间的推移而过时。

Karpenter 通过在其二进制文件中包含按需定价数据来解决此问题,但仅在升级 Karpenter 时才会更新该数据。

对定价数据的失败请求将导致以下错误消息:
```console

{"level":"ERROR","time":"2024-02-29T15:08:58.522Z","logger":"controller.pricing","message":"retreiving on-demand pricing data, RequestError: send request failed\ncaused by: Post \"https://api.pricing.<region>.amazonaws.com/\": dial tcp 18.196.224.8:443: i/o timeout; RequestError: send request failed\ncaused by: Post \"https://api.pricing.<region>.amazonaws.com/\": dial tcp 18.185.143.117:443: i/o timeout","commit":"596ea97"}

```

总而言之,要在完全私有的 EKS 集群中使用 Karpenter,您需要创建以下 VPC 终端节点:

. 对于 AWS STS,创建 STS VPC 终端节点。
. 对于 Amazon EC2 Systems Manager (SSM),创建 SSM VPC 终端节点。
. 对于 AWS 价格列表查询 API,创建价格列表查询 API VPC 终端节点。
```console

com.amazonaws.<region>.ec2

com.amazonaws.<region>.ecr.api

com.amazonaws.<region>.ecr.dkr

com.amazonaws.<region>.s3 – For pulling container images

com.amazonaws.<region>.sts – For IAM roles for service accounts

com.amazonaws.<region>.ssm - For resolving default AMIs

com.amazonaws.<region>.sqs - For accessing SQS if using interruption handling

```

!!! 注意

    Karpenter (控制器和 Webhook 部署)容器镜像必须存在于或复制到 Amazon ECR 私有镜像库或其他可从 VPC 内部访问的私有镜像库中。原因是 Karpenter 控制器和 Webhook Pod 当前使用公共 ECR 镜像。如果这些镜像无法从 VPC 内部或与 VPC 对等的网络访问,当 Kubernetes 尝试从 ECR 公共镜像库拉取这些镜像时,您将遇到镜像拉取错误。

有关更多信息,请参阅 [Issue 988](https://github.com/aws/karpenter/issues/988) 和 [Issue 1157](https://github.com/aws/karpenter/issues/1157)。

## 创建 NodePool

以下最佳实践涵盖了与创建 NodePool 相关的主题。

### 在以下情况下创建多个 NodePool...

当不同团队共享一个集群并需要在不同的工作节点上运行其工作负载,或者具有不同的操作系统或实例类型要求时,请创建多个 NodePool。例如,一个团队可能希望使用 Bottlerocket,而另一个团队可能希望使用 Amazon Linux。同样,一个团队可能有权访问昂贵的 GPU 硬件,而另一个团队则不需要。使用多个 NodePool 可确保每个团队都可获得最合适的资源。

### 创建相互排斥或加权的 NodePool

建议创建相互排斥或加权的 NodePool,以提供一致的调度行为。如果它们不是这种情况,并且多个 NodePool 匹配,Karpenter 将随机选择使用哪一个,从而导致意外结果。创建多个 NodePool 的有用示例包括以下内容:

创建一个带有 GPU 的 NodePool,并仅允许特殊工作负载在这些(昂贵的)节点上运行:
```yaml

# NodePool for GPU Instances with Taints

apiVersion: karpenter.sh/v1beta1

kind: NodePool

metadata:

  name: gpu

spec:

  disruption:

    consolidateAfter: 1m0s

    consolidationPolicy: WhenEmpty

    expireAfter: Never

  template:

    metadata: {}

    spec:

      nodeClassRef:

        name: default

      requirements:

      - key: node.kubernetes.io/instance-type

        operator: In

        values:

        - p3.8xlarge

        - p3.16xlarge

      - key: kubernetes.io/os

        operator: In

        values:

        - linux

      - key: kubernetes.io/arch

        operator: In

        values:

        - amd64

      - key: karpenter.sh/capacity-type

        operator: In

        values:

        - on-demand

      taints:

      - effect: NoSchedule

        key: nvidia.com/gpu

        value: "true"

```

以下是将上述文本翻译为简体中文的结果:

# Karpenter 最佳实践

## 部署时容忍污点:
```yaml

# Deployment of GPU Workload will have tolerations defined

apiVersion: apps/v1

kind: Deployment

metadata:

  name: inflate-gpu

spec:

  ...

    spec:

      tolerations:

      - key: "nvidia.com/gpu"

        operator: "Exists"

        effect: "NoSchedule"

```

对于一般部署给其他团队,NodePool 规范可以包括 nodeAffinify。然后,Deployment 可以使用 nodeSelectorTerms 来匹配 `billing-team`。

对于一般部署给其他团队,NodePool 规范可以包括 nodeAffinity。然后,Deployment 可以使用 nodeSelectorTerms 来匹配 `billing-team`。
```yaml

# NodePool for regular EC2 instances

apiVersion: karpenter.sh/v1beta1

kind: NodePool

metadata:

  name: generalcompute

spec:

  disruption:

    expireAfter: Never

  template:

    metadata:

      labels:

        billing-team: my-team

    spec:

      nodeClassRef:

        name: default

      requirements:

      - key: node.kubernetes.io/instance-type

        operator: In

        values:

        - m5.large

        - m5.xlarge

        - m5.2xlarge

        - c5.large

        - c5.xlarge

        - c5a.large

        - c5a.xlarge

        - r5.large

        - r5.xlarge

      - key: kubernetes.io/os

        operator: In

        values:

        - linux

      - key: kubernetes.io/arch

        operator: In

        values:

        - amd64

      - key: karpenter.sh/capacity-type

        operator: In

        values:

        - on-demand

```

以下是翻译后的文本:

# 使用nodeAffinity进行部署:

```yaml
# 具有定义的容忍度的GPU工作负载部署

apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate-gpu
spec:
  ...
    spec:
      tolerations:
      - key: "nvidia.com/gpu"
        operator: "Exists"
        effect: "NoSchedule"
```

对于提供给其他团队的一般部署,NodePool规范可以包括nodeAffinity。然后,Deployment可以使用nodeSelectorTerms来匹配`billing-team`。

```yaml
# 用于常规EC2实例的NodePool

apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: generalcompute
spec:
  disruption:
    expireAfter: Never
  template:
    metadata:
      labels:
        billing-team: my-team
    spec:
      nodeClassRef:
        name: default
      requirements:
      - key: node.kubernetes.io/instance-type
        operator: In
        values:
        - m5.large
        - m5.xlarge
        - m5.2xlarge
        - c5.large
        - c5.xlarge
        - c5a.large
        - c5a.xlarge
        - r5.large
        - r5.xlarge
      - key: kubernetes.io/os
        operator: In
        values:
        - linux
      - key: kubernetes.io/arch
        operator: In
        values:
        - amd64
      - key: karpenter.sh/capacity-type
        operator: In
        values:
        - on-demand
```
```yaml

# Deployment will have spec.affinity.nodeAffinity defined

kind: Deployment

metadata:

  name: workload-my-team

spec:

  replicas: 200

  ...

    spec:

      affinity:

        nodeAffinity:

          requiredDuringSchedulingIgnoredDuringExecution:

            nodeSelectorTerms:

              - matchExpressions:

                - key: "billing-team"

                  operator: "In"

                  values: ["my-team"]

```

以下是将给定的 Markdown 文本翻译为简体中文的结果:

### 使用定时器(TTL)自动从集群中删除节点

您可以在预配置的节点上使用定时器来设置何时删除没有工作负载 Pod 或已达到过期时间的节点。节点过期可用作升级的一种方式,以便淘汰旧节点并用更新版本替换。有关使用 `spec.disruption.expireAfter` 配置节点过期的信息,请参阅 Karpenter 文档中的[过期](https://karpenter.sh/docs/concepts/disruption/)。

### 避免过度限制 Karpenter 可以预配置的实例类型,尤其是在使用 Spot 时

在使用 Spot 时,Karpenter 使用 [价格容量优化](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-fleet-allocation-strategy.html) 分配策略来预配置 EC2 实例。该策略指示 EC2 从最深的池中预配置您要启动的实例数量,并且具有最低的中断风险。然后,EC2 Fleet 会从这些池中价格最低的池请求 Spot 实例。您允许 Karpenter 使用的实例类型越多,EC2 就越能优化您的 Spot 实例的运行时间。默认情况下,Karpenter 将使用 EC2 在您的集群所在的区域和可用区域中提供的所有实例类型。Karpenter 会根据待处理的 Pod 智能地从所有实例类型集合中进行选择,以确保您的 Pod 被调度到适当大小和配置的实例上。例如,如果您的 Pod 不需要 GPU,Karpenter 就不会将您的 Pod 调度到支持 GPU 的 EC2 实例类型上。当您不确定要使用哪些实例类型时,可以运行 Amazon [ec2-instance-selector](https://github.com/aws/amazon-ec2-instance-selector) 来生成符合您计算需求的实例类型列表。例如,该 CLI 以内存、vCPU、架构和区域作为输入参数,并为您提供满足这些约束条件的 EC2 实例列表。
```console

$ ec2-instance-selector --memory 4 --vcpus 2 --cpu-architecture x86_64 -r ap-southeast-1

c5.large

c5a.large

c5ad.large

c5d.large

c6i.large

t2.medium

t3.medium

t3a.medium

```

以下是将给定的 Markdown 文本翻译为简体中文的结果:

您不应对使用 Spot 实例的 Karpenter 施加太多限制,因为这样做可能会影响您的应用程序的可用性。例如,假设某种特定类型的所有实例都被回收,且没有合适的替代实例可用于替换它们。您的 Pod 将保持挂起状态,直到为配置的实例类型补充了 Spot 容量。您可以通过跨不同可用区域分布实例来降低容量不足错误的风险,因为不同可用区域的 Spot 池是不同的。也就是说,使用 Spot 时的一般最佳实践是允许 Karpenter 使用多种实例类型。

## 调度 Pod

以下最佳实践与使用 Karpenter 进行节点供应时在集群中部署 Pod 有关。

### 遵循 EKS 高可用性最佳实践

如果您需要运行高度可用的应用程序,请遵循一般 EKS 最佳实践[建议](https://aws.github.io/aws-eks-best-practices/reliability/docs/application/#recommendations)。有关如何跨节点和区域分布 Pod 的详细信息,请参阅 Karpenter 文档中的[拓扑扩展](https://karpenter.sh/docs/concepts/scheduling/#topology-spread)。使用[中断预算](https://karpenter.sh/docs/troubleshooting/#disruption-budgets)设置在尝试逐出或删除 Pod 时需要维护的最小可用 Pod 数量。

### 使用分层约束来限制您的云提供商提供的计算功能

Karpenter 的分层约束模型允许您创建一组复杂的 NodePool 和 Pod 部署约束,以获得 Pod 调度的最佳匹配。Pod 规范可以请求的约束示例包括以下内容:

* 需要在只有特定应用程序可用的可用区域中运行。例如,假设您有一个 Pod 需要与位于特定可用区域的 EC2 实例上运行的另一个应用程序进行通信。如果您的目标是减少 VPC 中的跨可用区域流量,您可能希望将 Pod 与该 EC2 实例位于同一可用区域。这种定位通常是使用节点选择器来实现的。有关[节点选择器](https://karpenter.sh/docs/concepts/scheduling/#selecting-nodes)的更多信息,请参阅 Kubernetes 文档。

* 需要特定类型的处理器或其他硬件。请参阅 Karpenter 文档中的[加速器](https://karpenter.sh/docs/concepts/scheduling/#acceleratorsgpu-resources)部分,了解需要在 GPU 上运行的 Pod 规范示例。

### 创建计费警报以监控您的计算支出
以下是将给定的 Markdown 文本翻译为简体中文的结果:

当您将集群配置为自动扩缩容时,您应该创建计费警报以在支出超过阈值时发出警告,并在 Karpenter 配置中添加资源限制。使用 Karpenter 设置资源限制类似于设置 AWS 自动扩缩组的最大容量,它代表 Karpenter NodePool 可以实例化的最大计算资源量。

!!! 注意
    无法为整个集群设置全局限制。限制适用于特定的 NodePool。

下面的代码片段告诉 Karpenter 最多只能提供 1000 个 CPU 内核和 1000Gi 内存。只有在达到或超过限制时,Karpenter 才会停止添加容量。当超过限制时,Karpenter 控制器将在控制器的日志中写入"内存资源使用量 1001 超过限制 1000"或类似的消息。如果您将容器日志路由到 CloudWatch 日志,您可以创建[指标过滤器](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/MonitoringLogData.html)来查找日志中的特定模式或术语,然后创建[CloudWatch 警报](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html)以在您配置的指标阈值被违反时提醒您。

有关使用 Karpenter 设置限制的更多信息,请参阅 Karpenter 文档中的[设置资源限制](https://karpenter.sh/docs/concepts/nodepools/#speclimits)。
```yaml

spec:

  limits:

    cpu: 1000

    memory: 1000Gi

```

如果您不使用限制或约束 Karpenter 可以预配置的实例类型,Karpenter 将根据需求继续为集群添加计算容量。虽然以这种方式配置 Karpenter 允许您的集群自由扩缩,但也可能会产生重大的成本影响。正因如此,我们建议配置计费警报。计费警报允许您在账户中的估计费用超过定义的阈值时得到主动通知和警报。有关详细信息,请参阅[设置 Amazon CloudWatch 计费警报以主动监控估计费用](https://aws.amazon.com/blogs/mt/setting-up-an-amazon-cloudwatch-billing-alarm-to-proactively-monitor-estimated-charges/)。

您还可能希望启用成本异常检测,这是一项 AWS 成本管理功能,使用机器学习持续监控您的成本和使用情况,以检测异常支出。更多信息可以在 [AWS 成本异常检测入门指南](https://docs.aws.amazon.com/cost-management/latest/userguide/getting-started-ad.html)中找到。如果您已经在 AWS Budgets 中创建了预算,您还可以配置操作以在违反特定阈值时通知您。通过预算操作,您可以发送电子邮件、发布到 SNS 主题或向聊天机器人(如 Slack)发送消息。有关更多信息,请参阅[配置 AWS Budgets 操作](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-controls.html)。

### 使用 karpenter.sh/do-not-disrupt 注解防止 Karpenter 逐出节点

如果您在 Karpenter 预配置的节点上运行关键应用程序(如长时间运行的批处理作业或有状态应用程序),并且节点的 TTL 已过期,则当实例终止时,应用程序将被中断。通过向 Pod 添加 `karpenter.sh/karpenter.sh/do-not-disrupt` 注解,您指示 Karpenter 保留该节点,直到 Pod 终止或删除 `karpenter.sh/do-not-disrupt` 注解。有关更多信息,请参阅[中断](https://karpenter.sh/docs/concepts/disruption/#node-level-controls)文档。

如果节点上仅剩下与作业关联的非 daemonset Pod,Karpenter 能够针对并终止这些节点,只要作业状态为成功或失败。

### 对于使用整合功能的所有非 CPU 资源,请配置 requests=limits
以下是将给定的 Markdown 文本翻译为简体中文的结果:

Karpenter 的整合功能通过将 Pod 重新打包到更合适的节点上来优化集群成本。但是,如果 Pod 请求的资源少于其限制,则在重新打包期间,可能会出现资源不足的情况。为了避免这种情况,请确保对于所有非 CPU 资源(如内存、GPU 等),将 requests 设置为等于 limits。

对于 CPU 资源,Kubernetes 默认将 requests 设置为 limits 的值。但是,对于其他资源类型,如果未指定 requests,则默认为 0。因此,最佳做法是为所有资源类型显式设置 requests 和 limits。

以下是一个示例 Pod 规范,其中内存 requests 和 limits 设置为 1Gi:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: app
    image: images.my-company.example/app:v5
    resources:
      requests:
        memory: 1Gi
      limits:
        memory: 1Gi
```

### 使用 LimitRanges 为资源请求和限制配置默认值

由于 Kubernetes 不设置默认的请求或限制,容器从底层主机消耗的 CPU 和内存资源是无限制的。Kubernetes 调度程序查看 Pod 的总请求(来自 Pod 容器或 Pod 的 Init 容器的总请求中的较高值)来确定将 Pod 调度到哪个工作节点上。同样,Karpenter 也会考虑 Pod 的请求来确定要提供哪种类型的实例。您可以使用限制范围为命名空间应用合理的默认值,以防某些 Pod 未指定资源请求。

请参阅[为命名空间配置默认内存请求和限制](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/)。

### 为所有工作负载应用精确的资源请求

当 Karpenter 对您的工作负载需求有准确的信息时,它就能够为您的工作负载启动最合适的节点。这一点对于使用 Karpenter 的整合功能尤为重要。

请参阅[为所有工作负载配置和调整资源请求/限制](https://aws.github.io/aws-eks-best-practices/reliability/docs/dataplane/#configure-and-size-resource-requestslimits-for-all-workloads)。

## CoreDNS 建议

### 更新 CoreDNS 配置以保持可靠性

在将 CoreDNS Pod 部署到由 Karpenter 管理的节点上时,鉴于 Karpenter 动态快速终止/创建新节点以满足需求的特性,建议遵循以下最佳实践:

[CoreDNS lameduck 持续时间](https://aws.github.io/aws-eks-best-practices/scalability/docs/cluster-services/#coredns-lameduck-duration)

[CoreDNS 就绪探针](https://aws.github.io/aws-eks-best-practices/scalability/docs/cluster-services/#coredns-readiness-probe)

这将确保 DNS 查询不会被路由到尚未就绪或已被终止的 CoreDNS Pod。

## Karpenter 蓝图

这是将给定的 Markdown 文本翻译为简体中文的结果。我只需要翻译后的文本作为回复,不要添加任何额外的代码块。此外,这里有一些之前已经翻译的文本:

# Karpenter 最佳实践

## Karpenter

Karpenter 是一个开源集群自动扩缩器,可自动为无法调度的 Pod 提供新节点。Karpenter 评估待处理 Pod 的总体资源需求,并选择最佳实例类型来运行它们。它将自动缩减或终止没有任何非 daemonset Pod 的实例,以减少浪费。它还支持一个整合功能,可主动移动 Pod,并删除或替换为更便宜版本的节点,以降低集群成本。

**使用 Karpenter 的原因**

在 Karpenter 推出之前,Kubernetes 用户主要依赖于 [Amazon EC2 Auto Scaling 组](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html) 和 [Kubernetes 集群自动扩缩器](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler) (CAS) 来动态调整集群的计算能力。使用 Karpenter,您无需创建数十个节点组即可获得 Karpenter 提供的灵活性和多样性。此外,Karpenter 与 Kubernetes 版本的耦合性不太紧密(与 CAS 相比),也不需要您在 AWS 和 Kubernetes API 之间跳转。

Karpenter 将实例编排职责合并到一个系统中,这种方式更简单、更稳定且对集群有意识。Karpenter 旨在克服集群自动扩缩器带来的一些挑战,提供简化的方式:

* 根据工作负载需求提供节点。
* 通过灵活的 NodePool 选项,按实例类型创建不同的节点配置。与管理许多特定的自定义节点组不同,Karpenter 可让您使用单个灵活的 NodePool 管理不同工作负载的容量。
* 通过快速启动节点和调度 Pod,实现大规模 Pod 调度的改进。

有关使用 Karpenter 的信息和文档,请访问 [karpenter.sh](https://karpenter.sh/) 网站。

## 建议

最佳实践分为 Karpenter 本身、NodePool 和 Pod 调度几个部分。

## Karpenter 最佳实践

以下最佳实践涵盖了与 Karpenter 本身相关的主题。

### 对容量需求变化的工作负载使用 Karpenter

与 [Auto Scaling 组](https://aws.amazon.com/blogs/containers/amazon-eks-cluster-multi-zone-auto-scaling-groups/) (ASG) 和 [Managed Node Group](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html) (MNG) 相比,Karpenter 将扩缩管理更接近 Kubernetes 原生 API。ASG 和 MNG 是 AWS 原生抽象,其中扩缩触发基于 AWS 级别的指标,如 EC2 CPU 负载。[集群自动扩缩器](https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html#cluster-autoscaler) 将 Kubernetes 抽象桥接到 AWS 抽象,但由于这种方式失去了一些灵活性,例如为特定可用区域调度。
Karpenter 移除了一层 AWS 抽象层,将一些灵活性直接带入 Kubernetes。Karpenter 最适合用于工作负载遇到高峰期需求或具有不同计算需求的集群。MNG 和 ASG 更适合运行工作负载相对静态和一致的集群。根据您的需求,您可以混合使用动态和静态管理的节点。

### 在以下情况时考虑其他自动扩缩项目...

如果您需要 Karpenter 尚未提供的功能,请暂时考虑其他自动扩缩项目。由于 Karpenter 是一个相对较新的项目,如果您需要 Karpenter 目前尚未包含的功能,请暂时考虑其他自动扩缩项目。

### 在 EKS Fargate 或属于节点组的工作节点上运行 Karpenter 控制器

Karpenter 使用 [Helm 图表](https://karpenter.sh/docs/getting-started/)进行安装。Helm 图表会安装 Karpenter 控制器和一个 Webhook Pod 作为 Deployment,在使用 Karpenter 进行集群扩缩之前,这些组件需要先运行。我们建议至少配置一个小型节点组,并至少包含一个工作节点。或者,您也可以通过为 `karpenter` 命名空间创建 Fargate 配置文件,在 EKS Fargate 上运行这些 Pod。这样做会导致部署到该命名空间的所有 Pod 都在 EKS Fargate 上运行。不要在由 Karpenter 管理的节点上运行 Karpenter。

### 避免使用自定义启动模板与 Karpenter

Karpenter 强烈建议不要使用自定义启动模板。使用自定义启动模板会阻止多架构支持、自动升级节点的能力以及安全组发现。使用启动模板也可能导致混淆,因为 Karpenter 的 NodePool 中某些字段是重复的,而其他字段(如子网和实例类型)则会被 Karpenter 忽略。

您通常可以通过使用自定义用户数据和/或直接在 AWS 节点模板中指定自定义 AMI 来避免使用启动模板。如何执行此操作的更多信息,请参阅 [NodeClasses](https://karpenter.sh/docs/concepts/nodeclasses/)。

### 排除不适合您工作负载的实例类型

如果您的集群中运行的工作负载不需要某些特定实例类型,请考虑使用 [node.kubernetes.io/instance-type](http://node.kubernetes.io/instance-type) 键将其排除。

以下示例展示了如何避免配置大型 Graviton 实例。
```yaml

- key: node.kubernetes.io/instance-type

  operator: NotIn

  values:

  - m6g.16xlarge

  - m6gd.16xlarge

  - r6g.16xlarge

  - r6gd.16xlarge

  - c6g.16xlarge

```

### 启用使
以下是将给定的 Markdown 文本翻译为简体中文的结果:

随着 Karpenter 采用面向应用程序的方法来为 Kubernetes 数据平面提供计算能力,您可能想知道如何正确配置一些常见的工作负载场景。[Karpenter 蓝图](https://github.com/aws-samples/karpenter-blueprints)是一个存储库,其中包含了一系列遵循此处描述的最佳实践的常见工作负载场景。您将拥有所需的所有资源,甚至可以创建一个已配置 Karpenter 的 EKS 集群,并测试存储库中包含的每个蓝图。您可以组合不同的蓝图来最终为您的工作负载创建所需的蓝图。

## 其他资源

* [Karpenter/Spot 研讨会](https://ec2spotworkshops.com/karpenter.html)

* [Karpenter 节点提供程序](https://youtu.be/_FXRIKWJWUk)

* [TGIK Karpenter](https://youtu.be/zXqrNJaTCrU)

* [Karpenter 与集群自动扩缩器](https://youtu.be/3QsVRHVdOnM)

* [Karpenter 无组自动扩缩](https://www.youtube.com/watch?v=43g8uPohTgc)

* [教程:使用 Amazon EC2 Spot 和 Karpenter 以更低成本运行 Kubernetes 集群](https://community.aws/tutorials/run-kubernetes-clusters-for-less-with-amazon-ec2-spot-and-karpenter#step-6-optional-simulate-spot-interruption)
