!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。
[身份和访问管理]

[身份和访问管理](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html)(IAM)是AWS服务执行两个基本功能:身份验证和授权。身份验证涉及身份的验证,而授权则管理AWS资源可以执行的操作。在AWS中,资源可以是另一个AWS服务,例如EC2,或者是[IAM用户](https://docs.aws.amazon.com/IAM/latest/UserGuide/id.html#id_iam-users)或[角色](https://docs.aws.amazon.com/IAM/latest/UserGuide/id.html#id_iam-roles)等[主体](https://docs.aws.amazon.com/IAM/latest/UserGuide/intro-structure.html#intro-structure-principal)。管理资源允许执行的操作的规则表述为[IAM策略](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html)。

## 控制对EKS集群的访问

Kubernetes项目支持多种不同的策略来验证对kube-apiserver服务的请求,例如Bearer令牌、X.509证书、OIDC等。EKS目前原生支持[Webhook令牌身份验证](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication)、[服务帐户令牌](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#service-account-tokens),并且从2021年2月21日开始支持OIDC身份验证。

Webhook身份验证策略调用一个Webhook来验证Bearer令牌。在EKS上,这些Bearer令牌是由AWS CLI或[aws-iam-authenticator](https://github.com/kubernetes-sigs/aws-iam-authenticator)客户端在运行`kubectl`命令时生成的。在执行命令时,令牌被传递给kube-apiserver,然后kube-apiserver将其转发给身份验证Webhook。如果请求格式正确,Webhook将调用令牌主体中嵌入的预签名URL。该URL验证请求的签名,并向kube-apiserver返回有关用户的信息,例如用户的帐户、Arn和UserId。

要手动生成身份验证令牌,请在终端窗口中键入以下命令:
```bash
aws eks get-token --cluster-name <cluster_name>
```

您也可以以编程方式获取令牌。下面是用Go编写的示例：
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

# 简化中文翻译

## 简介

这是一个简单的 Markdown 文件,包含了一些文本和图像。我们将把这些内容翻译成简体中文。

请注意以下要求:

- 如果一行包含 `![任何文本]`,则表示图像,不需要翻译。
- 我会以如下格式回复: "Your translation is:\n[翻译文本]"
- 不会改变/修复 Markdown 符号。
- 只返回翻译后的文本。
- 保持正式语气,使用行业术语。
- 不会添加额外信息,如标题等,只需翻译文本。
- 如果文本只有图像 Markdown 标签,则只翻译 `[]` 符号内的文本,不添加其他标题。
- 不会翻译 `` 符号内的文本。

## 正文

这是一个简单的 Markdown 文件,包含了一些文本和图像。

我们将把这些内容翻译成简体中文。

![一个简单的图像](image.jpg)

`这是一些代码`

这是一些普通的文本。
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

每个令牌都以 `k8s-aws-v1.` 开头,后跟一个 base64 编码的字符串。解码后的字符串应该类似于以下内容:
```bash
https://sts.amazonaws.com/?Action=GetCallerIdentity&Version=2011-06-15&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=XXXXJPFRILKNSRC2W5QA%2F20200219%2Fus-xxxx-1%2Fsts%2Faws4_request&X-Amz-Date=20200219T155427Z&X-Amz-Expires=60&X-Amz-SignedHeaders=host%3Bx-k8s-aws-id&X-Amz-Signature=XXXf8f3285e320ddb5e683a5c9a405301ad76546f24f28111fdad09cf648a393
```


您的翻译是:

令牌由包含 Amazon 凭证和签名的预签名 URL 组成。有关更多详细信息,请参见 [https://docs.aws.amazon.com/STS/latest/APIReference/API_GetCallerIdentity.html](https://docs.aws.amazon.com/STS/latest/APIReference/API_GetCallerIdentity.html)。

令牌的生存时间 (TTL) 为 15 分钟,之后需要生成一个新的令牌。当您使用像 `kubectl` 这样的客户端时,这是自动处理的,但是如果您使用 Kubernetes 仪表板,则需要在令牌过期时生成一个新的令牌并重新进行身份验证。

一旦用户的身份通过 AWS IAM 服务进行了身份验证,kube-apiserver 就会读取 `kube-system` 命名空间中的 `aws-auth` ConfigMap,以确定要与用户关联的 RBAC 组。`aws-auth` ConfigMap 用于在 IAM 主体(即 IAM 用户和角色)和 Kubernetes RBAC 组之间创建静态映射。RBAC 组可以在 Kubernetes RoleBindings 或 ClusterRoleBindings 中引用。它们类似于 IAM 角色,因为它们定义了可以对一组 Kubernetes 资源(对象)执行的一组操作(动词)。

### 集群访问管理器

集群访问管理器现在是管理 AWS IAM 主体对 Amazon EKS 集群访问的首选方式,它是 AWS API 的一项功能,是 EKS v1.23 及更高版本集群(新的或现有的)的一项选择性功能。它简化了 AWS IAM 和 Kubernetes RBAC 之间的身份映射,消除了在 AWS 和 Kubernetes API 之间切换或编辑 `aws-auth` ConfigMap 进行访问管理的需要,从而减少了操作开销,并有助于解决配置错误。该工具还使集群管理员能够自动撤销或细化授予用于创建集群的 AWS IAM 主体的 `cluster-admin` 权限。

此 API 依赖于两个概念:

- **访问条目:** 直接链接到允许对 Amazon EKS 集群进行身份验证的 AWS IAM 主体(用户或角色)的集群身份。
- **访问策略:** 是 Amazon EKS 特定的策略,为访问条目提供在 Amazon EKS 集群中执行操作的授权。

> 在发布时,Amazon EKS 仅支持预定义的 AWS 托管策略。访问策略不是 IAM 实体,而是由 Amazon EKS 定义和管理的。

集群访问管理器允许将上游 RBAC 与访问策略相结合,支持允许和传递(但不拒绝)Kubernetes AuthZ 决策,涉及 API 服务器请求。当上游 RBAC 和 Amazon EKS 授权器都无法确定请求评估的结果时,会发生拒绝决策。

通过此功能,Amazon EKS 支持三种身份验证模式:

1. `CONFIG_MAP` 继续独家使用 `aws-auth` configMap。
2. `API_AND_CONFIG_MAP` 从 EKS 访问入口 API 和 `aws-auth` configMap 中获取经过身份验证的 IAM 主体,优先使用访问入口。这是将现有 `aws-auth` 权限迁移到访问入口的理想方法。
3. `API` 专门依赖于 EKS 访问入口 API。这是新的**推荐方法**。

要开始使用,集群管理员可以创建或更新 Amazon EKS 集群,将首选身份验证设置为 `API_AND_CONFIG_MAP` 或 `API` 方法,并定义访问入口以授予所需的 AWS IAM 主体访问权限。
```bash
$ aws eks create-cluster \
    --name <CLUSTER_NAME> \
    --role-arn <CLUSTER_ROLE_ARN> \
    --resources-vpc-config subnetIds=<value>,endpointPublicAccess=true,endpointPrivateAccess=true \
    --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}' \
    --access-config authenticationMode=API_AND_CONFIG_MAP,bootstrapClusterCreatorAdminPermissions=false
```

以上命令是一个示例,用于创建一个Amazon EKS集群,而无需集群创建者的管理权限。

可以更新Amazon EKS集群配置以使用`update-cluster-config`命令启用`API`身份验证模式。要在使用`CONFIG_MAP`的现有集群上执行此操作,您首先需要更新为`API_AND_CONFIG_MAP`,然后再更新为`API`。**这些操作不可逆转**,这意味着无法从`API`切换到`API_AND_CONFIG_MAP`或`CONFIG_MAP`,也无法从`API_AND_CONFIG_MAP`切换到`CONFIG_MAP`。
```bash
$ aws eks update-cluster-config \
    --name <CLUSTER_NAME> \
    --access-config authenticationMode=API
```

API支持添加和撤销对集群的访问权限,以及验证指定集群的现有访问策略和访问条目。默认策略是根据Kubernetes RBAC创建的,如下所示。

| EKS访问策略 | Kubernetes RBAC |
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

> 在没有集群创建者管理员权限的情况下创建集群时,不会有任何访问条目可用,默认只创建一个条目。

### `aws-auth` ConfigMap _(已弃用)_

Kubernetes与AWS身份验证集成的一种方式是通过位于`kube-system`命名空间的`aws-auth` ConfigMap。它负责将AWS IAM身份(用户、组和角色)的身份验证映射到Kubernetes基于角色的访问控制(RBAC)授权。`aws-auth` ConfigMap在您的Amazon EKS集群配置过程中自动创建。它最初是为了允许节点加入您的集群而创建的,但如您所述,您也可以使用此ConfigMap为IAM主体添加RBAC访问权限。

要检查您集群的`aws-auth` ConfigMap,可以使用以下命令。
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


这个 ConfigMap 的主要部分位于 `data` 中的 `mapRoles` 块下,基本由 3 个参数组成。

- **groups:** 将 IAM 角色映射到的 Kubernetes 组。这可以是默认组,也可以是在 `clusterrolebinding` 或 `rolebinding` 中指定的自定义组。在上面的示例中,我们只有系统组被声明。
- **rolearn:** 要映射到 Kubernetes 组的 AWS IAM 角色的 ARN,使用以下格式: `arn:<PARTITION>:iam::<AWS_ACCOUNT_ID>:role/role-name`。
- **username:** 在 Kubernetes 中映射到 AWS IAM 角色的用户名。这可以是任何自定义名称。

> 也可以为 AWS IAM 用户映射权限,在 `aws-auth` ConfigMap 的 `data` 下定义一个新的 `mapUsers` 配置块,将 **rolearn** 参数替换为 **userarn**。但作为**最佳实践**,始终建议使用 `mapRoles`。

要管理权限,您可以编辑 `aws-auth` ConfigMap,添加或删除对您的 Amazon EKS 集群的访问权限。虽然可以手动编辑 `aws-auth` ConfigMap,但建议使用 `eksctl` 等工具,因为这是一个非常敏感的配置,不准确的配置可能会将您锁定在 Amazon EKS 集群之外。有关更多详细信息,请查看下面的[使用工具对 aws-auth ConfigMap 进行更改](https://aws.github.io/aws-eks-best-practices/security/docs/iam/#use-tools-to-make-changes-to-the-aws-auth-configmap)小节。

## 集群访问建议

### 使集群端点私有化

默认情况下,在配置 EKS 集群时,API 集群端点被设置为公共的,即可从互联网访问。尽管可从互联网访问,但该端点仍被视为安全,因为它要求所有 API 请求都经过 IAM 身份验证,然后由 Kubernetes RBAC 授权。但是,如果您的公司安全政策要求您限制从互联网访问 API,或者阻止您在集群 VPC 外路由流量,您可以:

- 将 EKS 集群端点配置为私有。有关此主题的更多信息,请参见[修改集群端点访问](https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html)。
- 保留公共集群端点,并指定哪些 CIDR 块可以与集群端点通信。这些块实际上是一个允许访问集群端点的公共 IP 地址的白名单。
- 配置公共访问,设置一组允许的 CIDR 块,并启用私有端点访问。这将允许从特定范围的公共 IP 进行公共访问,同时强制 kubelets(工作节点)与 Kubernetes API 之间的所有网络流量通过在配置控制平面时配置的跨账户 ENI。

### 不要使用服务帐户令牌进行身份验证
服务帐户令牌是一种长期的静态凭证。如果它被泄露、丢失或被盗,攻击者可能能够执行与该令牌相关的所有操作,直到删除该服务帐户。有时,您可能需要为必须从集群外部使用Kubernetes API的应用程序授予例外,例如CI/CD管道应用程序。如果这些应用程序在AWS基础设施(如EC2实例)上运行,请考虑使用实例配置文件并将其映射到Kubernetes RBAC角色。

### 对AWS资源实施最小权限访问

IAM用户无需被分配对AWS资源的权限即可访问Kubernetes API。如果您需要授予IAM用户访问EKS集群的权限,请在`aws-auth` ConfigMap中为该用户创建一个条目,将其映射到特定的Kubernetes RBAC组。

### 从集群创建者主体中删除cluster-admin权限

默认情况下,Amazon EKS集群是使用永久的`cluster-admin`权限绑定到集群创建者主体创建的。使用Cluster Access Manager API,可以在使用`API_AND_CONFIG_MAP`或`API`身份验证模式时,通过将`--access-config bootstrapClusterCreatorAdminPermissions`设置为`false`来创建没有此权限的集群。撤销此访问权限被认为是最佳实践,以避免对集群配置进行任何不必要的更改。撤销此访问权限的过程与撤销对集群的任何其他访问权限的过程相同。

该API为您提供了灵活性,只需取消IAM主体与访问策略(在本例中为`AmazonEKSClusterAdminPolicy`)的关联即可。
```bash
$ aws eks list-associated-access-policies \
    --cluster-name <CLUSTER_NAME> \
    --principal-arn <IAM_PRINCIPAL_ARN>

$ aws eks disassociate-access-policy --cluster-name <CLUSTER_NAME> \
    --principal-arn <IAM_PRINCIPAL_ARN. \
    --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy
```

完全删除与 `cluster-admin` 权限关联的访问条目。
```bash
$ aws eks list-access-entries --cluster-name <CLUSTER_NAME>

{
    "accessEntries": []
}

$ aws eks delete-access-entry --cluster-name <CLUSTER_NAME> \
  --principal-arn <IAM_PRINCIPAL_ARN>
```

> 如果需要在事故、紧急情况或破玻璃场景中访问集群,可以再次授予此访问权限,否则集群将无法访问。

如果集群仍配置有 `CONFIG_MAP` 身份验证方法,则应通过 `aws-auth` ConfigMap 授予所有其他用户对集群的访问权限,并在配置 `aws-auth` ConfigMap 后,可以删除分配给创建集群的实体的角色,仅在发生事故、紧急情况或破玻璃场景,或 `aws-auth` ConfigMap 损坏且集群无法访问时重新创建。这在生产集群中特别有用。

### 当多个用户需要对集群具有相同的访问权限时,请使用 IAM 角色

不要为每个单独的 IAM 用户创建一个条目,而是允许这些用户承担一个 IAM 角色,并将该角色映射到一个 Kubernetes RBAC 组。这将更容易维护,特别是当需要访问的用户数量增加时。

!!! attention
    使用 `aws-auth` ConfigMap 映射的 IAM 实体访问 EKS 集群时,将在 Kubernetes 审核日志的用户字段中记录描述的用户名。如果您使用 IAM 角色,实际承担该角色的用户将不会被记录,也无法进行审核。

如果仍在使用 `aws-auth` configMap 作为身份验证方法,在为 IAM 角色分配 K8s RBAC 权限时,您应该在用户名中包含 {{SessionName}}。这样,审核日志就会记录会话名称,以便您可以跟踪实际用户承担此角色的情况,以及 CloudTrail 日志。
```yaml
- rolearn: arn:aws:iam::XXXXXXXXXXXX:role/testRole
  username: testRole:{{SessionName}}
  groups:
    - system:masters
```


> 在 Kubernetes 1.20 及更高版本中,不再需要此更改,因为 `user.extra.sessionName.0` 已添加到 Kubernetes 审核日志中。

### 在创建 RoleBindings 和 ClusterRoleBindings 时采用最小权限访问

与之前关于授予对 AWS 资源的访问权限的观点一样,RoleBindings 和 ClusterRoleBindings 应该只包含执行特定功能所需的一组权限。除非绝对必要,否则请避免在您的角色和集群角色中使用 `["*"]`。如果您不确定要分配哪些权限,可以考虑使用 [audit2rbac](https://github.com/liggitt/audit2rbac) 等工具,根据 Kubernetes 审核日志中观察到的 API 调用自动生成角色和绑定。

### 使用自动化流程创建集群

如前几个步骤所示,在创建 Amazon EKS 集群时,如果不使用 `API_AND_CONFIG_MAP` 或 `API` 身份验证模式,并且不选择将 `cluster-admin` 权限委托给集群创建者,则创建集群的 IAM 实体用户或角色(例如联合用户)会自动获得集群 RBAC 配置中的 `system:masters` 权限。即使根据[此处](Rremove-the-cluster-admin-permissions-from-the-cluster-creator-principal)所述的最佳实践删除此权限是一个好主意,但如果使用 `CONFIG_MAP` 身份验证方法并依赖 `aws-auth` ConfigMap,则无法撤销此访问权限。因此,最好使用专用 IAM 角色通过基础设施自动化管道创建集群,该角色没有被其他用户或实体假定的权限,并定期审核此角色的权限、策略以及谁有权触发管道。此外,此角色不应用于执行集群上的日常操作,而应仅用于通过 SCM 代码更改等方式触发的管道中的集群级别操作。

### 使用专用 IAM 角色创建集群

当您创建 Amazon EKS 集群时,创建集群的 IAM 实体用户或角色(例如联合用户)会自动获得集群 RBAC 配置中的 `system:masters` 权限。此访问权限无法删除,并且不通过 `aws-auth` ConfigMap 进行管理。因此,最好使用专用 IAM 角色创建集群,并定期审核谁可以假定此角色。此角色不应用于执行集群上的日常操作,而应通过 `aws-auth` ConfigMap 向其他用户授予对集群的访问权限。配置 `aws-auth` ConfigMap 后,应该保护该角色,并仅在临时提升权限/紧急情况下使用,例如在集群无法访问的情况下。这在没有配置直接用户访问的集群中特别有用。

### 定期审核对集群的访问
谁需要访问权限很可能会随时间而变化。计划定期审核 `aws-auth` ConfigMap,以查看谁被授予了访问权限以及分配给他们的权限。您还可以使用开源工具,如 [kubectl-who-can](https://github.com/aquasecurity/kubectl-who-can) 或 [rbac-lookup](https://github.com/FairwindsOps/rbac-lookup) 来检查绑定到特定服务帐户、用户或组的角色。我们将在探讨[审核](detective.md)部分时进一步探讨这个主题。您可以在这篇[文章](https://www.nccgroup.trust/us/about-us/newsroom-and-events/blog/2019/august/tools-and-methods-for-auditing-kubernetes-rbac-policies/?mkt_tok=eyJpIjoiWWpGa056SXlNV1E0WWpRNSIsInQiOiJBT1hyUTRHYkg1TGxBV0hTZnRibDAyRUZ0VzBxbndnRzNGbTAxZzI0WmFHckJJbWlKdE5WWDdUQlBrYVZpMnNuTFJ1R3hacVYrRCsxYWQ2RTRcL2pMN1BtRVA1ZFZcL0NtaEtIUDdZV3pENzNLcE1zWGVwUndEXC9Pb2tmSERcL1pUaGUifQ%3D%3D)中找到更多想法。

### 如果依赖于 `aws-auth` configMap,请使用工具进行更改

格式不正确的 aws-auth ConfigMap 可能会导致您失去对集群的访问权限。如果需要对 ConfigMap 进行更改,请使用工具。

**eksctl**
`eksctl` CLI 包含一个用于向 aws-auth ConfigMap 添加身份映射的命令。

查看 CLI 帮助:
```bash
$ eksctl create iamidentitymapping --help
...
```

检查映射到您的 Amazon EKS 集群的身份。
```bash
$ eksctl get iamidentitymapping --cluster $CLUSTER_NAME --region $AWS_REGION
ARN                                                                   USERNAME                        GROUPS                                                  ACCOUNT
arn:aws:iam::788355785855:role/kube-system-<SELF_GENERATED_UUID>      system:node:{{SessionName}}     system:bootstrappers,system:nodes,system:node-proxier  
```

创建IAM角色作为集群管理员:

![any text](https://d2908q01vomqb2.cloudfront.net/b6692ea5df920cad691c20319a6fffd7a4a766b/2017/03/03/bigdata-cluster-admin.png)

要将IAM角色授予集群管理员权限,请执行以下步骤:

1. 创建一个IAM角色,并为其分配所需的权限。
2. 将该IAM角色附加到您的Amazon EKS集群。
3. 使用`kubectl`命令将该IAM角色授予集群管理员权限。

以下是具体步骤:

`aws eks update-kubeconfig --name my-cluster`

这将更新您的Kubernetes配置文件,以便您可以使用`kubectl`与您的Amazon EKS集群进行交互。

接下来,使用以下命令将IAM角色授予集群管理员权限:

`kubectl create clusterrolebinding my-cluster-admin-binding --clusterrole=cluster-admin --user=arn:aws:iam::123456789012:role/my-cluster-admin-role`

请将`my-cluster-admin-binding`和`my-cluster-admin-role`替换为您自己的值。
```bash
$ eksctl create iamidentitymapping --cluster  <CLUSTER_NAME> --region=<region> --arn arn:aws:iam::123456:role/testing --group system:masters --username admin
...
```

有关更多信息,请查看 [`eksctl` 文档](https://eksctl.io/usage/iam-identity-mappings/)

**[aws-auth](https://github.com/keikoproj/aws-auth) by keikoproj**

`aws-auth` by keikoproj 包括 CLI 和 Go 库。

下载并查看 CLI 帮助:
```bash
$ go get github.com/keikoproj/aws-auth
...
$ aws-auth help
...
```

另外，使用 [krew 插件管理器](https://krew.sigs.k8s.io) 为 kubectl 安装 `aws-auth`。
```bash
$ kubectl krew install aws-auth
...
$ kubectl aws-auth
...
```

[查看 GitHub 上的 aws-auth 文档](https://github.com/keikoproj/aws-auth/blob/master/README.md)以获取更多信息,包括 Go 库。

**[AWS IAM 认证器 CLI](https://github.com/kubernetes-sigs/aws-iam-authenticator/tree/master/cmd/aws-iam-authenticator)**

`aws-iam-authenticator` 项目包括一个用于更新 ConfigMap 的 CLI。

[在 GitHub 上下载发行版](https://github.com/kubernetes-sigs/aws-iam-authenticator/releases)。

将集群权限添加到 IAM 角色:
```bash
$ ./aws-iam-authenticator add role --rolearn arn:aws:iam::185309785115:role/lil-dev-role-cluster --username lil-dev-user --groups system:masters --kubeconfig ~/.kube/config
...
```


### 身份验证和访问管理的替代方法

虽然 IAM 是验证需要访问 EKS 集群的用户的首选方式,但也可以使用 OIDC 身份提供程序(如 GitHub)通过身份验证代理和 Kubernetes [模拟](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#user-impersonation)来进行身份验证。已在 AWS 开源博客上发布了两种此类解决方案的帖子:

- [使用 Teleport 通过 GitHub 凭据对 EKS 进行身份验证](https://aws.amazon.com/blogs/opensource/authenticating-eks-github-credentials-teleport/)
- [使用 kube-oidc-proxy 在多个 EKS 集群之间保持一致的 OIDC 身份验证](https://aws.amazon.com/blogs/opensource/consistent-oidc-authentication-across-multiple-eks-clusters-using-kube-oidc-proxy/)

!!! attention
    EKS 本地支持 OIDC 身份验证,无需使用代理。有关更多信息,请阅读发布博客[为 Amazon EKS 引入 OIDC 身份提供程序身份验证](https://aws.amazon.com/blogs/containers/introducing-oidc-identity-provider-authentication-amazon-eks/)。有关如何使用 Dex(一个流行的开源 OIDC 提供程序,具有各种不同身份验证方法的连接器)配置 EKS 的示例,请参见[使用 Dex & dex-k8s-authenticator 对 Amazon EKS 进行身份验证](https://aws.amazon.com/blogs/containers/using-dex-dex-k8s-authenticator-to-authenticate-to-amazon-eks/)。如博客所述,OIDC 提供程序验证的用户名/组将出现在 Kubernetes 审核日志中。

您还可以使用 [AWS SSO](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html) 将 AWS 与外部身份提供程序(如 Azure AD)联合。如果您决定使用此方法,AWS CLI v2.0 包括一个选项,可以创建一个命名配置文件,使您可以轻松地将 SSO 会话与当前 CLI 会话关联并承担 IAM 角色。请注意,您必须在运行 `kubectl` 之前先承担一个角色,因为 IAM 角色用于确定用户的 Kubernetes RBAC 组。

## EKS 容器的身份和凭据

某些在 Kubernetes 集群中运行的应用程序需要权限来调用 Kubernetes API,才能正常运行。例如, [AWS Load Balancer Controller](https://github.com/kubernetes-sigs/aws-load-balancer-controller) 需要能够列出服务的端点。该控制器还需要能够调用 AWS API 来配置和配置 ALB。在本节中,我们将探讨为 Pod 分配权限和特权的最佳实践。

### Kubernetes 服务帐户
服务帐户是一种特殊类型的对象,允许您将Kubernetes RBAC角色分配给pod。 每个集群中的每个命名空间都会自动创建一个默认服务帐户。 当您将pod部署到命名空间而不引用特定的服务帐户时,该命名空间的默认服务帐户将自动分配给该pod,并且该服务帐户(JWT)令牌将作为卷挂载到pod的`/var/run/secrets/kubernetes.io/serviceaccount`目录中。 解码该目录中的服务帐户令牌将显示以下元数据:
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

默认服务帐户具有以下对Kubernetes API的权限。
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


这个角色授权未经身份验证和经过身份验证的用户读取 API 信息,并被认为可以公开访问。

当一个运行在 Pod 中的应用程序调用 Kubernetes API 时,该 Pod 需要被分配一个服务帐户,该服务帐户明确授予它调用这些 API 的权限。与用户访问指南类似,绑定到服务帐户的角色或集群角色应仅限于应用程序需要运行的 API 资源和方法,而不应包括其他内容。要使用非默认服务帐户,只需将 Pod 的 `spec.serviceAccountName` 字段设置为您要使用的服务帐户的名称。有关创建服务帐户的更多信息,请参见 [https://kubernetes.io/docs/reference/access-authn-authz/rbac/#service-account-permissions](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#service-account-permissions)。

!!! note
    在 Kubernetes 1.24 之前,Kubernetes 会自动为每个服务帐户创建一个密钥。这个密钥被挂载到 pod 的 /var/run/secrets/kubernetes.io/serviceaccount 目录下,pod 会使用它来向 Kubernetes API 服务器进行身份验证。在 Kubernetes 1.24 中,服务帐户令牌在 pod 运行时动态生成,默认情况下只有一小时的有效期。不会再创建服务帐户的密钥。如果您有一个运行在集群外部的应用程序需要向 Kubernetes API 进行身份验证,例如 Jenkins,您需要创建一个类型为 `kubernetes.io/service-account-token` 的密钥,并添加一个引用服务帐户的注解,如 `metadata.annotations.kubernetes.io/service-account.name: <SERVICE_ACCOUNT_NAME>`。以这种方式创建的密钥不会过期。

### 面向服务帐户的 IAM 角色 (IRSA)

IRSA 是一个允许您将 IAM 角色分配给 Kubernetes 服务帐户的功能。它利用了 Kubernetes 的一个功能,称为[服务帐户令牌卷投射](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#serviceaccount-token-volume-projection)。当 Pod 配置了引用 IAM 角色的服务帐户时,Kubernetes API 服务器将在启动时调用集群的公共 OIDC 发现端点。该端点以加密签名的方式签署由 Kubernetes 颁发的 OIDC 令牌,并将其挂载为一个卷。这个签名令牌允许 Pod 调用与该 IAM 角色关联的 AWS API。当调用 AWS API 时,AWS SDK 会调用 `sts:AssumeRoleWithWebIdentity`。在验证令牌签名后,IAM 会交换 Kubernetes 颁发的令牌以获取临时的 AWS 角色凭证。

在使用 IRSA 时,重要的是[重复使用 AWS SDK 会话](#reuse-aws-sdk-sessions-with-irsa)以避免不必要的 AWS STS 调用。

解码 IRSA 的 (JWT) 令牌将产生类似于下面示例的输出:
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

这个特定的令牌通过假定一个IAM角色来授予Pod对S3的只读权限。当应用程序尝试从S3读取时,该令牌将被交换为一组临时的IAM凭据,类似于以下内容:
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

一个作为 EKS 控制平面一部分运行的变异 Webhook 将 AWS 角色 ARN 和 Web 身份令牌文件的路径注入到 Pod 作为环境变量。这些值也可以手动提供。
```bash
AWS_ROLE_ARN=arn:aws:iam::AWS_ACCOUNT_ID:role/IAM_ROLE_NAME
AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token
```

[简体中文翻译]

kubelet 将在令牌的总 TTL 的 80% 以上或 24 小时后自动轮换投射令牌。AWS SDK 负责在令牌轮换时重新加载令牌。有关 IRSA 的更多信息,请参见 [https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts-technical-overview.html](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts-technical-overview.html)。

### EKS Pod 身份

[EKS Pod 身份](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)是一项在 re:Invent 2023 推出的功能,允许您将 IAM 角色分配给 Kubernetes 服务帐户,而无需为您的 AWS 帐户中的每个集群配置 Open Id Connect (OIDC) 身份提供程序(IDP)。要使用 EKS Pod 身份,您必须部署一个代理,该代理作为 DaemonSet pod 在每个合格的工作节点上运行。此代理作为 EKS 附加组件提供给您,是使用 EKS Pod 身份功能的先决条件。您的应用程序必须使用[受支持的 AWS SDK 版本](https://docs.aws.amazon.com/eks/latest/userguide/pod-id-minimum-sdk.html)才能使用此功能。

当为 Pod 配置 EKS Pod 身份时,EKS 将在 `/var/run/secrets/pods.eks.amazonaws.com/serviceaccount/eks-pod-identity-token` 挂载和刷新 pod 身份令牌。AWS SDK 将使用此令牌与 EKS Pod 身份代理进行通信,该代理使用 pod 身份令牌和代理的 IAM 角色通过调用 [AssumeRoleForPodIdentity API](https://docs.aws.amazon.com/eks/latest/APIReference/API_auth_AssumeRoleForPodIdentity.html) 为您的 pod 创建临时凭证。传递给您的 pod 的 pod 身份令牌是从您的 EKS 集群颁发的 JWT,并经过加密签名,具有适用于 EKS Pod 身份的 JWT 声明。

要了解有关 EKS Pod 身份的更多信息,请参见[此博客](https://aws.amazon.com/blogs/containers/amazon-eks-pod-identity-a-new-way-for-applications-on-eks-to-obtain-iam-credentials/)。

您不需要对应用程序代码进行任何修改即可使用 EKS Pod 身份。受支持的 AWS SDK 版本将通过使用[凭证提供程序链](https://docs.aws.amazon.com/sdkref/latest/guide/standardized-credentials.html)自动发现通过 EKS Pod 身份提供的凭证。与 IRSA 一样,EKS pod 身份在您的 pod 中设置变量,以指示它们如何查找 AWS 凭证。

#### 使用 IAM 角色进行 EKS Pod 身份

- EKS Pod 身份只能直接承担属于与 EKS 集群相同 AWS 帐户的 IAM 角色。要访问另一个 AWS 帐户中的 IAM 角色,您必须通过[在 SDK 配置中配置配置文件](https://docs.aws.amazon.com/sdkref/latest/guide/feature-assume-role-credentials.html)或[应用程序代码](https://docs.aws.amazon.com/IAM/latest/UserGuide/sts_example_sts_AssumeRole_section.html)中承担该角色。

- 在为 EKS Pod 身份配置服务帐户时，配置 Pod 身份关联的人员或流程必须对该角色具有 `iam:PassRole` 授权。
- 每个服务帐户只能通过 EKS Pod 身份与一个 IAM 角色关联，但您可以将同一 IAM 角色与多个服务帐户关联。
- 与 EKS Pod 身份一起使用的 IAM 角色必须允许 `pods.eks.amazonaws.com` 服务主体担任它们,_并_设置会话标签。以下是一个示例角色信任策略,它允许 EKS Pod 身份使用 IAM 角色:
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


AWS 建议使用条件键如 `aws:SourceOrgId` 来帮助防范跨服务混淆代理问题(cross-service confused deputy problem)。在上述示例角色信任策略中，`ResourceOrgId` 是一个变量,等于 AWS 组织 ID,即 AWS 帐户所属的 AWS 组织的 ID。EKS 在使用 EKS Pod 身份(EKS Pod Identities)承担角色时,会传入一个等于该值的 `aws:SourceOrgId`。

#### ABAC 和 EKS Pod 身份

当 EKS Pod 身份承担一个 IAM 角色时,它会设置以下会话标签:

|EKS Pod 身份会话标签|值|
|:--|:--|
|kubernetes-namespace|与 EKS Pod 身份关联的 pod 所在的命名空间。|
|kubernetes-service-account|与 EKS Pod 身份关联的 Kubernetes 服务帐户的名称。|
|eks-cluster-arn|EKS 集群的 ARN,例如 `arn:${Partition}:eks:${Region}:${Account}:cluster/${ClusterName}`。集群 ARN 是唯一的,但如果一个集群被删除并在同一区域、同一 AWS 帐户下使用相同名称重新创建,它将具有相同的 ARN。|
|eks-cluster-name|EKS 集群的名称。请注意,EKS 集群名称在您的 AWS 帐户内可能相同,而在其他 AWS 帐户中的 EKS 集群也可能相同。|
|kubernetes-pod-name|EKS 中 pod 的名称。|
|kubernetes-pod-uid|EKS 中 pod 的 UID。|

这些会话标签允许您使用基于属性的访问控制(ABAC)来仅向特定的 Kubernetes 服务帐户授予对 AWS 资源的访问权限。在这样做时,非常重要的是要了解 Kubernetes 服务帐户只在命名空间内唯一,而 Kubernetes 命名空间只在 EKS 集群内唯一。可以通过使用 `aws:PrincipalTag/<tag-key>` 全局条件键(如 `aws:PrincipalTag/eks-cluster-arn`)在 AWS 策略中访问这些会话标签。

例如,如果您想授予只有特定服务帐户才能访问您帐户中的 AWS 资源,您需要检查 `eks-cluster-arn`、`kubernetes-namespace` 和 `kubernetes-service-account` 标签,以确保只有来自预期集群的该服务帐户才能访问该资源,因为其他集群可能具有相同的 `kubernetes-service-account` 和 `kubernetes-namespaces`。

此示例 S3 存储桶策略仅在 `kubernetes-service-account`、`kubernetes-namespace`、`eks-cluster-arn` 都满足预期值时,才授予对该 S3 存储桶中对象的访问权限,其中 EKS 集群托管在 AWS 帐户 `111122223333` 中。
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

EKS Pod 身份和 IRSA 都是向 EKS Pod 提供临时 AWS 凭证的首选方式。除非您有特定的 IRSA 用例,否则我们建议您在使用 EKS 时使用 EKS Pod 身份。此表有助于比较这两个功能。

|# |EKS Pod 身份 | IRSA |
|:--|:--|:--|
|需要权限在您的 AWS 帐户中创建 OIDC IDP 吗?|否|是|
|每个集群需要唯一的 IDP 设置吗|否|是|
|为 ABAC 设置相关的会话标签|是|否|
|需要 iam:PassRole 检查吗?|是| 否 |
|使用您 AWS 帐户的 AWS STS 配额?|否|是|
|可以访问其他 AWS 帐户| 通过角色链接间接 | 通过 sts:AssumeRoleWithWebIdentity 直接 |
|与 AWS SDK 兼容 |是|是|
|需要在节点上部署 Pod 身份代理 Daemonset 吗? |是|否|

## EKS Pod 身份和凭证的建议

### 更新 aws-node daemonset 以使用 IRSA

目前,aws-node daemonset 被配置为使用分配给 EC2 实例的角色来为 Pod 分配 IP。此角色包括几个 AWS 托管策略,例如 AmazonEKS_CNI_Policy 和 EC2ContainerRegistryReadOnly,有效地允许在节点上运行的**所有** Pod 附加/分离 ENI、分配/取消分配 IP 地址或从 ECR 拉取镜像。由于这会给您的集群带来风险,因此建议您更新 aws-node daemonset 以使用 IRSA。可以在此指南的[存储库](https://github.com/aws/aws-eks-best-practices/tree/master/projects/enable-irsa/src)中找到执行此操作的脚本。

aws-node daemonset 目前不支持 EKS Pod 身份。

### 限制对分配给工作节点的实例配置文件的访问

当您使用 IRSA 或 EKS Pod 身份时,它会更新 Pod 的凭证链以先使用 IRSA 或 EKS Pod 身份,但 Pod **仍然可以继承分配给工作节点的实例配置文件的权限**。使用 IRSA 或 EKS Pod 身份时,**强烈**建议您阻止对[实例元数据](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html)的访问,以确保您的应用程序只拥有所需的权限,而不是其节点的权限。

!!! caution
    阻止对实例元数据的访问将阻止不使用 IRSA 或 EKS Pod 身份的 Pod 继承分配给工作节点的角色。

您可以通过要求实例仅使用 IMDSv2 并更新跳数到 1 来阻止对实例元数据的访问,如下例所示。您也可以将这些设置包含在节点组的启动模板中。**不要**禁用实例元数据,因为这将阻止节点终止处理程序和其他依赖于实例元数据的组件正常工作。
```bash
$ aws ec2 modify-instance-metadata-options --instance-id <value> --http-tokens required --http-put-response-hop-limit 1
...
```

如果您使用 Terraform 创建用于托管节点组的启动模板，请添加元数据块以配置跳数，如以下代码片段所示:
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

您也可以通过在节点上操纵 iptables 来阻止 pod 访问 EC2 元数据。有关此方法的更多信息,请参见[限制对实例元数据服务的访问](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html#instance-metadata-limiting-access)。

如果您有一个使用较旧版本 AWS SDK 的应用程序,该版本不支持 IRSA 或 EKS Pod 身份,您应该更新 SDK 版本。

### 为 IRSA 角色设置 IAM 角色信任策略的范围,包括服务帐户名称、命名空间和集群

信任策略可以设置为针对命名空间或命名空间内的特定服务帐户。使用 IRSA 时,最好将角色信任策略设置得尽可能明确,包括服务帐户名称。这将有效地防止同一命名空间内的其他 Pod 假定该角色。CLI `eksctl` 在您使用它创建服务帐户/IAM 角色时会自动执行此操作。有关更多信息,请参见 [https://eksctl.io/usage/iamserviceaccounts/](https://eksctl.io/usage/iamserviceaccounts/)。

在直接使用 IAM 时,这意味着在角色的信任策略中添加条件,以确保 `:sub` 声明是您所期望的命名空间和服务帐户。例如,在我们有一个 IRSA 令牌的情况下,其 sub 声明为 "system:serviceaccount:default:s3-read-only"。这是 `default` 命名空间,服务帐户是 `s3-read-only`。您可以使用以下条件来确保只有您集群中给定命名空间内的服务帐户才能假定该角色:
```json
  "Condition": {
      "StringEquals": {
          "oidc.eks.us-west-2.amazonaws.com/id/D43CF17C27A865933144EA99A26FB128:aud": "sts.amazonaws.com",
          "oidc.eks.us-west-2.amazonaws.com/id/D43CF17C27A865933144EA99A26FB128:sub": "system:serviceaccount:default:s3-read-only"
      }
  }
```


### 为每个应用程序使用一个 IAM 角色

使用 IRSA 和 EKS Pod Identity 时,最佳实践是为每个应用程序分配自己的 IAM 角色。这可以提高隔离性,因为您可以修改一个应用程序而不影响另一个应用程序,并允许您应用最小权限原则,仅授予应用程序所需的权限。

在使用 ABAC 和 EKS Pod Identity 时,您可以跨多个服务帐户使用一个共同的 IAM 角色,并依赖于它们的会话属性进行访问控制。当以大规模运营时,这特别有用,因为 ABAC 允许您使用更少的 IAM 角色进行操作。

### 当您的应用程序需要访问 IMDS 时,使用 IMDSv2 并增加 EC2 实例上的跳数限制到 2

[IMDSv2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html) 要求您使用 PUT 请求来获取会话令牌。初始 PUT 请求必须包含会话令牌的 TTL。较新版本的 AWS SDK 将自动处理这个问题和令牌的续订。另外,您还需要注意,EC2 实例上的默认跳数限制被故意设置为 1,以防止 IP 转发。因此,在 EC2 实例上运行的 Pod 请求会话令牌可能最终会超时,并回退到使用 IMDSv1 数据流。EKS 通过_启用_v1 和 v2,并将节点上的跳数限制更改为 2(由 eksctl 或官方 CloudFormation 模板提供的节点)来支持 IMDSv2。

### 禁用服务帐户令牌的自动挂载

如果您的应用程序不需要调用 Kubernetes API,请在应用程序的 PodSpec 中将 `automountServiceAccountToken` 属性设置为 `false`,或者修补每个命名空间中的默认服务帐户,使其不会自动挂载到 Pod 中。例如:
```bash
kubectl patch serviceaccount default -p $'automountServiceAccountToken: false'
```


### 为每个应用程序使用专用的服务帐户

每个应用程序都应该有自己专用的服务帐户。这适用于 Kubernetes API 的服务帐户以及 IRSA 和 EKS Pod Identity。

!!! attention
    如果您采用蓝/绿集群升级方法而不是执行就地集群升级时使用 IRSA,您将需要更新每个 IRSA IAM 角色的信任策略,以使用新集群的 OIDC 端点。蓝/绿集群升级是指您创建一个运行较新版本 Kubernetes 的集群,并将其与旧集群并行运行,然后使用负载均衡器或服务网格无缝地将流量从在旧集群上运行的服务转移到新集群。
    在使用 EKS Pod Identity 进行蓝/绿集群升级时,您将在新集群中创建 pod 身份关联,并在存在 `sourceArn` 条件时更新 IAM 角色信任策略。

### 以非 root 用户身份运行应用程序

容器默认以 root 用户身份运行。虽然这允许它们读取 web 身份令牌文件,但以 root 用户身份运行容器并不被视为最佳实践。作为替代方案,请考虑在 PodSpec 中添加 `spec.securityContext.runAsUser` 属性。`runAsUser` 的值是任意值。

在以下示例中,Pod 中的所有进程都将在 `runAsUser` 字段指定的用户 ID 下运行。
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

当您以非 root 用户运行容器时，它会阻止容器读取 IRSA 服务帐户令牌,因为该令牌默认被分配了 0600 [root] 权限。如果您将容器的 securityContext 更新为包含 fsgroup=65534 [Nobody],它将允许容器读取该令牌。
```yaml
spec:
  securityContext:
    fsGroup: 65534
```


在 Kubernetes 1.19 及更高版本中,不再需要此更改,应用程序可以在不将它们添加到 Nobody 组的情况下读取 IRSA 服务帐户令牌。

### 授予应用程序最小特权访问权限

[Action Hero](https://github.com/princespaghetti/actionhero) 是一个实用程序,您可以将其与应用程序一起运行,以识别应用程序需要正常运行的 AWS API 调用和相应的 IAM 权限。它类似于 [IAM Access Advisor](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_access-advisor.html),因为它可以帮助您逐步限制分配给应用程序的 IAM 角色的范围。有关授予 [最小特权访问](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#grant-least-privilege) AWS 资源的更多信息,请参阅文档。

考虑在与 IRSA 和 Pod 身份一起使用的 IAM 角色上设置 [权限边界](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html)。您可以使用权限边界来确保 IRSA 或 Pod 身份使用的角色不能超过最大权限级别。有关使用权限边界的入门示例指南和示例权限边界策略,请参见此 [GitHub 存储库](https://github.com/aws-samples/example-permissions-boundary)。

### 审查并撤销对您的 EKS 集群的不必要的匿名访问

理想情况下,应该禁用所有 API 操作的匿名访问。匿名访问是通过为 Kubernetes 内置用户系统:anonymous 创建 RoleBinding 或 ClusterRoleBinding 来授予的。您可以使用 [rbac-lookup](https://github.com/FairwindsOps/rbac-lookup) 工具来识别 system:anonymous 用户在您的集群上拥有的权限:
```bash
./rbac-lookup | grep -P 'system:(anonymous)|(unauthenticated)'
system:anonymous               cluster-wide        ClusterRole/system:discovery
system:unauthenticated         cluster-wide        ClusterRole/system:discovery
system:unauthenticated         cluster-wide        ClusterRole/system:public-info-viewer
```

除了 system:public-info-viewer 之外的任何角色或 ClusterRole 都不应绑定到 system:anonymous 用户或 system:unauthenticated 组。

在某些特定的 API 上启用匿名访问可能有合法的理由。如果这适用于您的集群,请确保只有这些特定的 API 可以被匿名用户访问,并且在没有身份验证的情况下公开这些 API 不会使您的集群容易受到攻击。

在 Kubernetes/EKS 版本 1.14 之前,system:unauthenticated 组默认与 system:discovery 和 system:basic-user ClusterRoles 相关联。请注意,即使您已将集群更新到 1.14 或更高版本,这些权限可能仍然在您的集群上启用,因为集群更新不会撤销这些权限。
要检查哪些 ClusterRoles 除了 system:public-info-viewer 之外有"system:unauthenticated",您可以运行以下命令(需要 jq 工具):
```bash
kubectl get ClusterRoleBinding -o json | jq -r '.items[] | select(.subjects[]?.name =="system:unauthenticated") | select(.metadata.name != "system:public-info-viewer") | .metadata.name'
```

以下命令可以从除"system:public-info-viewer"之外的所有角色中删除"system:unauthenticated":
```bash
kubectl get ClusterRoleBinding -o json | jq -r '.items[] | select(.subjects[]?.name =="system:unauthenticated") | select(.metadata.name != "system:public-info-viewer") | del(.subjects[] | select(.name =="system:unauthenticated"))' | kubectl apply -f -
```

另外,您可以通过 kubectl describe 和 kubectl edit 手动检查和删除。要检查 system:unauthenticated 组在您的集群上是否具有 system:discovery 权限,请运行以下命令:
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

要检查 system:unauthenticated 组在您的集群上是否具有 system:basic-user 权限，请运行以下命令:
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

如果系统:未经身份验证的组绑定到您集群上的系统:发现和/或系统:基本用户 ClusterRoles,您应该将这些角色与系统:未经身份验证的组分离。使用以下命令编辑系统:发现 ClusterRoleBinding:
```bash
kubectl edit clusterrolebindings system:discovery
```

以上命令将在编辑器中打开系统:discovery ClusterRoleBinding的当前定义,如下所示:
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

从上述编辑器屏幕的"subjects"部分删除system:unauthenticated组的条目。

对system:basic-user ClusterRoleBinding重复相同的步骤。

### 使用IRSA重复使用AWS SDK会话

当您使用IRSA时,使用AWS SDK编写的应用程序使用传递到您的pod的令牌来调用`sts:AssumeRoleWithWebIdentity`以生成临时AWS凭证。这与其他AWS计算服务不同,在其他AWS计算服务中,计算服务直接向AWS计算资源(如lambda函数)传递临时AWS凭证。这意味着每次初始化AWS SDK会话时,都会进行一次对AWS STS的`AssumeRoleWithWebIdentity`调用。如果您的应用程序快速扩展并初始化许多AWS SDK会话,您可能会遇到来自AWS STS的节流,因为您的代码将进行许多`AssumeRoleWithWebIdentity`调用。

为了避免这种情况,我们建议在应用程序内重复使用AWS SDK会话,以避免不必要的`AssumeRoleWithWebIdentity`调用。

在以下示例代码中,使用boto3 python SDK创建了一个会话,并使用该会话创建客户端并与Amazon S3和Amazon SQS进行交互。`AssumeRoleWithWebIdentity`只调用一次,AWS SDK将在凭证过期时自动刷新`my_session`的凭证。
```py hl_lines="4 7 8"  
import boto3

# Create your own session
my_session = boto3.session.Session()

# Now we can create low-level clients from our session
sqs = my_session.client('sqs')
s3 = my_session.client('s3')

s3response = s3.list_buckets()
sqsresponse = sqs.list_queues()


#print the response from the S3 and SQS APIs
print("s3 response:")
print(s3response)
print("---")
print("sqs response:")
print(sqsresponse)
```


如果您正在将应用程序从另一个AWS计算服务(如EC2)迁移到使用IRSA的EKS,这个细节尤其重要。在其他计算服务上,初始化AWS SDK会话不会调用AWS STS,除非您指示它这样做。

### 替代方法

虽然IRSA和EKS Pod Identities是将AWS身份分配给pod的首选方式,但它们要求您在应用程序中包含最新版本的AWS SDK。有关当前支持IRSA的SDK的完整列表,请参见[https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts-minimum-sdk.html](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts-minimum-sdk.html),对于EKS Pod Identities,请参见[https://docs.aws.amazon.com/eks/latest/userguide/pod-id-minimum-sdk.html](https://docs.aws.amazon.com/eks/latest/userguide/pod-id-minimum-sdk.html)。如果您有一个无法立即使用兼容SDK更新的应用程序,社区中有几种解决方案可用于将IAM角色分配给Kubernetes pod,包括[kube2iam](https://github.com/jtblin/kube2iam)和[kiam](https://github.com/uswitch/kiam)。尽管AWS不认可、不赞同也不支持使用这些解决方案,但它们被广泛用于社区中,以实现与IRSA和EKS Pod Identities类似的结果。

如果您需要使用这些非AWS提供的解决方案,请谨慎行事,并确保了解这样做的安全影响。

## 工具和资源

- [Amazon EKS Security Immersion Workshop - Identity and Access Management](https://catalog.workshops.aws/eks-security-immersionday/en-US/2-identity-and-access-management)
- [Terraform EKS Blueprints Pattern - Fully Private Amazon EKS Cluster](https://github.com/aws-ia/terraform-aws-eks-blueprints/tree/main/patterns/fully-private-cluster)
- [Terraform EKS Blueprints Pattern - IAM Identity Center Single Sign-On for Amazon EKS Cluster](https://github.com/aws-ia/terraform-aws-eks-blueprints/tree/main/patterns/sso-iam-identity-center)
- [Terraform EKS Blueprints Pattern - Okta Single Sign-On for Amazon EKS Cluster](https://github.com/aws-ia/terraform-aws-eks-blueprints/tree/main/patterns/sso-okta)
- [audit2rbac](https://github.com/liggitt/audit2rbac)
- [rbac.dev](https://github.com/mhausenblas/rbac.dev) 有关Kubernetes RBAC的其他资源,包括博客和工具
- [Action Hero](https://github.com/princespaghetti/actionhero)
- [kube2iam](https://github.com/jtblin/kube2iam)
- [kiam](https://github.com/uswitch/kiam)
