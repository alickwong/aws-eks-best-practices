# Karpenter 最佳实践

## Karpenter

Karpenter 是一个开源的集群自动缩放器,可自动配置新节点以响应无法调度的 pod。Karpenter 评估待处理 pod 的总体资源需求,并选择运行它们的最佳实例类型。它将自动缩小或终止没有任何非 daemonset pod 的实例,以减少浪费。它还支持一个合并功能,可以主动移动 pod 并删除或替换较便宜的节点版本,以降低集群成本。

**使用 Karpenter 的原因**

在 Karpenter 推出之前,Kubernetes 用户主要依赖于 [Amazon EC2 Auto Scaling 组](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html)和 [Kubernetes 集群自动缩放器](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler) (CAS) 来动态调整其集群的计算容量。使用 Karpenter,您无需创建数十个节点组来实现 Karpenter 提供的灵活性和多样性。此外,Karpenter 与 Kubernetes 版本的耦合性不如 CAS 那么紧密,您也不需要在 AWS 和 Kubernetes API 之间来回跳转。

Karpenter 将实例编排职责集中在一个系统中,这更简单、更稳定和更了解集群。Karpenter 旨在克服集群自动缩放器提出的一些挑战,提供以下简化方式:

* 根据工作负载需求配置节点。
* 通过实例类型创建多样化的节点配置,使用灵活的 NodePool 选项。与管理许多特定的自定义节点组相比,Karpenter 可以让您使用单个灵活的 NodePool 管理多样化的工作负载容量。
* 通过快速启动节点和调度 pod 来实现更好的 pod 调度。

有关使用 Karpenter 的信息和文档,请访问 [karpenter.sh](https://karpenter.sh/) 网站。

## 建议

最佳实践分为 Karpenter 本身、NodePools 和 pod 调度的部分。

## Karpenter 最佳实践

以下最佳实践涵盖了与 Karpenter 本身相关的主题。

### 将 Karpenter 用于具有不断变化容量需求的工作负载

Karpenter 将缩放管理带到了比 [Auto Scaling 组](https://aws.amazon.com/blogs/containers/amazon-eks-cluster-multi-zone-auto-scaling-groups/) (ASGs) 和 [托管节点组](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html) (MNGs) 更接近 Kubernetes 原生 API 的地方。ASG 和 MNG 是 AWS 原生抽象,其中缩放是基于 AWS 级别指标(如 EC2 CPU 负载)触发的。[集群自动缩放器](https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html#cluster-autoscaler)将 Kubernetes 抽象转换为 AWS 抽象,但由于此原因而失去了一些灵活性,例如针对特定可用区进行调度。

Karpenter 删除了一层 AWS 抽象,将一些灵活性直接引入 Kubernetes。Karpenter 最适合于工作负载遇到高峰值需求期或具有多样化计算需求的集群。MNG 和 ASG 适合于运行工作负载更静态和一致的集群。您可以根据需求使用动态和静态管理的节点。

### 考虑其他自动缩放项目,当...

您需要 Karpenter 仍在开发的功能。由于 Karpenter 是一个相对较新的项目,如果您需要的功能还不是 Karpenter 的一部分,请考虑其他自动缩放项目。

### 在 EKS Fargate 上或属于节点组的工作节点上运行 Karpenter 控制器

Karpenter 使用 [Helm 图表](https://karpenter.sh/docs/getting-started/)进行安装。Helm 图表安装 Karpenter 控制器和一个 webhook pod 作为 Deployment,需要在控制器可用于缩放集群之前运行。我们建议至少有一个小型节点组,至少有一个工作节点。或者,您可以通过为 `karpenter` 命名空间创建 Fargate 配置文件,在 EKS Fargate 上运行这些 pod。这样做将导致部署到此命名空间的所有 pod 在 EKS Fargate 上运行。不要在由 Karpenter 管理的节点上运行 Karpenter。

### 避免对 Karpenter 使用自定义启动模板

Karpenter 强烈建议不要使用自定义启动模板。使用自定义启动模板会阻碍多架构支持、自动升级节点的能力以及安全组发现。使用启动模板也可能会造成混淆,因为某些字段在 Karpenter 的 NodePools 中重复,而其他字段被 Karpenter 忽略,例如子网和实例类型。

您通常可以通过使用自定义用户数据和/或直接在 AWS 节点模板中指定自定义 AMI 来避免使用启动模板。有关如何执行此操作的更多信息,请参阅 [NodeClasses](https://karpenter.sh/docs/concepts/nodeclasses/)。

### 排除不适合您工作负载的实例类型

如果某些实例类型不是您集群中运行的工作负载所需要的,请考虑使用 [node.kubernetes.io/instance-type](http://node.kubernetes.io/instance-type) 键将其排除。

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

### 在使用 Spot 时启用中断处理

Karpenter 支持[原生中断处理](https://karpenter.sh/docs/concepts/disruption/#interruption),通过 `--interruption-queue-name` CLI 参数启用,该参数是 SQS 队列的名称。中断处理监视即将发生的非自愿中断事件,这些事件会中断您的工作负载,例如:

* Spot 中断警告
* 计划的变更运行状况事件(维护事件)
* 实例终止事件
* 实例停止事件

当 Karpenter 检测到这些事件将发生在您的节点上时,它会自动隔离、排空和提前终止节点,以便在中断事件发生之前为工作负载清理提供最大的时间。不建议将 AWS 节点终止处理程序与 Karpenter 一起使用,如[此处](https://karpenter.sh/docs/faq/#interruption-handling)所述。

需要检查点或其他形式的优雅排空的 pod,需要在关机前 2 分钟,应在其集群中启用 Karpenter 中断处理。

### **没有出站互联网访问的 Amazon EKS 私有集群**

在 VPC 中配置 EKS 集群时没有到互联网的路由,您必须确保您的环境已根据 EKS 文档中出现的私有集群[要求](https://docs.aws.amazon.com/eks/latest/userguide/private-clusters.html#private-cluster-requirements)进行了配置。此外,您需要确保在 VPC 中创建了 STS VPC 区域端点。否则,您将看到类似于下面出现的错误。

```console
{"level":"FATAL","time":"2024-02-29T14:28:34.392Z","logger":"controller","message":"Checking EC2 API connectivity, WebIdentityErr: failed to retrieve credentials\ncaused by: RequestError: send request failed\ncaused by: Post \"https://sts.<region>.amazonaws.com/\": dial tcp 54.239.32.126:443: i/o timeout","commit":"596ea97"}
```

这些更改在私有集群中是必要的,因为 Karpenter 控制器使用 IAM 角色服务帐户 (IRSA)。配置有 IRSA 的 pod 通过调用 AWS 安全令牌服务 (AWS STS) API 获取凭据。如果没有出站互联网访问,您必须创建并使用 ***VPC 中的 AWS STS 端点***。

私有集群还要求您创建 ***VPC 端点以访问 SSM***。当 Karpenter 尝试配置新节点时,它会查询启动模板配置和 SSM 参数。如果您的 VPC 中没有 SSM VPC 端点,它将导致以下错误:

```console
{"level":"ERROR","time":"2024-02-29T14:28:12.889Z","logger":"controller","message":"Unable to hydrate the AWS launch template cache, RequestCanceled: request context canceled\ncaused by: context canceled","commit":"596ea97","tag-key":"karpenter.k8s.aws/cluster","tag-value":"eks-workshop"}
...
{"level":"ERROR","time":"2024-02-29T15:08:58.869Z","logger":"controller.nodeclass","message":"discovering amis from ssm, getting ssm parameter \"/aws/service/eks/optimized-ami/1.27/amazon-linux-2/recommended/image_id\", RequestError: send request failed\ncaused by: Post \"https://ssm.<region>.amazonaws.com/\": dial tcp 67.220.228.252:443: i/o timeout","commit":"596ea97","ec2nodeclass":"default","query":"/aws/service/eks/optimized-ami/1.27/amazon-linux-2/recommended/image_id"}
```

没有 ***VPC 端点用于[价格列表查询 API](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/using-pelong.html)***。
因此,定价数据将随时间而过时。
Karpenter 通过在其二进制文件中包含按需定价数据来解决这个问题,但只在 Karpenter 升级时更新该数据。
定价数据请求失败将导致以下错误消息:

```console
{"level":"ERROR","time":"2024-02-29T15:08:58.522Z","logger":"controller.pricing","message":"retreiving on-demand pricing data, RequestError: send request failed\ncaused by: Post \"https://api.pricing.<region>.amazonaws.com/\": dial tcp 18.196.224.8:443: i/o timeout; RequestError: send request failed\ncaused by: Post \"https://api.pricing.<region>.amazonaws.com/\": dial tcp 18.185.143.117:443: i/o timeout","commit":"596ea97"}
```

总之,要在完全私有的 EKS 集群中使用 Karpenter,您需要创建以下 VPC 端点:

```console
com.amazonaws.<region>.ec2
com.amazonaws.<region>.ecr.api
com.amazonaws.<region>.ecr.dkr
com.amazonaws.<region>.s3 – 用于拉取容器镜像
com.amazonaws.<region>.sts – 用于 IAM 角色服务帐户
com.amazonaws.<region>.ssm - 用于解析默认 AMI
com.amazonaws.<region>.sqs - 如果使用中断处理,用于访问 SQS
```

!!! note
    Karpenter(控制器和 webhook 部署)容器镜像必须位于 Amazon ECR 私有或可从 VPC 内部访问的其他私有注册表中。这是因为 Karpenter 控制器和 webhook pod 当前使用公共 ECR 镜像。如果这些镜像在 VPC 内部或与 VPC 对等的网络中不可用,Kubernetes 在尝试从 ECR 公共拉取这些镜像时会出现 Image pull 错误。

有关更多信息,请参见[问题 988](https://github.com/aws/karpenter/issues/988)和[问题 1157](https://github.com/aws/karpenter/issues/1157)。

## 创建 NodePools

以下最佳实践涵盖了与创建 NodePools 相关的主题。

### 当...时创建多个 NodePools

当不同的团队共享一个集群并需要在不同的工作节点上运行其工作负载,或者有不同的操作系统或实例类型要求时,请创建多个 NodePools。例如,一个团队可能想使用 Bottlerocket,而另一个团队可能想使用 Amazon Linux。同样,一个团队可能有权访问其他团队不需要的昂贵的 GPU 硬件。使用多个 NodePools 可确保每个团队都可以访问最合适的资产。

### 创建互斥或加权的 NodePools

建议创建互斥或加权的 NodePools,以提供一致的调度行为。如果不是这样,并且匹配多个 NodePools,Karpenter 将随机选择使用哪一个,从而导致意外结果。创建多个 NodePools 的有用示例包括以下内容:

创建一个具有 GPU 的 NodePool,并仅允许特殊工作负载在这些(昂贵)节点上运行:

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

具有容忍度的部署:

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

对于另一个团队的一般部署,NodePool 规范可以包括 nodeAffinify。部署然后可以使用 nodeSelectorTerms 来匹配 `billing-team`。

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

使用 nodeAffinity 的部署:

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

### 使用定时器(TTL)自动从集群中删除节点

您可以在配置的节点上使用定时器来设置何时删除没有工作负载 pod 或已到期的节点。节点到期可用作升级的一种方式,使节点被退役并替换为更新的版本。有关使用 `spec.disruption.expireAfter` 配置节点到期的信息,请参见 Karpenter 文档中的[到期](https://karpenter.sh/docs/concepts/disruption/)。

### 避免过度限制 Karpenter 可以配置的实例类型,特别是在使用 Spot 时

使用 Spot 时,Karpenter 使用 [Price Capacity Optimized](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-fleet-allocation-strategy.html) 分配策略来配置 EC2 实例。此策略指示 EC2 从您正在启动的实例数量的最深池中配置实例,这些实例具有最低的中断风险。EC2 Fleet 然后从这些池中价格最低的实例请求 Spot 实例。您允许 Karpenter 使用的实例类型越多,EC2 就能越好地优化您的 Spot 实例运行时间。默认情况下,Karpenter 将使用 EC2 在集群部署所在区域和可用区中提供的所有实例类型。Karpenter 根据待处理的 pod 智能地从所有实例类型集合中进行选择,以确保您的 pod 被调度到适当大小和配备的实例上。例如,如果您的 pod 不需要 GPU,Karpenter 就不会将您的 pod 调度到支持 GPU 的 EC2 实例类型上。当您不确定使用哪些实例类型时,可以运行 Amazon [ec2-instance-selector](https://github.com/aws/amazon-ec2-instance-selector) 来生成与您的计算要求相匹配的实例类型列表。例如,CLI 接受内存 vCPU、架构和区域作为输入参数,并为您提供满足这些约束条件的 EC2 实例列表。

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

在使用 Spot 实例时,您不应该对 Karpenter 施加太多限制,因为这样做可能会影响您应用程序的可用性。例如,某个特定类型的所有实例都被回收,并且没有合适的替代品可用。您的 pod 将保持待处理状态,直到为配置的实例类型补充 Spot 容量。通过将实例分散在不同的可用区中,您可以降低容量不足错误的风险,因为 Spot 池在 AZ 之间是不同的。总的来说,在使用 Spot 时,一般最佳做法是让 Karpenter 使用多样化的实例类型。

## 调度 Pod

以下最佳实践与使用 Karpenter 进行节点配置时部署 pod 相关。

### 遵循 EKS 高可用性最佳实践

如果您需要运行高可用性应用程序,请遵循一般 EKS 最佳实践[建议](https://aws.github.io/aws-eks-best-practices/reliability/docs/application/#recommendations)。有关如何跨节点和区域分散 pod 的详细信息,请参见 Karpenter 文档中的[拓扑传播](https://karpenter.sh/docs/concepts/scheduling/#topology-spread)。使用[中断预算](https://karpenter.sh/docs/troubleshooting/#disruption-budgets)设置在尝试驱逐或删除 pod 时需要维护的最小可用 pod 数。

### 使用分层约束来限制云提供商提供的计算功能

Karpenter 的分层约束模型允许您创建一个复杂的 NodePool 和 pod 部署约束集,以获得最佳的 pod 调度匹配。 pod 规范可以请求的约束示例包括以下内容:

* 需要在只有特定应用程序可用的可用区域内运行。例如,您有一个 pod 必须与运行在特定可用区的 EC2 实例上的另一个应用程序进行通信。如果您的目标是减少 VPC 中的跨 AZ 流量,您可能希望将 pod 与 EC2 实例位于同一 AZ 中。这种定位通常通过节点选择器来完成。有关[节点选择器](https://karpenter.sh/docs/concepts/scheduling/#selecting-nodes)的更多信息,请参阅 Kubernetes 文档。
* 需要特定类型的处理器或其他硬件。有关 pod 规范示例,请参见 Karpenter 文档中的[加速器](https://karpenter.sh/docs/concepts/scheduling/#acceleratorsgpu-resources)部分,该示例要求 pod 在 GPU 上运行。

### 创建计费警报来监控您的计算支出

当您配置集群以自动扩展时,您应该创建计费警报来警告您支出是否超过了阈值,并在 Karpenter 配置中添加资源限制。使用 Karpenter 设置资源限制类似于在 AWS 自动缩放组中设置最大容量,它代表 Karpenter NodePool 可以实例化的最大计算资源量。

!!! note
    无法为整个集群设置全局限制。限制适用于特定的 NodePools。

下面的片段告诉 Karpenter 只配置最多 1000 个 CPU 内核和 1000Gi 内存。只有当达到或超过限制时,Karpenter 才会停止添加容量。当超过限制时,Karpenter 控制器将在控制器日志中写入 `memory resource usage of 1001 exceeds limit of 1000` 或类似的消息。如果您将容器日志路由到 CloudWatch 日志,您可以创建一个[指标过滤器](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/MonitoringLogData.html)来查找日志中的特定模式或术语,然后创建一个[CloudWatch 警报](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html)来在您配置的指标阈值被违反时提醒您。

有关将限制与 Karpenter 一起使用的更多信息,请参见 Karpenter 文档中的[设置资源限制](https://karpenter.sh/docs/concepts/nodepools/#speclimits)。

```yaml
spec:
  limits:
    cpu: 1000
    memory: 1000Gi
```

如果您不使用限制或限制 Karpenter 可以配置的实例类型,Karpenter 将继续根据需要向集群添加计算容量。虽然以这种方式配置 Karpenter 允许集群自由扩展,但也可能产生重大的成本影响。正因为如此,我们建议您配置计费警报。计费警报允许您在您的帐户中的计算估算费用超过定义的阈值时得到提醒和主动通知。有关更多信息,请参见[设置 Amazon CloudWatch 计费警报以主动监控估算费用](https://aws.amazon.com/blogs/mt/setting-up-an-amazon-cloudwatch-billing-alarm-to-proactively-monitor-estimated-charges/)。

您还可能想启用成本异常检测,这是 AWS 成本管理功能之一,它使用机器学习持续监控您的成本和使用情况,以检测异常支出。更多信息可以在 [AWS 成本异常检测入门](https://docs.aws.amazon.com/cost-management/latest/userguide/getting-started-ad.html)指南中找到。如果您已经在 AWS Budgets 中创建了预算,您也可以配置一个操作来通知您何时违反了特定的阈值。使用预算操作,您可以发送电子邮件、发布消息到 SNS 主题或发送消息到 Slack 等聊天机器人。有关更多信息,请参见[配置 AWS Budgets 操作](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-controls.html)。

### 使用 karpenter.sh/do-not-disrupt 注解来防止 Karpenter 取消配置节点

如果您在 Karpenter 配置的节点上运行关键应用程序,例如*长时间运行*的批处理作业或有状态应用程序,并且节点的 TTL 已到期,应用程序将在实例终止时中断。通过在 pod 上添加 `karpenter.sh/karpenter.sh/do-not-disrupt` 注解,您正在指示 Karpenter 保留节点,直到 Pod 被终止或删除 `karpenter.sh/do-not-disrupt` 注解。有关更多信息,请参见[中断](https://karpenter.sh/docs/concepts/disruption/#node-level-controls)文档。

如果节点上只剩下与作业相关的非 daemonset pod,只要作业状态为成功或失败,Karpenter 就能够定位和终止这些节点。

### 在使用合并时为所有非 CPU 资源配置 requests=limits

合并和调度通常通过比较 pod 资源请求与节点上可分配资源的数量来工作。资源限制不会被考虑。例如,具有大于内存请求的内存限制的 pod 可以突破请求。如果同一节点上的多个 pod 同时突破,这可能会导致由于内存不足 (OOM) 而终止某些 pod。合并可能会使这种情况更容易发生,因为它只考虑 pod 的请求来打包 pod。

### 使用 LimitRanges 为资源请求和限制配置默认值

由于 Kubernetes 没有设置默认请求或限制,容器对主机、CPU 和内存资源的消耗是无限制的。Kubernetes 调度程序查看 pod 的总请求(pod 容器的总请求或 pod Init 容器的总资源,以较高者为准)来确定将 pod 调度到哪个工作节点上。同样,Karpenter 考虑 pod 的请求来确定配置哪种类型的实例。您可以使用限制范围为命名空间应用合理的默认值,以防某些 pod 未指定资源请求。

请参见[为命名空间配置默认内存请求和限制](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/)

### 为所有工作负载应用准确的资源请求

当 Karpenter 对您的工作负载需求的信息准确时,它就能够启动最适合您工作负载的节点。这在使用 Karpenter 的合并功能时尤其重要。

请参见[为所有工作负载配置和调整资源请求/限制](https://aws.github.io/aws-eks-best-practices/reliability/docs/dataplane/#configure-and-size-resource-requestslimits-for-all-workloads)

## CoreDNS 建议

### 更新 CoreDNS 的配置以保持可靠性
在 Karpenter 管理的节点上部署 CoreDNS pod 时,考虑到 Karpenter 动态的性质会快速终止/创建新节点以匹配需求,建议遵循以下最佳实践:

[CoreDNS 延迟退出持续时间](https://aws.github.io/aws-eks-best-practices/scalability/docs/cluster-services/#coredns-lameduck-duration)

[CoreDNS 就绪探测](https://aws.github.io/aws-eks-best-practices/scalability/docs/cluster-services/#coredns-readiness-probe)

这将确保 DNS 查询不会定向到尚未就绪或已被终止的 CoreDNS Pod。

## Karpenter 蓝图
由于 Karpenter 采用以应用程序为先的方法来为 Kubernetes 数据平面配置计算容量,因此存在一些常见的工作负载场景,您可能想知道如何正确配置它们。[Karpenter 蓝图](https://github.com/aws-samples/karpenter-blueprints)是一个存储库,包含了遵循此处描述的最佳实践的常见工作负载场景列表。您将拥有您需要的所有资源,甚至可以创建一个配置有 Karpenter 的 EKS 集群,并测试存储库中包含的每个蓝图。您可以组合不同的蓝图来最终创建您的工作负载所需的蓝图。

## 其他资源
* [Karpenter/Spot 研讨会](https://ec2spotworkshops.com/karpenter.html)
* [Karpenter 节点配置器](https://youtu.be/_FXRIKWJWUk)
* [TGIK Karpenter](https://youtu.be/zXqrNJaTCrU)
* [Karpenter vs. 集群自动缩放器](https://youtu.be/3QsVRHVdOnM)
* [无组 Autoscaling with Karpenter](https://www.youtube.com/watch?v=43g8uPohTgc)
* [教程:使用 Amazon EC2 Spot 和 Karpenter 以较低的成本运行 Kubernetes 集群](https://community.aws/tutorials/run-kubernetes-clusters-for-less-with-amazon-ec2-spot-and-karpenter#step-6-optional-simulate-spot-interruption)