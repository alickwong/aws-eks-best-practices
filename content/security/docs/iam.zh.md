# 身份和访问管理

[身份和访问管理](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html) (IAM) 是一项 AWS 服务,执行两个基本功能:身份验证和授权。身份验证涉及身份的验证,而授权则管理 AWS 资源可以执行的操作。在 AWS 中,资源可以是另一个 AWS 服务,例如 EC2,或者是 [IAM 主体](https://docs.aws.amazon.com/IAM/latest/UserGuide/intro-structure.html#intro-structure-principal),例如 [IAM 用户](https://docs.aws.amazon.com/IAM/latest/UserGuide/id.html#id_iam-users)或 [角色](https://docs.aws.amazon.com/IAM/latest/UserGuide/id.html#id_iam-roles)。管理资源允许执行的操作的规则表示为 [IAM 策略](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html)。

## 控制对 EKS 集群的访问

Kubernetes 项目支持各种不同的策略来验证对 kube-apiserver 服务的请求,例如 Bearer 令牌、X.509 证书、OIDC 等。EKS 目前原生支持 [Webhook 令牌身份验证](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication)、[服务帐户令牌](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#service-account-tokens),以及从 2021 年 2 月 21 日开始的 OIDC 身份验证。

Webhook 身份验证策略调用一个 Webhook 来验证 Bearer 令牌。在 EKS 上,这些 Bearer 令牌是由 AWS CLI 或 [aws-iam-authenticator](https://github.com/kubernetes-sigs/aws-iam-authenticator) 客户端生成的,当您运行 `kubectl` 命令时。当您执行命令时,令牌会传递给 kube-apiserver,后者将其转发给身份验证 Webhook。如果请求格式正确,Webhook 会调用令牌主体中嵌入的预签名 URL。此 URL 验证请求的签名并向 kube-apiserver 返回有关用户的信息,例如用户的帐户、Arn 和 UserId。

要手动生成身份验证令牌,请在终端窗口中键入以下命令:

```bash
aws eks get-token --cluster-name <cluster_name>
```

您也可以以编程方式获取令牌。下面是一个用 Go 编写的示例:

```golang
package main

import (
  "fmt"
  "log"
  "sigs.k8s.io/aws-iam-authenticator/pkg/token"
)

func main()  {
  g, _ := token.NewGenerator(false, false)
  tk, err := g.Get("<cluster_name>")
  if err != nil {
    log.Fatal(err)
  }
  fmt.Println(tk)
}
```

输出应该类似于此:

```json
{
  "kind": "ExecCredential",
  "apiVersion": "client.authentication.k8s.io/v1alpha1",
  "spec": {},
  "status": {
    "expirationTimestamp": "2020-02-19T16:08:27Z",
    "token": "k8s-aws-v1.aHR0cHM6Ly9zdHMuYW1hem9uYXdzLmNvbS8_QWN0aW9uPUdldENhbGxlcklkZW50aXR5JlZlcnNpb249MjAxMS0wNi0xNSZYLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFKTkdSSUxLTlNSQzJXNVFBJTJGMjAyMDAyMTklMkZ1cy1lYXN0LTElMkZzdHMlMkZhd3M0X3JlcXVlc3QmWC1BbXotRGF0ZT0yMDIwMDIxOVQxNTU0MjdaJlgtQW16LUV4cGlyZXM9NjAmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JTNCeC1rOHMtYXdzLWlkJlgtQW16LVNpZ25hdHVyZT0yMjBmOGYzNTg1ZTMyMGRkYjVlNjgzYTVjOWE0MDUzMDFhZDc2NTQ2ZjI0ZjI4MTExZmRhZDA5Y2Y2NDhhMzkz"
  }
}
```

每个令牌都以 `k8s-aws-v1.` 开头,后跟一个 base64 编码的字符串。解码该字符串后,应该类似于以下内容:

```bash
https://sts.amazonaws.com/?Action=GetCallerIdentity&Version=2011-06-15&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=XXXXJPFRILKNSRC2W5QA%2F20200219%2Fus-xxxx-1%2Fsts%2Faws4_request&X-Amz-Date=20200219T155427Z&X-Amz-Expires=60&X-Amz-SignedHeaders=host%3Bx-k8s-aws-id&X-Amz-Signature=XXXf8f3285e320ddb5e683a5c9a405301ad76546f24f28111fdad09cf648a393
```

该令牌由一个预签名的 URL 组成,其中包含 Amazon 凭证和签名。有关更多详细信息,请参见 [https://docs.aws.amazon.com/STS/latest/APIReference/API_GetCallerIdentity.html](https://docs.aws.amazon.com/STS/latest/APIReference/API_GetCallerIdentity.html)。

该令牌的生存时间 (TTL) 为 15 分钟,之后需要生成一个新令牌。当您使用 `kubectl` 等客户端时,这是自动处理的,但是如果您使用 Kubernetes 仪表板,则每次令牌过期时都需要生成一个新令牌并重新进行身份验证。

一旦 AWS IAM 服务验证了用户的身份,kube-apiserver 就会读取 `kube-system` 命名空间中的 `aws-auth` ConfigMap,以确定要与用户关联的 RBAC 组。`aws-auth` ConfigMap 用于在 IAM 主体(即 IAM 用户和角色)和 Kubernetes RBAC 组之间创建静态映射。RBAC 组可以在 Kubernetes RoleBindings 或 ClusterRoleBindings 中引用。它们类似于 IAM 角色,因为它们定义了一组可以对一组 Kubernetes 资源(对象)执行的操作(动词)。

### 集群访问管理器

集群访问管理器是 AWS API 的一项功能,是 EKS v1.23 及更高版本集群(新集群或现有集群)的一项可选功能。它简化了 AWS IAM 和 Kubernetes RBAC 之间的身份映射,消除了在 AWS 和 Kubernetes API 之间切换或编辑 `aws-auth` ConfigMap 进行访问管理的需要,从而减少了操作开销并有助于解决配置错误。该工具还使集群管理员能够自动撤销或细化创建集群的 AWS IAM 主体自动授予的 `cluster-admin` 权限。

此 API 依赖于两个概念:

- **访问条目:** 直接链接到允许对 Amazon EKS 集群进行身份验证的 AWS IAM 主体(用户或角色)的集群身份。
- **访问策略:** 是 Amazon EKS 特定的策略,提供访问条目在 Amazon EKS 集群中执行操作的授权。

> 在发布时,Amazon EKS 仅支持预定义和 AWS 托管的策略。访问策略不是 IAM 实体,而是由 Amazon EKS 定义和管理的。

集群访问管理器允许将上游 RBAC 与访问策略相结合,支持允许和传递(但不拒绝)Kubernetes AuthZ 决策,涉及 API 服务器请求。当上游 RBAC 和 Amazon EKS 授权器都无法确定请求评估的结果时,会发生拒绝决策。

通过此功能,Amazon EKS 支持三种身份验证模式:

1. `CONFIG_MAP` 继续独家使用 `aws-auth` configMap。
2. `API_AND_CONFIG_MAP` 从 EKS 访问条目 API 和 `aws-auth` configMap 中获取经过身份验证的 IAM 主体,优先考虑访问条目。这是将现有 `aws-auth` 权限迁移到访问条目的理想选择。
3. `API` 完全依赖于 EKS 访问条目 API。这是新的**推荐方法**。

要开始使用,集群管理员可以创建或更新 Amazon EKS 集群,将首选身份验证设置为 `API_AND_CONFIG_MAP` 或 `API` 方法,并定义访问条目以授予所需的 AWS IAM 主体访问权限。

```bash
$ aws eks create-cluster \
    --name <CLUSTER_NAME> \
    --role-arn <CLUSTER_ROLE_ARN> \
    --resources-vpc-config subnetIds=<value>,endpointPublicAccess=true,endpointPrivateAccess=true \
    --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}' \
    --access-config authenticationMode=API_AND_CONFIG_MAP,bootstrapClusterCreatorAdminPermissions=false
```

上述命令是一个示例,用于创建一个 Amazon EKS 集群,而不赋予集群创建者管理员权限。

可以使用 `update-cluster-config` 命令更新 Amazon EKS 集群配置以启用 `API` 身份验证模式。要在使用 `CONFIG_MAP` 的现有集群上执行此操作,您必须先更新为 `API_AND_CONFIG_MAP`,然后再更新为 `API`。**这些操作不可逆转**,这意味着无法从 `API` 切换回 `API_AND_CONFIG_MAP` 或 `CONFIG_MAP`,也无法从 `API_AND_CONFIG_MAP` 切换回 `CONFIG_MAP`。

```bash
$ aws eks update-cluster-config \
    --name <CLUSTER_NAME> \
    --access-config authenticationMode=API
```

该 API 支持添加和撤销对集群的访问权限,以及验证指定集群的现有访问策略和访问条目。默认策略是根据 Kubernets RBAC 创建的,如下所示。

| EKS 访问策略 | Kubernetes RBAC |
|--|--|
| AmazonEKSClusterAdminPolicy | cluster-admin |
| AmazonEKSAdminPolicy | admin |
| AmazonEKSEditPolicy | edit |
| AmazonEKSViewPolicy | view |

```bash
$ aws eks list-access-policies
{
    "accessPolicies": [
        {
            "name": "AmazonEKSAdminPolicy",
            "arn": "arn:aws:eks::aws:cluster-access-policy/AmazonEKSAdminPolicy"
        },
        {
            "name": "AmazonEKSClusterAdminPolicy",
            "arn": "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"
        },
        {
            "name": "AmazonEKSEditPolicy",
            "arn": "arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy"
        },
        {
            "name": "AmazonEKSViewPolicy",
            "arn": "arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy"
        }
    ]
}

$ aws eks list-access-entries --cluster-name <CLUSTER_NAME>

{
    "accessEntries": []
}
```

> 当集群在没有集群创建者管理员权限的情况下创建时,不会创建任何访问条目,这是唯一默认创建的条目。

### `aws-auth` ConfigMap _(已弃用)_

Kubernetes 与 AWS 身份验证集成的一种方式是通过 `kube-system` 命名空间中的 `aws-auth` ConfigMap。它负责将 AWS IAM 身份(用户、组和角色)身份验证映射到 Kubernetes 基于角色的访问控制 (RBAC) 授权。`aws-auth` ConfigMap 在 Amazon EKS 集群配置过程中自动创建。它最初是为了允许节点加入您的集群,但如前所述,您也可以使用此 ConfigMap 为 IAM 主体添加 RBAC 访问权限。

要检查您集群的 `aws-auth` ConfigMap,可以使用以下命令。

```bash
kubectl -n kube-system get configmap aws-auth -o yaml
```

这是 `aws-auth` ConfigMap 的默认配置示例。

```yaml
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      - system:node-proxier
      rolearn: arn:aws:iam::<AWS_ACCOUNT_ID>:role/kube-system-<SELF_GENERATED_UUID>
      username: system:node:{{SessionName}}
kind: ConfigMap
metadata:
  creationTimestamp: "2023-10-22T18:19:30Z"
  name: aws-auth
  namespace: kube-system
```

此 ConfigMap 的主要部分位于 `data` 下的 `mapRoles` 块中,由 3 个参数组成。

- **groups:** 将 IAM 角色映射到的 Kubernetes 组。这可以是默认组,也可以是在 `clusterrolebinding` 或 `rolebinding` 中指定的自定义组。在上面的示例中,我们只有系统组声明。
- **rolearn:** 要映射到 Kubernetes 组的 AWS IAM 角色的 ARN,使用以下格式 `arn:<PARTITION>:iam::<AWS_ACCOUNT_ID>:role/role-name`。
- **username:** 在 Kubernetes 中映射到 AWS IAM 角色的用户名。这可以是任何自定义名称。

> 也可以为 AWS IAM 用户定义一个新的配置块 `mapUsers`,在 `data` 中的 `aws-auth` ConfigMap 下,将 **rolearn** 参数替换为 **userarn**,但作为**最佳实践**,始终建议使用 `mapRoles`。

要管理权限,您可以编辑 `aws-auth` ConfigMap,添加或删除对 Amazon EKS 集群的访问权限。虽然可以手动编辑 `aws-auth` ConfigMap,但建议使用工具如 `eksctl`,因为这是一个非常敏感的配置,不准确的配置可能会将您锁定在 Amazon EKS 集群之外。有关更多详细信息,请查看下面的[使用工具对 aws-auth ConfigMap 进行更改](https://aws.github.io/aws-eks-best-practices/security/docs/iam/#use-tools-to-make-changes-to-the-aws-auth-configmap)小节。

## 集群访问建议

### 使 EKS 集群端点私有化

默认情况下,在配置 EKS 集群时,API 集群端点设置为公共的,即可从互联网访问。尽管可从互联网访问,但端点仍被认为是安全的,因为它要求所有 API 请求都通过 IAM 进行身份验证,然后通过 Kubernetes RBAC 进行授权。但是,如果您的公司安全政策要求您限制从互联网访问 API,或者阻止您将流量路由到集群 VPC 之外,您可以:

- 将 EKS 集群端点配置为私有。有关此主题的更多信息,请参见[修改集群端点访问](https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html)。
- 保留公共集群端点,并指定哪些 CIDR 块可以与集群端点通信。这些块实际上是允许访问集群端点的公共 IP 地址的白名单。
- 配置公共访问,设置一组白名单 CIDR 块,并启用私有端点访问。这将允许从特定范围的公共 IP 进行公共访问,同时强制 kubelets(工作节点)与 Kubernetes API 之间的所有网络流量通过在配置控制平面时配置的跨账户 ENI。

### 不要使用服务帐户令牌进行身份验证

服务帐户令牌是一个长期的静态凭证。如果它被泄露、丢失或被盗,攻击者可能能够执行与该令牌关联的所有操作,直到删除该服务帐户。有时,您可能需要为从集群外部使用 Kubernetes API 的应用程序(例如 CI/CD 管道应用程序)授予例外。如果这些应用程序在 AWS 基础设施(如 EC2 实例)上运行,请考虑使用实例配置文件并将其映射到 Kubernetes RBAC 角色。

### 对 AWS 资源实施最低权限访问

IAM 用户不需要被分配 AWS 资源的权限来访问 Kubernetes API。如果您需要授予 IAM 用户访问 EKS 集群的权限,请在 `aws-auth` ConfigMap 中为该用户创建一个条目,将其映射到特定的 Kubernetes RBAC 组。

### 删除集群创建者主体的 cluster-admin 权限

默认情况下,Amazon EKS 集群是使用永久 `cluster-admin` 权限绑定到集群创建者主体创建的。使用集群访问管理器 API,可以在使用 `API_AND_CONFIG_MAP` 或 `API` 身份验证模式时创建没有此权限设置的集群,将 `--access-config bootstrapClusterCreatorAdminPermissions` 设置为 `false`。撤销此访问被认为是最佳实践,以避免对集群配置进行任何意外更改。撤销此访问的过程与撤销对集群的任何其他访问的过程相同。

该 API 使您能够仅取消 IAM 主体与访问策略的关联,在本例中为 `AmazonEKSClusterAdminPolicy`。

```bash
$ aws eks list-associated-access-policies \
    --cluster-name <CLUSTER_NAME> \
    --principal-arn <IAM_PRINCIPAL_ARN>

$ aws eks disassociate-access-policy --cluster-name <CLUSTER_NAME> \
    --principal-arn <IAM_PRINCIPAL_ARN. \
    --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy
```

或者完全删除与 `cluster-admin` 权限关联的访问条目。

```bash
$ aws eks list-access-entries --cluster-name <CLUSTER_NAME>

{
    "accessEntries": []
}

$ aws eks delete-access-entry --cluster-name <CLUSTER_NAME> \
  --principal-arn <IAM_PRINCIPAL_ARN>
```

> 如果需要,可以在事故、紧急情况或紧急情况下重新授予此访问权限,此时集群否则无法访问。

如果集群仍使用 `CONFIG_MAP` 身份验证方法,所有其他用户都应通过 `aws-auth` ConfigMap 被授予对集群的访问权限,并在配置 `aws-auth` ConfigMap 后,可以删除创建集群的实体分配的角色,并仅在事故、紧急情况或紧急情况下或 `aws-auth` ConfigMap 损坏且集群否则无法访问时重新创建该角色。这在生产集群中特别有用。

### 当多个用户需要对集群具有相同的访问权限时,请使用 IAM 角色

不要为每个个人 IAM 用户创建一个条目,而是让这些用户能够承担一个 IAM 角色,并将该角色映射到一个 Kubernetes RBAC 组。这将更容易维护,特别是当需要访问的用户数量增加时。

!!! attention
    当使用 `aws-auth` ConfigMap 映射的 IAM 实体访问 EKS 集群时,Kubernetes 审核日志中会记录描述的用户名。如果您使用 IAM 角色,实际承担该角色的用户不会被记录,无法进行审核。

如果仍在使用 `aws-auth` configMap 作为身份验证方法,在将 K8s RBAC 权限分配给 IAM 角色时,您应该包含 {{SessionName}} 在您的用户名中。这样,审核日志就会记录会话名称,以便您可以跟踪实际用户承担此角色,以及 CloudTrail 日志。

```yaml
- rolearn: arn:aws:iam::XXXXXXXXXXXX:role/testRole
  username: testRole:{{SessionName}}
  groups:
    - system:masters
```

> 在 Kubernetes 1.20 及更高版本中,不再需要此更改,因为 ```user.extra.sessionName.0``` 已添加到 Kubernetes 审核日志中。

### 在创建 RoleBindings 和 ClusterRoleBindings 时实施最低权限访问

与前面关于授予对 AWS 资源的访问权限的观点一样,RoleBindings 和 ClusterRoleBindings 应该只包含执行特定功能所需的一组权限。除非绝对必要,否则请避免在您的角色和 ClusterRoles 中使用 `["*"]`。如果您不确定要分配哪些权限,请考虑使用 [audit2rbac](https://github.com/liggitt/audit2rbac) 之类的工具,根据 Kubernetes 审核日志中观察到的 API 调用自动生成角色和绑定。

### 使用自动化过程创建集群

如前面几个步骤所示,当创建 Amazon EKS 集群时,如果不使用 `API_AND_CONFIG_MAP` 或 `API` 身份验证模式,并且不选择将 `cluster-admin` 权限委托给集群创建者,则创建集群的 IAM 实体用户或角色(如联合用户)会自动被授予集群 RBAC 配置中的 `system:masters` 权限。即使删除此权限是最佳实践,如[从集群创建者主体中删除 cluster-admin 权限](Rremove-the-cluster-admin-permissions-from-the-cluster-creator-principal)所述,如果使用 `CONFIG_MAP` 身份验证方法依赖于 `aws-auth` ConfigMap,此访问权限无法撤销。因此,最好使用与专用 IAM 角色绑定的基础设施自动化管道来创建集群,该角色没有被其他用户或实体假定的权限,并定期审核此角色的权限、策略和谁有权触发管道。此外,此角色不应用于对集群执行日常操作,而应仅用于通过 SCM 代码更改等管道触发的集群级操作。

### 使用专用 IAM 角色创建集群

当您创建 Amazon EKS 集群时,创建集群的 IAM 实体用户或角色(如联合用户)会自动被授予集群 RBAC 配置中的 `system:masters` 权限。此访问权限无法删除,也不通过 `aws-auth` ConfigMap 进行管理。因此,最好使用专用 IAM 角色创建集群,并定期审核谁可以承担此角色。此角色不应用于对集群执行日常操作,而是应通过 `aws-auth` ConfigMap 为其他用户授予对集群的访问权限。在配置 `aws-auth` ConfigMap 之后,应该保护该角色并仅在临时提升权限模式/紧急情况下使用,例如集群否则无法访问的情况。这在没有直接用户访问配置的集群中特别有用。

### 定期审核对集群的访问

需要访问的人可能会随时间而变化。计划定期审核 `aws-auth` ConfigMap,查看谁被授予了访问权限以及分配给他们的权限。您还可以使用开源工具,如 [kubectl-who-can](https://github.com/aquasecurity/kubectl-who-can) 或 [rbac-lookup](https://github.com/FairwindsOps/rbac-lookup) 来检查绑定到特定服务帐户、用户或组的角色。我们将在[审计](detective.md)部分进一步探讨这个主题。您可以在[这篇文章](https://www.nccgroup.trust/us/about-us/newsroom-and-events/blog/2019/august/tools-and-methods-for-auditing-kubernetes-rbac-policies/?mkt_tok=eyJpIjoiWWpGa056SXlNV1E0WWpRNSIsInQiOiJBT1hyUTRHYkg1TGxBV0hTZnRibDAyRUZ0VzBxbndnRzNGbTAxZzI0WmFHckJJbWlKdE5WWDdUQlBrYVZpMnNuTFJ1R3hacVYrRCsxYWQ2RTRcL2pMN1BtRVA1ZFZcL0NtaEtIUDdZV3pENzNLcE1zWGVwUndEXC9Pb2tmSERcL1pUaGUifQ%3D%3D)中找到更多想法。

### 如果依赖于 `aws-auth` configMap,请使用工具进行更改

格式不正确的 aws-auth ConfigMap 可能会导致您无法访问集群。如果您需要对 ConfigMap 进行更改,请使用工具。

**eksctl**
`eksctl` CLI 包括一个用于向 aws-auth ConfigMap 添加身份映射的命令。

查看 CLI 帮助:

```bash
$ eksctl create iamidentitymapping --help
...
```

检查映射到您 Amazon EKS 集群的身份。

```bash
$ eksctl get iamidentitymapping --cluster $CLUSTER_NAME --region $AWS_REGION
ARN                                                                   USERNAME                        GROUPS                                                  ACCOUNT
arn:aws:iam::788355785855:role/kube-system-<SELF_GENERATED_UUID>      system:node:{{SessionName}}     system:bootstrappers,system:nodes,system:node-proxier  
```

使 IAM 角色成为集群管理员:

```bash
$ eksctl create iamidentitymapping --cluster  <CLUSTER_NAME> --region=<region> --arn arn:aws:iam::123456:role/testing --group system:masters --username admin
...
```

有关更多信息,请查看 [`eksctl` 文档](https://eksctl.io/usage/iam-identity-mappings/)

**[keikoproj 的 aws-auth](https://github.com/keikoproj/aws-auth)**

keikoproj 的 `aws-auth` 包括 cli 和 go 库。

下载并查看 CLI 帮助:

```bash
$ go get github.com/keikoproj/aws-auth
...
$ aws-auth help
...
```

或者,使用 [krew 插件管理器](https://krew.sigs.k8s.io)为 kubectl 安装 `aws-auth`。

```bash
$ kubectl krew install aws-auth
...
$ kubectl aws-auth
...
```

[在 GitHub 上查看 aws-auth 文档](https://github.com/keikoproj/aws-auth/blob/master/README.md),了解更多信息,包括 go 库。

**[AWS IAM Authenticator CLI](https://github.com/kubernetes-sigs/aws-iam-authenticator/tree/master/cmd/aws-iam-authenticator)**

`aws-iam-authenticator` 项目包括一个用于更新 ConfigMap 的 CLI。

[在 GitHub 上下载一个版本](https://github.com/kubernetes-sigs/aws-iam-authenticator/releases)。

将集群权限添加到 IAM 角色:

```bash
$ ./aws-iam-authenticator add role --rolearn arn:aws:iam::185309785115:role/lil-dev-role-cluster --username lil-dev-user --groups system:masters --kubeconfig ~/.kube/config
...
```

### 身份验证和访问管理的替代方法

虽然 IAM 是访问 EKS 集群的首选方式,但也可以使用 OIDC 身份提供程序(如 GitHub)通过身份验证代理和 Kubernetes [模拟](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#user-impersonation)。AWS 开源博客上发布了两种这样的解决方案的帖子:

- [使用 Teleport 通过 GitHub 凭据对 EKS 进行身份验证](https://aws.amazon.com/blogs/opensource/authenticating-eks-github-credentials-teleport/)
- [使用 kube-oidc-proxy 在多个 EKS 集群之间保持一致的 OIDC 身份验证](https://aws.amazon.com/blogs/opensource/consistent-oidc-authentication-across-multiple-eks-clusters-using-kube-oidc-proxy/)

!!! attention
    EKS 原生支持 OIDC 身份验证,无需使用代理。有关更多信息,请阅读发布博客[为 Amazon EKS 引入 OIDC 身份提供程序身份验证](https://aws.amazon.com/blogs/containers/introducing-oidc-identity-provider-authentication-amazon-eks/)。有关如何使用 Dex(一个流行的开源 OIDC 提供程序,具有各种不同身份验证方法的连接器)配置 EKS 的示例,请参见[使用 Dex & dex-k8s-authenticator 对 Amazon EKS 进行身份验证](https://aws.amazon.com/blogs/containers/using-dex-dex-k8s-authenticator-to-authenticate-to-amazon-eks/)。如博客所述,通过 OIDC 提供程序验证的用户名/组将出现在 Kubernetes 审核日志中。

您还可以使用 [AWS SSO](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html) 将 AWS 与外部身份提供程序(如 Azure AD)联合。如果您决定使用它,AWS CLI v2.0 包括一个选项,可以创建一个命名配置文件,使您可以轻松地将 SSO 会话与您当前的 CLI 会话相关联并承担一个 IAM 角色。请注意,您必须在运行 `kubectl` 之前先承担一个角色,因为 IAM 角色用于确定用户的 Kubernetes RBAC 组。

## EKS Pods 的身份和凭据

某些在 Kubernetes 集群中运行的应用程序需要权限来调用 Kubernetes API,才能正常工作。例如, [AWS Load Balancer Controller](https://github.com/kubernetes-sigs/aws-load-balancer-controller) 需要能够列出服务的端点。该控制器还需要能够调用 AWS API 来配置和配置 ALB。在本节中,我们将探讨为 Pods 分配权限和特权的最佳实践。

### Kubernetes 服务帐户

服务帐户是一种特殊类型的对象,允许您将 Kubernetes RBAC 角色分配给 pod。每个命名空间中都会自动创建一个默认服务帐户。当您在不引用特定服务帐户的情况下将 pod 部署到命名空间中时,该命名空间的默认服务帐户将自动分配给 Pod,并且服务帐户(JWT)令牌将作为卷挂载到 pod 中的 `/var/run/secrets/kubernetes.io/serviceaccount` 目录。解码该目录中的服务帐户令牌将显示以下元数据:

```json
{
  "iss": "kubernetes/serviceaccount",
  "kubernetes.io/serviceaccount/namespace": "default",
  "kubernetes.io/serviceaccount/secret.name": "default-token-5pv4z",
  "kubernetes.io/serviceaccount/service-account.name": "default",
  "kubernetes.io/serviceaccount/service-account.uid": "3b36ddb5-438c-11ea-9438-063a49b60fba",
  "sub": "system:serviceaccount:default:default"
}
```

默认服务帐户对 Kubernetes API 具有以下权限。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2020-01-30T18:13:25Z"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:discovery
  resourceVersion: "43"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterroles/system%3Adiscovery
  uid: 350d2ab8-438c-11ea-9438-063a49b60fba
rules:
- nonResourceURLs:
  - /api
  - /api/*
  - /apis
  - /apis/*
  - /healthz
  - /openapi
  - /openapi/*
  - /version
  - /version/
  verbs:
  - get
```

此角色授权未经身份验证和经过身份验证的用户读取 API 信息,被认为是安全的公开访问。

当应用程序在 Pod 中调用 Kubernetes API 时,Pod 需要被分配一个服务帐户,该服务帐户明确授予它调用这些 API 的权限。与用户访问指南类似,绑定到服务帐户的角色或 ClusterRole 应该仅限于应用程序需要的 API 资源和方法,而不是其他资源。要使用非默认服务帐户,只需将 Pod 的 `spec.serviceAccountName` 字段设置为您要使用的服务帐户的名称。有关创建服务帐户的更多信息,请参见 [https://kubernetes.io/docs/reference/access-authn-authz/rbac/#service-account-permissions](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#service-account-permissions)。

!!! note
    在 Kubernetes 1.24 之前,Kubernetes 会自动为每个服务帐户创建一个密钥。此密钥被挂载到 pod 的 /var/run/secrets/kubernetes.io/serviceaccount 中,并由 pod 用于对 Kubernetes API 服务器进行身份验证。在 Kubernetes 1.24 中,当 pod 运行时会动态生成一个服务帐户令牌,该令牌默认有效期为 1 小时。不会创建服务帐户的密钥。如果您有一个在集群外运行的应用程序需要对 Kubernetes API 进行身份验证,例如 Jenkins,您需要创建一个类型为 `kubernetes.io/service-account-token` 的密钥,并添加一个引用服务帐户名称的注释,如 `metadata.annotations.kubernetes.io/service-account.name: <SERVICE_ACCOUNT_NAME>`。这种方式创建的密钥不会过期。

### IAM 角色的服务帐户 (IRSA)

IRSA 是一项功能,允许您将 IAM 角色分配给 Kubernetes 服务帐户。它利用 Kubernetes 的一项功能,称为[服务帐户令牌卷投射](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#serviceaccount-token-volume-projection)。当 Pod 配置了引用 IAM 角色的服务帐户时,Kubernetes API 服务器将在启动时调用集群的公共 OIDC 发现端点。端点以加密方式签署 Kubernetes 颁发的 OIDC 令牌,并将其挂载为卷。此签名令牌允许 Pod 调用与 IAM 角色关联的 AWS API。当调用 AWS API 时,AWS SDK 会调用 `sts:AssumeRoleWithWebIdentity`。在验证令牌的签名后,IAM 会交换 Kubernetes 颁发的令牌以获取临时的 AWS 角色凭证。

使用 IRSA 时,重要的是[重复使用 AWS SDK 会话](#重复使用-irsa-的-aws-sdk-会话)以避免不必要地调用 AWS STS。

解码 IRSA 的(JWT)令牌将产生类似于以下示例的输出:

```json
{
  "aud": [
    "sts.amazonaws.com"
  ],
  "exp": 1582306514,
  "iat": 1582220114,
  "iss": "https://oidc.eks.us-west-2.amazonaws.com/id/D43CF17C27A865933144EA99A26FB128",
  "kubernetes.io": {
    "namespace": "default",
    "pod": {
      "name": "alpine-57b5664646-rf966",
      "uid": "5a20f883-5407-11ea-a85c-0e62b7a4a436"
    },
    "serviceaccount": {
      "name": "s3-read-only",
      "uid": "a720ba5c-5406-11ea-9438-063a49b60fba"
    }
  },
  "nbf": 1582220114,
  "sub": "system:serviceaccount:default:s3-read-only"
}
```

这个特定的令牌通过假定一个 IAM 角色来授予 Pod 对 S3 的只读权限。当应用程序尝试从 S3 读取时,该令牌将被交换为临时的 IAM 凭证,类似于以下内容:

```json
{
    "AssumedRoleUser": {
        "AssumedRoleId": "AROA36C6WWEJULFUYMPB6:abc",
        "Arn": "arn:aws:sts::123456789012:assumed-role/eksctl-winterfell-addon-iamserviceaccount-de-Role1-1D61LT75JH3MB/abc"
    },
    "Audience": "sts.amazonaws.com",
    "Provider": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-west-2.amazonaws.com/id/D43CF17C27A865933144EA99A26FB128",
    "SubjectFromWebIdentityToken": "system:serviceaccount:default:s3-read-only",
    "Credentials": {
        "SecretAccessKey": "ORJ+8Adk+wW+nU8FETq7+mOqeA8Z6jlPihnV8hX1",
        "SessionToken": "FwoGZXIvYXdzEGMaDMLxAZkuLpmSwYXShiL9A1S0X87VBC1mHCrRe/pB2oes+l1eXxUYnPJyC9ayOoXMvqXQsomq0xs6OqZ3vaa5Iw1HIyA4Cv1suLaOCoU3hNvOIJ6C94H1vU0siQYk7DIq9Av5RZe+uE2FnOctNBvYLd3i0IZo1ajjc00yRK3v24VRq9nQpoPLuqyH2jzlhCEjXuPScPbi5KEVs9fNcOTtgzbVf7IG2gNiwNs5aCpN4Bv/Zv2A6zp5xGz9cWj2f0aD9v66vX4bexOs5t/YYhwuwAvkkJPSIGvxja0xRThnceHyFHKtj0H+bi/PWAtlI8YJcDX69cM30JAHDdQH+ltm/4scFptW1hlvMaP+WReCAaCrsHrAT+yka7ttw5YlUyvZ8EPog+j6fwHlxmrXM9h1BqdikomyJU00gm1++FJelfP+1zAwcyrxCnbRl3ARFrAt8hIlrT6Vyu8WvWtLxcI8KcLcJQb/LgkW+sCTGlYcY8z3zkigJMbYn07ewTL5Ss7LazTJJa758I7PZan/v3xQHd5DEc5WBneiV3iOznDFgup0VAMkIviVjVCkszaPSVEdK2NU7jtrh6Jfm7bU/3P6ZG+CkyDLIa8MBn9KPXeJd/y+jTk5Ii+fIwO/+mDpGNUribg6TPxhzZ8b/XdZO1kS1gVgqjXyVC+M+BRBh6C4H21w/eMzjCtDIpoxt5rGKL6Nu/IFMipoC4fgx6LIIHwtGYMG7SWQi7OsMAkiwZRg0n68/RqWgLzBt/4pfjSRYuk=",
        "Expiration": "2020-02-20T18:49:50Z",
        "AccessKeyId": "XXXX36C6WWEJUMHA3L7Z"
    }
}
```

EKS 控制平面的一个变异 Webhook 将 AWS 角色 ARN 和 Web 身份令牌文件的路径注入到 Pod 作为环境变量。这些值也可以手动提供。

```bash
AWS_ROLE_ARN=arn:aws:iam::AWS_ACCOUNT_ID:role/IAM_ROLE_NAME
AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token
```

kubelet 将在令牌超过其总 TTL 的 80% 或 24 小时后自动轮换投射的令牌。AWS SDK 负责在令牌轮换时重新加载令牌。有关 IRSA 的更多信息,请参见 [https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts-technical-overview.html](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts-technical-overview.html)。

### EKS Pod 身份

[EKS Pod 身份](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)是一项在 re:Invent 2023 推出的功能,允许您将 IAM 角色分配给 kubernetes 服务帐户,而无需为您的 AWS 帐户中的每个集群配置 Open Id Connect (OIDC) 身份提供程序(IDP)。要使用 EKS Pod 身份,您必须部署一个代理,该代理作为 DaemonSet pod 在每个合格的工作节点上运行。此代理作为 EKS 附加组件提供给您,是使用 EKS Pod 身份功能的先决条件。您的应用程序必须使用[受支持版本的 AWS SDK](https://docs.aws.amazon.com/eks/latest/userguide/pod-id-minimum-sdk.html)才能使用此功能。

为 Pod 配置 EKS Pod 身份时,EKS 将在 `/var/run/secrets/pods.eks.amazonaws.com/serviceaccount/eks-pod-identity-token` 挂载和刷新 pod 身份令牌。此令牌将由 AWS SDK 用于与 EKS Pod 身份代理通信,EKS Pod 身份代理使用 pod 身份令牌和代理的 IAM 角色来创建 pod 的临时凭证,方法是调用 [AssumeRoleForPodIdentity API](https://docs.aws.amazon.com/eks/latest/APIReference/API_auth_AssumeRoleForPodIdentity.html)。传递给 pod 的 pod 身份令牌是从您的 EKS 集群颁发的 JWT,并经过加密签名,具有适用于 EKS Pod 身份的 JWT 声明。

要了解有关 EKS Pod 身份的更多信息,请参见[此博客](https://aws.amazon.com/blogs/containers/amazon-eks-pod-identity-a-new-way-for-applications-on-eks-to-obtain-iam-credentials/)。

您不需要对您的应用程序代码进行任何修改即可使用 EKS Pod 身份。受支持的 AWS SDK 版本将通过使用[凭证提供程序链](https://docs.aws.amazon.com/sdkref/latest/guide/standardized-credentials.html)自动发现通过 EKS Pod 身份提供的凭证。与 IRSA 一样,EKS pod 身份在您的 pod 中设置变量,指导它们如何查找 AWS 凭证。

#### 使用 EKS Pod 身份的 IAM 角色

- EKS Pod 身份只能直接承担与 EKS 集群相同 AWS 帐户中的 IAM 角色。要访问另一个 AWS 帐户中的 IAM 角色,您必须通过[在 SDK 配置中配置配置文件](https://docs.aws.amazon.com/sdkref/latest/guide/feature-assume-role-credentials.html)或[应用程序代码](https://docs.aws.amazon.com/IAM/latest/UserGuide/sts_example_sts_AssumeRole_section.html)中承担该角色。
- 在为服务帐户配置 EKS Pod 身份时,配置 Pod 身份关联的人员或过程必须对该角色具有 `iam:PassRole` 授权。
- 每个服务帐户只能通过 EKS Pod 身份与一个 IAM 角色关联,但您可以将同一 IAM 角色与多个服务帐户关联。
- 与 EKS Pod 身份一起使用的 IAM 角色必须允许 `pods.eks.amazonaws.com` 服务主体承担它们,_并且_设置会话标签。以下是一个示例角色信任策略,允许 EKS Pod 身份使用 IAM 角色:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "pods.eks.amazonaws.com"
      },
      "Action": [
        "sts:AssumeRole",
        "sts:TagSession"
      ],
      "Condition": {
        "StringEquals": {
          "aws:SourceOrgId": "${aws:ResourceOrgId}"
        }
      }
    }
  ]
}
```

AWS 建议使用像 `aws:SourceOrgId` 这样的条件键来帮助防范[跨服务混淆代理问题](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html#cross-service-confused-deputy-prevention)。在上面的示例角色信任策略中,`ResourceOrgId` 是一个等于 AWS 组织 ID 的变量,AWS 组织是 AWS 帐户所属的组织。EKS 将在使用 EKS Pod 身份承担角色时传入 `aws:SourceOrgId` 的值。

#### ABAC 和 EKS Pod 身份

当 EKS Pod 身份承担 IAM 角色时,它会设置以下会话标签:

|EKS Pod 身份会话标签 | 值 |
|:--|:--|
|kubernetes-namespace | pod 与 EKS Pod 身份关联的命名空间。|
|kubernetes-service-account | 与 EKS Pod 身份关联的 kubernetes 服务帐户名称|
|eks-cluster-arn | EKS 集群的 ARN,例如 `arn:${Partition}:eks:${Region}:${Account}:cluster/${ClusterName}`。集群 ARN 是唯一的,但如果在同一 AWS 帐户中的同一区域内删除并重新创建具有相同名称的集群,它将具有相同的 ARN。 |
|eks-cluster-name | EKS 集群的名称。请注意,EKS 集群名称在您的 AWS 帐户中可能相同,而在其他 AWS 帐户中的 EKS 集群也可能相同。 |
|kubernetes-pod-name | EKS 中 pod 的名称。 |
|kubernetes-pod-uid | EKS 中 pod 的 UID。 |

这些会话标签允许您使用[属性基础访问控制(ABAC)](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction_attribute-based-access-control.html)来授予对您的 AWS 资源的访问权限,仅限于特定的 kubernetes 服务帐户。在这样做时,_非常重要_的是要理解 kubernetes 服务帐户只在命名空间内是唯一的,而 kubernetes 命名空间只在 EKS 集群内是唯一的。可以通过使用全局条件键 `aws:PrincipalTag/<tag-key>` 访问这些会话标签,例如 `aws:PrincipalTag/eks-cluster-arn`

例如,如果您想授予对您帐户中某个 AWS 资源的访问权限,仅限于特定的服务帐户,您需要同时检查 `eks-cluster-arn` 和 `kubernetes-namespace` 标签以及 `kubernetes-service-account`,以确保只有来自预期集群的该服务帐户才能访问该资源,因为其他集群可能具有相同的 `kubernetes-service-accounts` 和 `kubernetes-namespaces`。

这个示例 S3 Bucket 策略仅授予对附加到它的 S3 存储桶中对象的访问权限,前提是 `kubernetes-service-account`、`kubernetes-namespace`、`eks-cluster-arn` 都满足其预期值,其中 EKS 集群托管在 AWS 帐户 `111122223333` 中。

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::111122223333:root"
            },
            "Action": "s3:*",
            "Resource":            [
                "arn:aws:s3:::ExampleBucket/*"
            ],
            "Condition": {
                "StringEquals": {
                    "aws:PrincipalTag/kubernetes-service-account": "s3objectservice",
                    "aws:PrincipalTag/eks-cluster-arn": "arn:aws:eks:us-west-2:111122223333:cluster/ProductionCluster",
                    "aws:PrincipalTag/kubernetes-namespace": "s3datanamespace"
                }
            }
        }
    ]
}
```

### EKS Pod 身份与 IRSA 的比较

EKS Pod 身份和 IRSA 都是向 EKS pod 提供临时 AWS 凭证的首选方式。除非您有特定的 IRSA 用例,否则我们建议您在使用 EKS 时使用 EKS Pod 身份。下表有助于比较这两个功能。

|# |EKS Pod 身份 | IRSA |
|:--|:--|:--|
|需要在您的 AWS 帐户中创建 OIDC IDP 的权限吗?|否|是|
|每个集群需要唯一的 IDP 设置吗 |否|是|
|为 ABAC 设置相关会话标签|是|否|
|需要 iam:PassRole 检查吗?|是| 否 |
|使用您 AWS 帐户的 AWS STS 配额?|否|是|
|可以访问其他 AWS 帐户 | 通过角色链接间接 | 通过 sts:AssumeRoleWithWebIdentity 直接 |
|与 AWS SDK 兼容 |是|是|
|需要 Pod 身份代理 Daemonset 在节点上吗? |是|否|

## EKS Pods 的身份和凭据建议

### 将 aws-node daemonset 更新为使用 IRSA

目前,aws-node daemonset 配置为使用分配给 EC2 实例的角色来分配 IP 地址到 pod。此角色包括几个 AWS 托管策略,例如 AmazonEKS_CNI_Policy 和 EC2ContainerRegistryReadOnly,实际上允许在节点上运行的**所有** pod 附加/分离 ENI、分配/取消分配 IP 地址或从 ECR 拉取镜像。由于这会给您的集群带来风险,建议您将 aws-node daemonset 更新为使用 IRSA。可以在此指南的[存储库](https://github.com/aws/aws-eks-best-practices/tree/master/projects/enable-irsa/src)中找到一个脚本来执行此操作。

aws-node daemonset 目前不支持 EKS Pod 身份。

### 限制分配给工作节点的实例配置文件的访问权限

当您使用 IRSA 或 EKS Pod 身份时,它会更新 pod 的凭证链,以先使用 IRSA 或 EKS Pod 身份,但 pod _仍然可以继承分配给工作节点的实例配置文件的权限_。使用 IRSA 或 EKS Pod 身份时,**强烈**建议您阻止对[实例元数据](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html)的访问,以确保您的应用程序只拥有它们所需的权限,而不是它们的节点。

!!! caution
    阻止对实例元数据的访问将阻止不使用 IRSA 或 EKS Pod 身份的 pod 继承分配给工作节点的角色。

您可以通过要求实例仅使用 IMDSv2 并更新跳数到 1 来阻止对实例元数据的访问,如下例所示。您也可以将这些设置包含在节点组的启动模板中。**不要**禁用实例元数据,因为这将阻止节点终止处理程序和其他依赖于实例元数据的组件正常工作。

```bash
$ aws ec2 modify-instance-metadata-options --instance-id <value> --http-tokens required --http-put-response-hop-limit 1
...
```

如果您使用 Terraform 为托管节点组创建启动模板,请将元数据块添加到配置跳数,如此代码片段所示:

``` tf hl_lines="7"
resource "aws_launch_template" "foo" {
  name = "foo"
  ...
    metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"
    http_put_response_hop_limit = 1
    instance_metadata_tags      = "enabled"
  }
  ...
```

您还可以通过操纵节点上的 iptables 来阻止 pod 访问 EC2 元数据。有关此方法的更多信息,请参见[限制对实例元数据服务的访问](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html#instance-metadata-limiting-access)。

如果您有一个使用不支持 IRSA 或 EKS Pod 身份的较旧版本 AWS SDK 的应用程序,您应该更新 SDK 版本。

### 将 IRSA 角色的信任策略范围缩小到服务帐户名称、命名空间和集群

信任策略可以缩小到命名空间或命名空间中的特定服务帐户。使用 IRSA 时,最好尽可能明确地制定角色信任策略,包括服务帐户名称。这将有效地防止同一命名空间中的其他 Pod 承担该角色。`eksctl` 会自动执行此操作,当您使用它创建服务帐户/IAM 角色时。有关更多信息,请参见 [https://eksctl.io/usage/iamserviceaccounts/](https://eksctl.io/usage/iamserviceaccounts/)。

在直接使用 IAM 时,这是在角色的信任策略中添加条件,以确保 `:sub` 声明是您所期望的命名空间和服务帐户。例如,在我们有一个 IRSA 令牌的 sub 声明为"system:serviceaccount:default:s3-read-only"之前。这是 `default` 命名空间,服务帐户是 `s3-read-only`。您将使用以下条件来确保只有您的集群中给定命名空间的服务帐户才能承担该角色:

```json
  "Condition": {
      "StringEquals": {
          "oidc.eks.us-west-2.amazonaws.com/id/D43CF17C27A865933144EA99A26FB128:aud": "sts.amazonaws.com",
          "oidc.eks.us-west-2.amazonaws.com/id/D43CF17C27A865933144EA99A26FB128:sub": "system:serviceaccount:default:s3-read-only"
      }
  }
```

### 为每个应用程序使用一个 IAM 角色

对于 IRSA 和 EKS Pod 身份,最佳实践是为每个应用程序分配一个自己的 IAM 角色。这可以提高隔离,因为您可以修改一个应用程序而不影响另一个应用程序,并允许您通过仅授予应用程序所需的权限来应用最小特权原则。

使用 EKS Pod 身份时,如果您使用 ABAC,您可以跨多个服务帐户使用一个通用的 IAM 角色,并依赖于它们的会话属性进行访问控制。这在大规模操作时特别有用,因为 ABAC 允许您使用更少的 IAM 角色进行操作。

### 当您的应用程序需要访问 IMDS 时,请使用 IMDSv2 并将 EC2 实例的跳数增加到 2

[IMDSv2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html) 要求您使用 PUT 请求来获取会话令牌。初始 PUT 请求必须包含会话令牌的 TTL。较新版本的 AWS SDK 将自动处理这个问题以及对该令牌的续订。您还需要注意,EC2 实例上的默认跳数限制为 1,以防止 IP 转发。因此,在 EC2 实例上运行的 Pod 请求会话令牌可能最终会超时,并回退到使用 IMDSv1 数据流。EKS 通过_启用_v1 和 v2 并将跳数更改为 2 来添加对 IMDSv2 的支持,这是由 eksctl 或官方 CloudFormation 模板配置的节点。

### 禁用自动挂载服务帐户令牌

如果您的应用程序不需要调用 Kubernetes API,请在应用程序的 PodSpec 中将 `automountServiceAccountToken` 属性设置为 `false`,或者修补每个命名空间中的默认服务帐户,使其不会自动挂载到 pod 上。例如:

```bash
kubectl patch serviceaccount default -p $'automountServiceAccountToken: false'
```

### 为每个应用程序使用专用的服务帐户

每个应用程序都应该有自己专用的服务帐户。这适用于 Kubernetes API 的服务帐户以及 IRSA 和 EKS Pod 身份。

!!! attention
    如果您在使用 IRSA 时采用蓝/绿集群升级方法,而不是执行就地集群升级,您将需要更新每个 IRSA IAM 角色的信任策略,以包含新集群的 OIDC 端点。蓝/绿集群升级是指您创建一个运行较新版本 Kubernetes 的集群,并将其与旧集群并行运行,然后使用负载均衡器或服务网格无缝地将流量从在旧集群上运行的服务转移到新集群。
    在使用蓝/绿集群升级时使用 EKS Pod 身份,您将在新集群中创建 pod 身份关联,并更新 IAM 角色信任策略(如果您有 `sourceArn` 条件)。

### 以非 root 用户运行应用程序

容器默认以 root 用户运行。虽然这允许它们读取 Web 身份令牌文件,但以 root 用户运行容器不被视为最佳实践。作为替代方案,请考虑在 PodSpec 中添加 `spec.securityContext.runAsUser` 属性。`runAsUser` 的值是任意值。

在以下示例中,Pod 中的所有进程都将在 `runAsUser` 字段中指定的用户 ID 下运行。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
  containers:
  - name: sec-ctx-demo
    image: busybox
    command: [ "sh", "-c", "sleep 1h" ]
```

当您以非 root 用户运行容器时,它会阻止容器读取 IRSA 服务帐户令牌,因为该令牌默认具有 0600 [root] 权限。如果您将容器的 securityContext 更新为包括 fsgroup=65534 [Nobody],它将允许容器读取令牌。

```yaml
spec:
  securityContext:
    fsGroup: 65534
```

在 Kubernetes 1.19 及更高版本中,不再需要此更改,应用程序可以在不将它们添加到 Nobody 组的情况下读取 IRSA 服务帐户令牌。

### 授予应用程序最低特权访问权限

[Action Hero](https://github.com/princespaghetti/actionhero) 是一个您可以与应用程序一起运行的实用程序,用于识别应用程序需要的 AWS API 调用和相应的 IAM 权限。它类似于 [IAM Access Advisor](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_access-advisor.html),因为它可以帮助您逐步限制分配给应用程序的 IAM 角色的范围。有关授予 AWS 资源[最低特权访问](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#grant-least-privilege)的更多信息,请参阅文档。

考虑在 IRSA 和 Pod 身份使用的 IAM 角色上设置[权限边界](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html)。您可以使用权限边界来确保 IRSA 或 Pod 身份使用的角色不能超过最大权限级别。有关使用权限边界的示例指南和示例权限边界策略,请参见[此 GitHub 存储库](https://github.com/aws-samples/example-permissions-boundary)。

### 检查并撤销对您的 EKS 集群的不必要的匿名访问

理想情况下,应该禁用所有 API 操作的匿名访问。匿名访问是通过为 Kubernetes 内置用户 system:anonymous 创建 RoleBinding 或 ClusterRoleBinding 来授予的。您可以使用 [rbac-lookup](https://github.com/FairwindsOps/rbac-lookup) 工具识别 system:anonymous 用户在集群上拥有的权限:

```bash
./rbac-lookup | grep -P 'system:(anonymous)|(unauthenticated)'
system:anonymous               cluster-wide        ClusterRole/system:discovery
system:unauthenticated         cluster-wide        ClusterRole/system:discovery
system:unauthenticated         cluster-wide        ClusterRole/system:public-info-viewer
```

除了 system:public-info-viewer 之外的任何角色或 ClusterRole 都不应绑定到 system:anonymous 用户或 system:unauthenticated 组。

可能有一些合法的理由允许在特定 API 上启用匿名访问。如果这是您集群的情况,请确保只有那些特定的 API 可以由匿名用户访问,并且在没有身份验证的情况下公开这些 API 不会使您的集群容易受到攻击。

在 Kubernetes/EKS 版本 1.14 之前,system:unauthenticated 组默认与 system:discovery 和 system:basic-user ClusterRoles 相关联。请注意,即使您已将集群更新到 1.14 或更高版本,这些权限可能仍然启用在您的集群上,因为集群更新不会撤销这些权限。
要检查除 system:public-info-viewer 之外哪些 ClusterRoles 有 "system:unauthenticated",您可以运行以下命令(需要 jq 实用程序):

```bash
kubectl get ClusterRoleBinding -o json | jq -r '.items[] | select(.subjects[]?.name =="system:unauthenticated") | select(.metadata.name != "system:public-info-viewer") | .metadata.name'
```

并且可以从除 "system:public-info-viewer" 之外的所有角色中删除 "system:unauthenticated",如下所示:

```bash
kubectl get ClusterRoleBinding -o json | jq -r '.items[] | select(.subjects[]?.name =="system:unauthenticated") | select(.metadata.name != "system:public-info-viewer") | del(.subjects[] | select(.name =="system:unauthenticated"))' | kubectl apply -f -
```

或者,您可以手动检查并删除它,使用 kubectl describe 和 kubectl edit。要检查 system:unauthenticated 组是否在您的集群上拥有 system:discovery 权限,请运行以下命令:

```bash
kubectl describe clusterrolebindings system:discovery

Name:         system:discovery
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
Role:
  Kind:  ClusterRole
  Name:  system:discovery
Subjects:
  Kind   Name                    Namespace
  ----   ----                    ---------
  Group  system:authenticated
  Group  system:unauthenticated
```

要检查 system:unauthenticated 组是否在您的集群上拥有 system:basic-user 权限,请运行以下命令:

```bash
kubectl describe clusterrolebindings system:basic-user

Name:         system:basic-user
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate: true
Role:
  Kind:  ClusterRole
  Name:  system:basic-user
Subjects:
  Kind   Name                    Namespace
  ----   ----                    ---------
  Group  system:authenticated
  Group  system:unauthenticated
```

如果 system:unauthenticated 组绑定到您集群上的 system:discovery 和/或 system:basic-user ClusterRoles,您应该取消这些角色与 system:unauthenticated 组的关联。使用以下命令编辑 system:discovery ClusterRoleBinding:

```bash
kubectl edit clusterrolebindings system:discovery
```

上述命令将在编辑器中打开 system:discovery ClusterRoleBinding 的当前定义,如下所示:

```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: "2021-06-17T20:50:49Z"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:discovery
  resourceVersion: "24502985"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/system%3Adiscovery
  uid: b7936268-5043-431a-a0e1-171a423abeb6
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:discovery
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:authenticated
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:unauthenticated
```

从上述编辑器屏幕的"subjects"部分删除 system:unauthenticated 组的条目。

对 system:basic-user ClusterRoleBinding 重复相同的步骤。

### 重复使用 IRSA 的 AWS SDK 会话

当您使用 IRSA 时,使用 AWS SDK 编写的应用程序使用传递到 pod 的令牌调用 `sts:AssumeRoleWithWebIdentity` 来生成临时 AWS 凭证。这与其他 AWS 计算服务不同,在其他 AWS 计算服务中,计算服务直接向 AWS 计算资源(如 lambda 函数)传递临时 AWS 凭证。这意味着每次初始化 AWS SDK 会话时,都会调用 AWS STS 进行 `AssumeRoleWithWebIdentity`。如果您的应用程序快速扩展并初始化许多 AWS SDK 会话,您可能会遇到来自 AWS STS 的节流,因为您的代码将进行大量 `AssumeRoleWithWebIdentity` 调用。

为了避免这种情况,我们建议在应用程序中重复使用 AWS SDK 会话,以避免不必要的 `AssumeRoleWithWebIdentity` 调用。

在以下示例代码中,使用 boto3 python SDK 创建一个会话,并使用该会话创建客户端与 Amazon S3 和 Amazon SQS 进行交互。`AssumeRoleWithWebIdentity` 只调用一次,AWS SDK 将在凭证过期时自动刷新 `my_session` 的凭证。

```py hl_lines="4 7 8"  
import boto3

# 创建您自己的会话
my_session = boto3.session.Session()

# 现在我们可以从我们的会话创建低级客户端
sqs = my_session.client('sqs')
s3 = my_session.client('s3')

s3response = s3.list_buckets()
sqsresponse = sqs.list_queues()


#打印来自 S3 和 SQS API 的响应
print("s3 response:")
print(s3response)
print("---")
print("sqs response:")
print(sqsresponse)
```

如果您正在将应用程序从另一个 AWS 计算服务(如 EC2)迁移到使用 IRSA 的 EKS,这个细节尤其重要。在其他计算服务上,初始化 AWS SDK 会话不会调用 AWS STS,除非您指示它这样做。

### 替代方法

虽然 IRSA 和 EKS Pod 身份是_首选方式_来为 pod 分配 AWS 身份,但它们要求您在应用程序中包含 AWS SDK 的最新版本。有关当前支持 IRSA 的 SDK 的完整列表,请参见 [https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts-minimum-sdk.html](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts-minimum-sdk.html),对于 EKS Pod 身份,请参见 [https://docs.aws.amazon.com/eks/latest/userguide/pod-id-minimum-sdk.html](https://docs.aws.amazon.com/eks/latest/userguide/pod-id-minimum-sdk.html)。如果您有一个无法立即使用兼容 SDK 的应用程序,社区中有几个可用的解决方案,用于将 IAM 角色分配给 Kubernetes pod,包括 [kube2iam](https://github.com/jtblin/kube2iam) 和 [kiam](https://github.com/uswitch/kiam)。尽管 AWS 不认可、不赞同或支持使用这些解决方案,但它们被广泛用于社区中,以实现与 IRSA 和 EKS Pod 身份类似的结果。

如果您需要使用这些非 AWS 提供的解决方案,请仔细调查并确保您了解这样做的安全影响。

## 工具和资源

- [Amazon EKS 安全沉浸式研讨会 - 身份和访问管理](https://catalog.workshops.aws/eks-security-immersionday/en-US/2-identity-and-access-management)
- [Terraform EKS 蓝图模式 - 完全私有的 Amazon EKS 集群](https://github.com/aws-ia/terraform-aws-eks-blueprints/tree/main/patterns/fully-private-cluster)
- [Terraform EKS 蓝图模式 - IAM Identity Center 单点登录 Amazon EKS 集群](https://github.com/aws-ia/terraform-aws-eks-blueprints/tree/main/patterns/sso-iam-identity-center)
- [Terraform EKS 蓝图模式 - Okta 单点登录 Amazon EKS 集群](https://github.com/aws-ia/terraform-aws-eks-blueprints/tree/main/patterns/sso-okta)
- [audit2rbac](https://github.com/liggitt/audit2rbac)
- [rbac.dev](https://github.com/mhausenblas/rbac.dev) Kubernetes RBAC 的其他资源列表,包括博客和工具
- [Action Hero](https://github.com/princespaghetti/actionhero)
- [kube2iam](https://github.com/jtblin/kube2iam)
- [kiam](https://github.com/uswitch/kiam)