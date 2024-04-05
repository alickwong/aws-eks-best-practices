
# Karpenter 最佳实践

## Karpenter

Karpenter 是一个开源的集群自动扩缩器,可根据无法调度的 pod 自动配置新节点。Karpenter 评估待调度 pod 的总体资源需求,并选择最合适的实例类型来运行它们。它将自动缩小或终止没有任何非 daemonset pod 的实例,以减少浪费。它还支持一个合并功能,可以主动移动 pod 并删除或替换节点以降低集群成本。

**使用 Karpenter 的原因**

在 Karpenter 推出之前,Kubernetes 用户主要依赖于 [Amazon EC2 Auto Scaling 组](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html)和 [Kubernetes 集群自动缩放器](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler) (CAS) 来动态调整集群的计算能力。使用 Karpenter,您无需创建数十个节点组即可实现 Karpenter 提供的灵活性和多样性。此外,Karpenter 与 Kubernetes 版本的耦合性不如 CAS 那么紧密,您也无需在 AWS 和 Kubernetes API 之间来回切换。

Karpenter 将实例编排职责集中在一个系统中,这更简单、更稳定且更了解集群。Karpenter 旨在克服集群自动缩放器带来的一些挑战,提供以下简化方式:

* 根据工作负载需求配置节点。
* 通过实例类型创建多样化的节点配置,使用灵活的 NodePool 选项。与管理许多特定的自定义节点组不同,Karpenter 可以让您使用单个灵活的 NodePool 管理多样化的工作负载容量。
* 通过快速启动节点和调度 pod 来实现更好的 pod 调度。

有关使用 Karpenter 的信息和文档,请访问 [karpenter.sh](https://karpenter.sh/) 网站。

## 建议

最佳实践分为 Karpenter 本身、NodePools 和 pod 调度三个部分。

## Karpenter 最佳实践

以下最佳实践涵盖了与 Karpenter 本身相关的主题。

### 将 Karpenter 用于需求变化的工作负载

与 [Auto Scaling 组](https://aws.amazon.com/blogs/containers/amazon-eks-cluster-multi-zone-auto-scaling-groups/) (ASGs) 和 [托管节点组](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html) (MNGs) 相比,Karpenter 将扩缩管理更接近于 Kubernetes 原生 API。ASGs 和 MNGs 是 AWS 原生抽象,其扩缩是基于 AWS 级别的指标,如 EC2 CPU 负载。[集群自动缩放器](https://docs.aws.amazon.com/eks/latest/userguide/autoscaling.html#cluster-autoscaler)将 Kubernetes 抽象转换为 AWS 抽象,但由于此原因而失去了一些灵活性,例如针对特定可用区进行调度。

卡彭特移除了 AWS 抽象层的一层,直接将一些灵活性带入 Kubernetes。卡彭特最适用于工作负载在高峰需求期和计算需求多样化的集群。MNG 和 ASG 更适合于运行负载较为静态和一致的集群。您可以根据需求使用动态管理和静态管理的节点混合。

### 在以下情况下考虑其他自动扩展项目...

您需要的功能仍在卡彭特开发中。由于卡彭特是一个相对较新的项目,如果您需要的功能还没有包含在卡彭特中,请暂时考虑其他自动扩展项目。

### 在 EKS Fargate 或属于节点组的工作节点上运行卡彭特控制器

卡彭特使用 [Helm 图表](https://karpenter.sh/docs/getting-started/)进行安装。Helm 图表会安装卡彭特控制器和一个 Webhook pod,作为一个 Deployment,需要在控制器可用于扩展集群之前运行。我们建议至少有一个小型节点组,至少有一个工作节点。或者,您可以通过为 `karpenter` 命名空间创建一个 Fargate 配置文件,在 EKS Fargate 上运行这些 pod。这样做会导致部署到此命名空间的所有 pod 在 EKS Fargate 上运行。请不要在由卡彭特管理的节点上运行卡彭特。

### 避免在卡彭特中使用自定义启动模板

卡彭特强烈建议不要使用自定义启动模板。使用自定义启动模板会阻碍多架构支持、自动升级节点的能力以及安全组发现。使用启动模板也可能会造成混淆,因为某些字段在卡彭特的 NodePools 中被复制,而其他字段被卡彭特忽略,例如子网和实例类型。

您通常可以通过使用自定义用户数据和/或直接在 AWS 节点模板中指定自定义 AMI 来避免使用启动模板。有关如何执行此操作的更多信息,请参阅 [NodeClasses](https://karpenter.sh/docs/concepts/nodeclasses/)。

### 排除不适合您工作负载的实例类型

如果某些实例类型不被您集群中运行的工作负载所需要,请考虑使用 [node.kubernetes.io/instance-type](http://node.kubernetes.io/instance-type) 键排除它们。

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

使用 Spot 时启用中断处理

Karpenter 支持[原生中断处理](https://karpenter.sh/docs/concepts/disruption/#interruption)，通过 `--interruption-queue-name` CLI 参数启用,并指定 SQS 队列的名称。中断处理会监视即将发生的非自愿中断事件,这些事件会对您的工作负载造成中断,例如:

* Spot 中断警告
* 计划的健康变更事件(维护事件)
* 实例终止事件
* 实例停止事件

当 Karpenter 检测到这些事件将发生在您的节点上时,它会自动隔离、排空并在中断事件发生前终止节点,以便在中断前为工作负载清理提供最大的时间。不建议将 AWS Node Termination Handler 与 Karpenter 一起使用,如[此处](https://karpenter.sh/docs/faq/#interruption-handling)所述。

需要检查点或其他形式的优雅排空的 Pod,在关机前需要 2 分钟的时间,应在其集群中启用 Karpenter 中断处理。

### **没有出站互联网访问的 Amazon EKS 私有集群**

在将 EKS 集群配置到没有到互联网路由的 VPC 中时,您必须确保已根据 EKS 文档中的[私有集群要求](https://docs.aws.amazon.com/eks/latest/userguide/private-clusters.html#private-cluster-requirements)配置好您的环境。此外,您还需要确保在 VPC 中创建了 STS VPC 区域端点。否则,您将看到类似于下面的错误。
```console
{"level":"FATAL","time":"2024-02-29T14:28:34.392Z","logger":"controller","message":"Checking EC2 API connectivity, WebIdentityErr: failed to retrieve credentials\ncaused by: RequestError: send request failed\ncaused by: Post \"https://sts.<region>.amazonaws.com/\": dial tcp 54.239.32.126:443: i/o timeout","commit":"596ea97"}
```

这些更改在私有集群中是必要的,因为 Karpenter 控制器使用 IAM 角色服务帐户 (IRSA)。使用 IRSA 配置的 Pod 通过调用 AWS 安全令牌服务 (AWS STS) API 获取凭据。如果没有出站互联网访问,您必须在 VPC 中创建并使用 ***AWS STS VPC 端点***。

私有集群还要求您创建 ***SSM VPC 端点***。当 Karpenter 尝试配置新节点时,它会查询启动模板配置和 SSM 参数。如果您的 VPC 中没有 SSM VPC 端点,将会出现以下错误:
```console
{"level":"ERROR","time":"2024-02-29T14:28:12.889Z","logger":"controller","message":"Unable to hydrate the AWS launch template cache, RequestCanceled: request context canceled\ncaused by: context canceled","commit":"596ea97","tag-key":"karpenter.k8s.aws/cluster","tag-value":"eks-workshop"}
...
{"level":"ERROR","time":"2024-02-29T15:08:58.869Z","logger":"controller.nodeclass","message":"discovering amis from ssm, getting ssm parameter \"/aws/service/eks/optimized-ami/1.27/amazon-linux-2/recommended/image_id\", RequestError: send request failed\ncaused by: Post \"https://ssm.<region>.amazonaws.com/\": dial tcp 67.220.228.252:443: i/o timeout","commit":"596ea97","ec2nodeclass":"default","query":"/aws/service/eks/optimized-ami/1.27/amazon-linux-2/recommended/image_id"}
```

没有 ***[价格列表查询 API](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/using-pelong.html)*** 的 VPC 端点。
因此,定价数据将随时间而过时。
Karpenter 通过在其二进制文件中包含按需定价数据来解决这个问题,但只在升级 Karpenter 时更新该数据。
获取定价数据失败将导致以下错误消息:
```console
{"level":"ERROR","time":"2024-02-29T15:08:58.522Z","logger":"controller.pricing","message":"retreiving on-demand pricing data, RequestError: send request failed\ncaused by: Post \"https://api.pricing.<region>.amazonaws.com/\": dial tcp 18.196.224.8:443: i/o timeout; RequestError: send request failed\ncaused by: Post \"https://api.pricing.<region>.amazonaws.com/\": dial tcp 18.185.143.117:443: i/o timeout","commit":"596ea97"}
```

在总结中,要在完全私有的 EKS 集群中使用 Karpenter,您需要创建以下 VPC 端点:
```console
com.amazonaws.<region>.ec2
com.amazonaws.<region>.ecr.api
com.amazonaws.<region>.ecr.dkr
com.amazonaws.<region>.s3 – For pulling container images
com.amazonaws.<region>.sts – For IAM roles for service accounts
com.amazonaws.<region>.ssm - For resolving default AMIs
com.amazonaws.<region>.sqs - For accessing SQS if using interruption handling
```


创建 Karpenter（控制器和 webhook 部署）容器映像时，必须将其放在 Amazon ECR 私有存储库或 VPC 内部可访问的其他私有注册表中。这是因为 Karpenter 控制器和 webhook pod 当前使用公共 ECR 映像。如果这些映像在 VPC 内部或与 VPC 对等的网络中不可用，当 Kubernetes 尝试从 ECR 公共存储库拉取这些映像时，您将遇到映像拉取错误。

有关更多信息，请参见 [Issue 988](https://github.com/aws/karpenter/issues/988) 和 [Issue 1157](https://github.com/aws/karpenter/issues/1157)。

## 创建节点池

以下最佳实践涵盖了与创建节点池相关的主题。

### 在以下情况下创建多个节点池

当不同的团队共享一个集群并需要在不同的工作节点上运行其工作负载，或者有不同的操作系统或实例类型要求时，请创建多个节点池。例如，一个团队可能想使用 Bottlerocket，而另一个团队可能想使用 Amazon Linux。同样，一个团队可能有权访问另一个团队不需要的昂贵的 GPU 硬件。使用多个节点池可确保为每个团队提供最合适的资产。

### 创建互斥或加权的节点池

建议创建互斥或加权的节点池以提供一致的调度行为。如果不是这样，并且匹配到多个节点池，Karpenter 将随机选择使用哪个节点池，从而导致意外结果。创建多个节点池的有用示例包括以下内容:

创建一个仅允许特殊工作负载在这些（昂贵的）节点上运行的 GPU 节点池:
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

部署具有容忍污点的功能:
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

对于另一个团队的一般部署,NodePool 规范可以包括 nodeAffinify。然后,部署可以使用 nodeSelectorTerms 来匹配 `billing-team`。
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

使用 nodeAffinity 进行部署:
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

您可以在预配置的节点上使用定时器来设置何时删除没有工作负载 pod 或已到期的节点。节点到期可用作升级的一种方式,使节点被退役并替换为更新的版本。有关使用 `spec.disruption.expireAfter` 配置节点到期的信息,请参见 Karpenter 文档中的[到期](https://karpenter.sh/docs/concepts/disruption/)。

### 避免过度限制 Karpenter 可以预配的实例类型,特别是在使用 Spot 时

在使用 Spot 时,Karpenter 使用[价格容量优化](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-fleet-allocation-strategy.html)分配策略来预配 EC2 实例。此策略指示 EC2 从您正在启动的实例数量的最深池中预配实例,这些实例具有最低的中断风险。EC2 Fleet 然后从这些池中价格最低的池请求 Spot 实例。您允许 Karpenter 使用的实例类型越多,EC2 就能越好地优化您的 Spot 实例的运行时间。默认情况下,Karpenter 将使用您的集群部署所在区域和可用区中 EC2 提供的所有实例类型。Karpenter 会根据待处理的 pod 智能地从所有实例类型集合中进行选择,以确保您的 pod 被调度到适当大小和配置的实例上。例如,如果您的 pod 不需要 GPU,Karpenter 将不会将您的 pod 调度到支持 GPU 的 EC2 实例类型上。当您不确定使用哪些实例类型时,可以运行 Amazon [ec2-instance-selector](https://github.com/aws/amazon-ec2-instance-selector) 来生成一个满足您计算需求的实例类型列表。例如,该 CLI 接受内存 vCPU、架构和区域作为输入参数,并为您提供满足这些约束条件的 EC2 实例列表。
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


当使用Spot实例时，您不应该对Karpenter施加太多限制,因为这样做可能会影响您应用程序的可用性。例如,某种类型的所有实例都被回收,并且没有合适的替代品可用。您的pod将保持待处理状态,直到配置的实例类型的Spot容量得到补充。您可以通过在不同的可用区域分散您的实例来降低容量不足错误的风险,因为Spot池在各个可用区域是不同的。也就是说,一般的最佳做法是允许Karpenter在使用Spot时使用多种实例类型。

## 调度Pod

以下最佳实践与使用Karpenter进行节点配置时部署pod相关。

### 遵循EKS高可用性最佳实践

如果您需要运行高可用性应用程序,请遵循一般的EKS最佳实践[建议](https://aws.github.io/aws-eks-best-practices/reliability/docs/application/#recommendations)。有关如何在节点和区域之间分散pod的详细信息,请参见Karpenter文档中的[拓扑传播](https://karpenter.sh/docs/concepts/scheduling/#topology-spread)。使用[中断预算](https://karpenter.sh/docs/troubleshooting/#disruption-budgets)来设置需要维护的最小可用pod数量,以防止pod被驱逐或删除。

### 使用分层约束来限制云提供商提供的计算功能

Karpenter的分层约束模型允许您创建一组复杂的NodePool和pod部署约束,以获得最佳的pod调度匹配。pod规范可以请求的约束示例包括以下内容:

* 需要在仅特定应用程序可用的可用区域运行。例如,您有一个pod必须与运行在特定可用区域的EC2实例上的另一个应用程序进行通信。如果您的目标是减少VPC中的跨AZ流量,您可能希望将pod与EC2实例位于同一AZ中。这种定位通常通过节点选择器来完成。有关[节点选择器](https://karpenter.sh/docs/concepts/scheduling/#selecting-nodes)的更多信息,请参阅Kubernetes文档。
* 需要特定类型的处理器或其他硬件。有关pod规范示例,请参见Karpenter文档中的[加速器](https://karpenter.sh/docs/concepts/scheduling/#acceleratorsgpu-resources)部分,该示例要求pod在GPU上运行。

### 创建计费警报以监控您的计算支出
当您将集群配置为自动扩展时，您应该创建计费警报来警告您当您的支出超过阈值时，并将资源限制添加到您的 Karpenter 配置中。使用 Karpenter 设置资源限制类似于设置 AWS 自动缩放组的最大容量，因为它代表了 Karpenter NodePool 可以实例化的最大计算资源量。

!!! 注意
    无法为整个集群设置全局限制。限制适用于特定的 NodePools。

下面的代码段告诉 Karpenter 只提供最多 1000 个 CPU 内核和 1000Gi 的内存。只有当达到或超过限制时，Karpenter 才会停止添加容量。当超过限制时，Karpenter 控制器将在控制器日志中写入"内存资源使用量 1001 超过 1000 的限制"或类似的消息。如果您将容器日志路由到 CloudWatch 日志，您可以创建一个[指标过滤器](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/MonitoringLogData.html)来查找日志中的特定模式或术语，然后创建一个[CloudWatch 警报](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html)来在达到您配置的指标阈值时发出警报。

有关在 Karpenter 中使用限制的更多信息，请参见 Karpenter 文档中的[设置资源限制](https://karpenter.sh/docs/concepts/nodepools/#speclimits)。
```yaml
spec:
  limits:
    cpu: 1000
    memory: 1000Gi
```


如果您不使用限制或约束 Karpenter 可以配置的实例类型，Karpenter 将继续根据需要为您的集群添加计算能力。虽然以这种方式配置 Karpenter 允许您的集群自由扩展，但也可能会产生重大的成本影响。正因如此，我们建议您配置账单警报。账单警报可以在您的账户中的计算估算费用超过定义的阈值时提醒您并主动通知您。有关更多信息,请参见[设置 Amazon CloudWatch 账单警报以主动监控估算费用](https://aws.amazon.com/blogs/mt/setting-up-an-amazon-cloudwatch-billing-alarm-to-proactively-monitor-estimated-charges/)。

您还可以启用成本异常检测,这是 AWS 成本管理功能,它使用机器学习持续监控您的成本和使用情况,以检测异常支出。有关更多信息,请参见[AWS 成本异常检测入门](https://docs.aws.amazon.com/cost-management/latest/userguide/getting-started-ad.html)指南。如果您已经在 AWS Budgets 中创建了预算,您也可以配置一个操作来通知您何时违反了特定的阈值。使用预算操作,您可以发送电子邮件、发布消息到 SNS 主题或发送消息到 Slack 等聊天机器人。有关更多信息,请参见[配置 AWS Budgets 操作](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-controls.html)。

### 使用 karpenter.sh/do-not-disrupt 注解来防止 Karpenter 取消配置节点

如果您在 Karpenter 配置的节点上运行关键应用程序,例如*长期运行*的批处理作业或有状态应用程序,并且节点的 TTL 已过期,则应用程序将在实例终止时中断。通过向 pod 添加 `karpenter.sh/karpenter.sh/do-not-disrupt` 注解,您正在指示 Karpenter 保留节点,直到 Pod 终止或删除 `karpenter.sh/do-not-disrupt` 注解。有关更多信息,请参见[中断](https://karpenter.sh/docs/concepts/disruption/#node-level-controls)文档。

如果节点上只剩下与作业相关的非 daemonset pod,只要作业状态为成功或失败,Karpenter 就能够定位和终止这些节点。

### 在使用合并时为所有非 CPU 资源配置 requests=limits

整合和调度通常通过比较 pod 的资源请求与节点上可分配资源的数量来实现。不考虑资源限制。例如，内存限制大于内存请求的 pod 可以突破请求。如果同一节点上的多个 pod 同时突破,这可能导致某些 pod 由于内存不足 (OOM) 而被终止。整合可能会增加这种情况发生的可能性,因为它只考虑 pod 的请求来将其打包到节点上。

### 使用 LimitRanges 配置资源请求和限制的默认值

由于 Kubernetes 没有设置默认请求或限制,容器对底层主机、CPU 和内存的消耗是无限的。Kubernetes 调度程序查看 pod 的总请求(pod 容器的总请求或 pod Init 容器的总资源中较高者)来确定将 pod 调度到哪个工作节点。类似地,Karpenter 考虑 pod 的请求来确定配置哪种类型的实例。您可以使用限制范围为命名空间应用合理的默认值,以防某些 pod 未指定资源请求。

请参见[为命名空间配置默认内存请求和限制](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/)

### 为所有工作负载应用准确的资源请求

当 Karpenter 对工作负载需求的信息准确时,它能够启动最适合您工作负载的节点。如果使用 Karpenter 的整合功能,这一点尤为重要。

请参见[为所有工作负载配置和调整资源请求/限制](https://aws.github.io/aws-eks-best-practices/reliability/docs/dataplane/#configure-and-size-resource-requestslimits-for-all-workloads)

## CoreDNS 建议

### 更新 CoreDNS 配置以保持可靠性
在由 Karpenter 管理的节点上部署 CoreDNS pod 时,鉴于 Karpenter 动态的性质,它会快速终止/创建新节点以与需求保持一致,因此建议遵循以下最佳实践:

[CoreDNS 延迟退出持续时间](https://aws.github.io/aws-eks-best-practices/scalability/docs/cluster-services/#coredns-lameduck-duration)

[CoreDNS 就绪探测](https://aws.github.io/aws-eks-best-practices/scalability/docs/cluster-services/#coredns-readiness-probe)

这将确保 DNS 查询不会定向到尚未就绪或已被终止的 CoreDNS Pod。

## Karpenter 蓝图

作为 Karpenter 采取应用程序优先的方法来为 Kubernetes 数据平面配置计算能力,您可能会想知道如何正确配置常见的工作负载场景。[Karpenter Blueprints](https://github.com/aws-samples/karpenter-blueprints) 是一个存储库,其中包含了一系列遵循此处描述的最佳实践的常见工作负载场景。您将拥有创建配置有 Karpenter 的 EKS 集群并测试存储库中包含的每个蓝图所需的所有资源。您可以组合不同的蓝图来最终创建适合您工作负载的蓝图。

## 其他资源
* [Karpenter/Spot Workshop](https://ec2spotworkshops.com/karpenter.html)
* [Karpenter Node Provisioner](https://youtu.be/_FXRIKWJWUk)
* [TGIK Karpenter](https://youtu.be/zXqrNJaTCrU)
* [Karpenter vs. Cluster Autoscaler](https://youtu.be/3QsVRHVdOnM)
* [Groupless Autoscaling with Karpenter](https://www.youtube.com/watch?v=43g8uPohTgc)
* [教程: 使用 Amazon EC2 Spot 和 Karpenter 以更低的成本运行 Kubernetes 集群](https://community.aws/tutorials/run-kubernetes-clusters-for-less-with-amazon-ec2-spot-and-karpenter#step-6-optional-simulate-spot-interruption)
