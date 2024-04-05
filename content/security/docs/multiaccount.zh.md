!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

# 多账户策略

AWS 建议使用多账户策略和 AWS 组织来帮助隔离和管理您的业务应用程序和数据。使用多账户策略有[许多好处](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/benefits-of-using-multiple-aws-accounts.html):

- 增加 AWS API 服务配额。配额适用于 AWS 账户,使用多个账户进行工作负载可增加可用于工作负载的总体配额。
- 更简单的身份和访问管理 (IAM) 策略。仅向工作负载及其支持人员授予对其自身 AWS 账户的访问权限,意味着花费的时间来制定细粒度的 IAM 策略以实现最小权限原则会更少。
- 改善 AWS 资源的隔离。从设计上来说,在一个账户中配置的所有资源在逻辑上都与其他账户中配置的资源隔离。这种隔离边界为您提供了一种方式来限制应用程序相关问题、配置错误或恶意行为的风险。如果在一个账户中发生问题,对其他账户中包含的工作负载的影响可以减少或消除。
- 更多好处,如 [AWS 多账户策略白皮书](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/benefits-of-using-multiple-aws-accounts.html#group-workloads-based-on-business-purpose-and-ownership)中所述

以下部分将解释如何使用集中式或去中心化 EKS 集群方法为您的 EKS 工作负载实施多账户策略。

## 为多租户集群规划多工作负载账户策略

在多账户 AWS 策略中,属于给定工作负载的资源,如 S3 存储桶、ElastiCache 集群和 DynamoDB 表,都是在包含该工作负载所有资源的 AWS 账户中创建的。这些被称为工作负载账户,而 EKS 集群部署在被称为集群账户的账户中。集群账户将在下一节中探讨。将资源部署到专用的工作负载账户类似于将 Kubernetes 资源部署到专用的命名空间。

工作负载账户然后可以根据软件开发生命周期或其他要求进一步细分。例如,给定的工作负载可以有一个生产账户、一个开发账户,或者在特定区域托管该工作负载实例的账户。[此 AWS 白皮书](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/organizing-workload-oriented-ous.html)中提供了更多信息。

在实施 EKS 多账户策略时,您可以采用以下方法:

## 集中式 EKS 集群

在这种方法中，您的 EKS 集群将部署在一个名为"集群帐户"的单个 AWS 帐户中。使用 [IAM 角色服务帐户 (IRSA)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) 或 [EKS Pod 身份](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html) 提供临时 AWS 凭证，以及 [AWS 资源访问管理器 (RAM)](https://aws.amazon.com/ram/) 简化网络访问，您可以为多租户 EKS 集群采用多帐户策略。集群帐户将包含 VPC、子网、EKS 集群、EC2/Fargate 计算资源(工作节点)以及运行 EKS 集群所需的任何其他网络配置。

在多租户集群的多工作负载帐户策略中，AWS 帐户通常与 [Kubernetes 命名空间](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)保持一致,作为隔离资源组的机制。在实施多帐户策略以支持多租户 EKS 集群时,仍应遵循 [EKS 集群内多租户隔离的最佳实践](/security/docs/multitenancy/)。

您的 AWS 组织中可以拥有多个"集群帐户",并且拥有多个与您的软件开发生命周期需求相匹配的"集群帐户"是一种最佳实践。对于规模非常大的工作负载,您可能需要多个"集群帐户",以确保为所有工作负载提供足够的 Kubernetes 和 AWS 服务配额。

| ![multi-account-eks](./images/multi-account-eks.jpg) |
|:--:|
| 在上图中,AWS RAM 用于将子网从集群帐户共享到工作负载帐户。然后,在 EKS Pod 中运行的工作负载使用 IRSA 或 EKS Pod 身份和角色链接来假定其工作负载帐户中的角色,并访问其 AWS 资源。|

### 实施多工作负载帐户策略以支持多租户集群

#### 使用 AWS 资源访问管理器共享子网

[AWS 资源访问管理器](https://aws.amazon.com/ram/)(RAM)允许您跨 AWS 帐户共享资源。

如果[为您的 AWS 组织启用了 RAM](https://docs.aws.amazon.com/ram/latest/userguide/getting-started-sharing.html#getting-started-sharing-orgs),您可以将集群帐户中的 VPC 子网共享到您的工作负载帐户。这将允许您的工作负载帐户拥有的 AWS 资源,如 [Amazon ElastiCache](https://aws.amazon.com/elasticache/) 集群或 [Amazon Relational Database Service (RDS)](https://aws.amazon.com/rds/) 数据库,部署到与您的 EKS 集群相同的 VPC 中,并由在您的 EKS 集群上运行的工作负载使用。

要通过 RAM 共享资源,请在集群帐户的 AWS 控制台中打开 RAM,选择"资源共享"和"创建资源共享"。为您的资源共享命名,并选择要共享的子网。再次选择下一步,输入您希望与之共享子网的工作负载帐户的 12 位数帐户 ID,再次选择下一步,然后单击创建资源共享以完成。完成此步骤后,工作负载帐户可以在这些子网中部署资源。

RAM 共享也可以以编程方式创建,或使用基础设施即代码。

#### 在 EKS Pod 身份和 IRSA 之间进行选择

在 re:Invent 2023 上,AWS 推出了 EKS Pod 身份,这是一种更简单的为您在 EKS 上的 pod 提供临时 AWS 凭证的方式。IRSA 和 EKS Pod 身份都是为 EKS pod 提供临时 AWS 凭证的有效方法,并将继续得到支持。您应该考虑哪种交付方式最能满足您的需求。

在使用 EKS 集群和多个 AWS 帐户时,IRSA 可以直接在 EKS 集群所在帐户以外的 AWS 帐户中承担角色,而 EKS Pod 身份要求您配置角色链接。请参考 [EKS 文档](https://docs.aws.amazon.com/eks/latest/userguide/service-accounts.html#service-accounts-iam)进行深入比较。

##### 使用 IAM 角色访问 AWS API 资源

[IAM 角色服务帐户 (IRSA)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) 允许您为在 EKS 上运行的工作负载提供临时 AWS 凭证。IRSA 可用于从集群帐户获取工作负载帐户中 IAM 角色的临时凭证。这允许您在集群帐户中的 EKS 集群上运行的工作负载无缝地使用 IAM 身份验证来使用托管在工作负载帐户中的 AWS API 资源,如 S3 存储桶或 Amazon RDS 数据库。

只有在跨帐户访问可用且已明确启用的情况下,工作负载帐户中的 AWS API 资源和其他使用 IAM 身份验证的资源才能被工作负载帐户中的 IAM 角色凭证访问。

###### 为跨帐户访问启用 IRSA

要为集群帐户中的工作负载启用对工作负载帐户中资源的 IRSA 访问,您首先必须在工作负载帐户中创建一个 IAM OIDC 身份提供商。这可以使用与设置 [IRSA](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html) 相同的过程完成,只是身份提供商将在工作负载帐户中创建。
[在为您在 EKS 上的工作负载配置 IRSA 时，您可以遵循文档中的相同步骤](https://docs.aws.amazon.com/eks/latest/userguide/associate-service-account-role.html)，但使用[工作负载帐户的 12 位数帐户 ID](https://docs.aws.amazon.com/eks/latest/userguide/cross-account-access.html)，如"从另一个帐户的集群创建身份提供程序"部分所述。

配置完成后，在 EKS 中运行的应用程序将能够直接使用其服务帐户来承担工作负载帐户中的角色，并使用其中的资源。

##### 使用 EKS Pod 身份访问 AWS API 资源

[EKS Pod 身份](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)是一种向在 EKS 上运行的工作负载提供 AWS 凭证的新方式。EKS Pod 身份简化了 AWS 资源的配置，因为您不再需要管理 OIDC 配置来向 EKS 上的 pod 提供 AWS 凭证。

###### 为跨账户访问启用 EKS Pod 身份

与 IRSA 不同，EKS Pod 身份只能用于直接授予对与 EKS 集群相同帐户中角色的访问权限。要访问另一个 AWS 帐户中的角色，使用 EKS Pod 身份的 pod 必须执行[角色链接](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_terms-and-concepts.html#iam-term-role-chaining)。

角色链接可以在应用程序配置文件的 aws 配置文件中使用[进程凭证提供程序](https://docs.aws.amazon.com/sdkref/latest/guide/feature-process-credentials.html)进行配置，该提供程序可在各种 AWS SDK 中使用。`credential_process` 可用作配置配置文件时的凭证源，例如:
```bash
# Content of the AWS Config file
[profile account_b_role] 
source_profile = account_a_role 
role_arn = arn:aws:iam::444455556666:role/account-b-role

[profile account_a_role] 
credential_process = /eks-credential-processrole.sh
```

凭证过程调用的脚本源:

![any text]
```bash
#!/bin/bash
# Content of the eks-credential-processrole.sh
# This will retreive the credential from the pod identities agent,
# and return it to the AWS SDK when referenced in a profile
curl -H "Authorization: $(cat $AWS_CONTAINER_AUTHORIZATION_TOKEN_FILE)" $AWS_CONTAINER_CREDENTIALS_FULL_URI | jq -c '{AccessKeyId: .AccessKeyId, SecretAccessKey: .SecretAccessKey, SessionToken: .Token, Expiration: .Expiration, Version: 1}' 
```

您可以像上面所示创建一个 AWS 配置文件,其中包含 A 和 B 两个账户的角色,并在 pod 规格中指定 AWS_CONFIG_FILE 和 AWS_PROFILE 环境变量。如果 pod 规格中已经存在这些环境变量,EKS Pod 身份 webhook 不会覆盖它们。
```yaml
# Snippet of the PodSpec
containers: 
  - name: container-name
    image: container-image:version
    env:
    - name: AWS_CONFIG_FILE
      value: path-to-customer-provided-aws-config-file
    - name: AWS_PROFILE
      value: account_b_role
```

在为 EKS pod 身份配置角色信任策略进行角色链接时,您可以将 [EKS 特定属性](https://docs.aws.amazon.com/eks/latest/userguide/pod-id-abac.html)作为会话标签进行引用,并使用基于属性的访问控制 (ABAC) 来限制对您的 IAM 角色的访问,仅限于特定的 EKS Pod 身份会话,例如 pod 所属的 Kubernetes 服务帐户。

请注意,这些属性中的一些可能并不是完全唯一的,例如两个 EKS 集群可能具有相同的命名空间,一个集群可能在不同命名空间中具有相同名称的服务帐户。因此,在通过 EKS Pod 身份和 ABAC 授予访问权限时,始终考虑集群 arn 和命名空间是最佳实践。

###### ABAC 和 EKS Pod 身份用于跨账户访问

在使用 EKS Pod 身份来承担其他账户中的角色(角色链接)作为多账户策略的一部分时,您可以为需要访问另一个账户的每个服务帐户分配一个唯一的 IAM 角色,或使用一个通用的 IAM 角色跨多个服务帐户,并使用 ABAC 来控制它可以访问的账户。

要使用 ABAC 来控制哪些服务帐户可以通过角色链接承担另一个账户中的角色,您需要创建一个角色信任策略语句,该语句仅在存在预期值时允许角色被承担。以下角色信任策略将仅允许 EKS 集群账户(账户 ID 111122223333)中的角色在 `kubernetes-service-account`、`eks-cluster-arn` 和 `kubernetes-namespace` 标签都具有预期值的情况下被承担。
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::111122223333:root"
            },
            "Action": "sts:AssumeRole",
            "Condition": {
                "StringEquals": {
                    "aws:PrincipalTag/kubernetes-service-account": "PayrollApplication",
                    "aws:PrincipalTag/eks-cluster-arn": "arn:aws:eks:us-east-1:111122223333:cluster/ProductionCluster",
                    "aws:PrincipalTag/kubernetes-namespace": "PayrollNamespace"
                }
            }
        }
    ]
}
```


当使用此策略时,最佳做法是确保常见的 IAM 角色只有 `sts:AssumeRole` 权限,而没有其他 AWS 访问权限。

在使用 ABAC 时,重要的是您要控制谁有权标记 IAM 角色和用户,只有那些确实需要这样做的人才能这样做。有能力标记 IAM 角色或用户的人可以在角色/用户上设置与 EKS Pod 身份设置的标签相同的标签,并可能升级他们的权限。您可以使用 IAM 策略或服务控制策略 (SCP) 限制谁有权设置 `kubernetes-` 和 `eks-` 标签。

## 去中心化 EKS 集群

在这种方法中,EKS 集群部署到各自的工作负载 AWS 帐户,并与其他 AWS 资源(如 Amazon S3 存储桶、VPC、Amazon DynamoDB 表等)并存。每个工作负载帐户都是独立的、自给自足的,由各自的业务部门/应用程序团队运营。这种模型允许为各种集群功能(AI/ML 集群、批处理、通用等)创建可重复使用的蓝图,并根据应用程序团队的需求提供集群。应用程序和平台团队都在各自的 [GitOps](https://www.weave.works/technologies/gitops/) 存储库中运作,以管理对工作负载集群的部署。

![去中心化 EKS 集群架构](./images/multi-account-eks-decentralized.png)

在上图中,Amazon EKS 集群和其他 AWS 资源部署到各自的工作负载帐户。然后,在 EKS 容器中运行的工作负载使用 IRSA 或 EKS Pod 身份来访问它们的 AWS 资源。

GitOps 是一种管理应用程序和基础设施部署的方式,整个系统在 Git 存储库中以声明方式描述。这是一种操作模型,它为您提供了使用版本控制、不可变的工件和自动化的最佳实践来管理多个 Kubernetes 集群状态的能力。在这个多集群模型中,每个工作负载集群都是使用多个 Git 存储库引导的,允许每个团队(应用程序、平台、安全性等)在集群上部署各自的更改。

您将在每个帐户中使用 [IAM 角色服务帐户 (IRSA)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) 或 [EKS Pod 身份](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html) 来允许您的 EKS 工作负载获取临时 AWS 凭证,以安全地访问其他 AWS 资源。在各自的工作负载 AWS 帐户中创建 IAM 角色,并将它们映射到 k8s 服务帐户,以提供临时 IAM 访问权限。因此,在这种方法中不需要跨帐户访问。请参阅 [IAM 角色服务帐户](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) 文档了解如何在每个工作负载中设置 IRSA,以及 [EKS Pod 身份](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html) 文档了解如何在每个帐户中设置 EKS pod 身份。

### 集中式网络

您还可以利用 AWS RAM 共享 VPC 子网到工作负载帐户,并在这些子网中启动 Amazon EKS 集群和其他 AWS 资源。这实现了集中式网络管理/管理、简化的网络连接和去中心化的 EKS 集群。请参考此 [AWS 博客](https://aws.amazon.com/blogs/containers/use-shared-vpcs-in-amazon-eks/) 了解此方法的详细演练和注意事项。

|![使用 VPC 共享子网的去中心化 EKS 集群架构](./images/multi-account-eks-shared-subnets.png)|
|:--:|
| 在上图中,AWS RAM 用于将子网从中央网络帐户共享到工作负载帐户。然后在这些子网中的各自的工作负载帐户中启动 EKS 集群和其他 AWS 资源。EKS pod 使用 IRSA 或 EKS Pod 身份访问其 AWS 资源。|

## 集中式与去中心化 EKS 集群

是否采用集中式或去中心化运行将取决于您的需求。此表格演示了每种策略的关键差异。

|# |集中式 EKS 集群 | 去中心化 EKS 集群 |
|:--|:--|:--|
|集群管理:  |管理单个 EKS 集群比管理多个集群更容易 | 需要高效的集群管理自动化来减少管理多个 EKS 集群的运营开销|
|成本效率: | 允许重复使用 EKS 集群和网络资源,从而提高成本效率 | 需要为每个工作负载设置网络和集群,这需要额外的资源|
|弹性: | 如果集群出现故障,可能会影响多个工作负载 | 如果集群出现故障,只会影响在该集群上运行的工作负载。其他工作负载不受影响 |

隔离和安全性:
使用 k8s 原生构造(如 `Namespaces`)实现隔离/软多租户。工作负载可能共享底层资源(如 CPU、内存等)。AWS 资源被隔离到自己的工作负载账户中,默认情况下无法从其他 AWS 账户访问。
工作负载在独立的集群和节点上运行,不共享任何资源,因此计算资源的隔离更强。AWS 资源被隔离到自己的工作负载账户中,默认情况下无法从其他 AWS 账户访问。

性能和可扩展性:
随着工作负载规模的不断增大,您可能会遇到集群账户中的 Kubernetes 和 AWS 服务配额。您可以部署更多集群账户来进一步扩展。
随着更多集群和 VPC 的出现,每个工作负载都有更多可用的 k8s 和 AWS 服务配额。

网络:
每个集群使用单个 VPC,使该集群上的应用程序连接更简单。
需要在分散的 EKS 集群 VPC 之间建立路由。

Kubernetes 访问管理:
需要在集群中维护许多不同的角色和用户,以为所有工作负载团队提供访问权限,并确保 Kubernetes 资源得到适当隔离。
简化了访问管理,因为每个集群都专用于一个工作负载/团队。

AWS 访问管理:
AWS 资源部署到自己的账户中,默认情况下只能通过工作负载账户中的 IAM 角色访问。工作负载账户中的 IAM 角色通过 IRSA 或 EKS Pod 身份跨账户进行假设。
AWS 资源部署到自己的账户中,默认情况下只能通过工作负载账户中的 IAM 角色访问。IAM 角色直接通过 IRSA 或 EKS Pod 身份传递给 pod。
