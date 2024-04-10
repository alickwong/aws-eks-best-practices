# 图像安全

您应该将容器镜像视为防御攻击的第一道防线。不安全、构建不当的镜像可能允许攻击者逃脱容器的限制并访问主机。一旦进入主机,攻击者就可以访问敏感信息或在集群或您的AWS帐户内横向移动。以下最佳实践将有助于降低此类风险。

## 建议

### 创建最小化镜像

首先从容器镜像中删除所有多余的二进制文件。如果您使用的是来自Dockerhub的不熟悉的镜像,请使用[Dive](https://github.com/wagoodman/dive)等应用程序检查镜像,它可以显示容器每一层的内容。删除所有具有SETUID和SETGID位的二进制文件,因为它们可用于提升权限,并考虑删除所有shell和诸如nc和curl之类的实用程序,它们可用于不正当目的。您可以使用以下命令找到具有SETUID和SETGID位的文件:

```bash
find / -perm /6000 -type f -exec ls -ld {} \;
```

要删除这些文件的特殊权限,请将以下指令添加到容器镜像中:

```docker
RUN find / -xdev -perm /6000 -type f -exec chmod a-s {} \; || true
```

通俗地说,这被称为"去毒"您的镜像。

### 使用多阶段构建

使用多阶段构建是创建最小化镜像的一种方式。通常,多阶段构建用于自动化持续集成周期的某些部分。例如,多阶段构建可用于对源代码进行lint或执行静态代码分析。这为开发人员提供了获得即时反馈的机会,而不是等待管道执行。从安全的角度来看,多阶段构建很有吸引力,因为它们允许您最小化推送到容器注册表的最终镜像的大小。没有构建工具和其他多余二进制文件的容器镜像通过减少镜像的攻击面来改善您的安全态势。有关多阶段构建的更多信息,请参见[Docker的多阶段构建文档](https://docs.docker.com/develop/develop-images/multistage-build/)。

### 为容器镜像创建软件物料清单(SBOM)

"软件物料清单"(SBOM)是构成您的容器镜像的软件构件的嵌套清单。
SBOM是软件安全和软件供应链风险管理的关键构建块。[生成、存储SBOM并在中央存储库中扫描SBOM以检测漏洞](https://anchore.com/sbom/)有助于解决以下问题:

- **可见性**:了解构成您的容器镜像的组件。存储在中央存储库中允许随时审核和扫描SBOM,即使在部署后也可以检测和响应新的漏洞,例如零日漏洞。
- **来源验证**:确保现有的关于构件来源和构建方式的假设是正确的,并且构件或其随附的元数据在构建或交付过程中没有被篡改。
- **可信度**:确保给定的构件及其内容可以被信任去执行它所声称的操作,即适合于某个目的。这涉及判断代码是否安全可执行,并就执行代码相关的风险做出明智的决策。通过创建经过认证的管道执行报告、经过认证的SBOM和经过认证的CVE扫描报告来确保可信度,以向镜像的消费者保证该镜像确实是通过安全的方式(管道)创建的,使用的组件也是安全的。
- **依赖关系信任验证**:递归检查构件依赖树的可信度和来源的可信度。SBOM的漂移可以帮助检测恶意活动,包括未经授权的、不受信任的依赖项、渗透尝试。

以下工具可用于生成SBOM:

- [Amazon Inspector](https://docs.aws.amazon.com/inspector)可用于[创建和导出SBOM](https://docs.aws.amazon.com/inspector/latest/user/sbom-export.html)。
- [Anchore的Syft](https://github.com/anchore/syft)也可用于SBOM生成。为了更快地进行漏洞扫描,可以将为容器镜像生成的SBOM用作输入进行扫描。然后[对SBOM和扫描报告进行认证并附加](https://github.com/sigstore/cosign/blob/main/doc/cosign_attach_attestation.md)到镜像,然后将镜像推送到中央OCI存储库,如Amazon ECR,以供审查和审核。

通过查看[CNCF软件供应链最佳实践指南](https://project.linuxfoundation.org/hubfs/CNCF_SSCP_v1.pdf)了解更多关于保护您的软件供应链的信息。

### 定期扫描镜像以检测漏洞

与虚拟机对应物一样,容器镜像也可能包含带有漏洞的二进制文件或应用程序库,或者随时间而产生漏洞。防范利用的最佳方法是定期使用镜像扫描器扫描您的镜像。存储在Amazon ECR中的镜像可以在推送时或按需(每24小时一次)进行扫描。ECR目前支持[两种类型的扫描 - 基本和增强](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning.html)。基本扫描利用开源的[Clair](https://github.com/quay/clair)镜像扫描解决方案,无需付费。[增强扫描](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning-enhanced.html)使用Amazon Inspector提供自动持续扫描,[需要额外付费](https://aws.amazon.com/inspector/pricing/)。扫描镜像后,结果将记录到EventBridge中的ECR事件流。您也可以在ECR控制台中查看扫描结果。具有HIGH或CRITICAL漏洞的镜像应该被删除或重建。如果已部署的镜像出现漏洞,应尽快替换。

知道已部署了哪些带有漏洞的镜像对于保持环境安全至关重要。虽然您可以自行构建镜像跟踪解决方案,但已经有几种商业产品提供这些以及其他高级功能,包括:

- [Grype](https://github.com/anchore/grype)
- [Palo Alto - Prisma Cloud (twistcli)](https://docs.paloaltonetworks.com/prisma/prisma-cloud/prisma-cloud-admin-compute/tools/twistcli_scan_images)
- [Aqua](https://www.aquasec.com/)
- [Kubei](https://github.com/Portshift/kubei)
- [Trivy](https://github.com/aquasecurity/trivy)
- [Snyk](https://support.snyk.io/hc/en-us/articles/360003946917-Test-images-with-the-Snyk-Container-CLI)

还可以使用Kubernetes验证 webhook 来验证镜像是否没有严重漏洞。验证 webhook 在Kubernetes API之前被调用。它们通常用于拒绝不符合webhook中定义的验证标准的请求。[这](https://aws.amazon.com/blogs/containers/building-serverless-admission-webhooks-for-kubernetes-with-aws-sam/)是一个调用ECR describeImageScanFindings API来确定是否有关键漏洞的无服务器 webhook 的示例。如果发现漏洞,则拒绝该pod,并将CVE列表作为事件返回。

### 使用证明来验证构件完整性

证明是一个经过加密签名的"声明",声称某些事情 - 一个"谓词"(例如管道运行或SBOM或漏洞扫描报告)是关于另一件事 - "主体"(即容器镜像)的真实情况。

证明有助于用户验证构件来自软件供应链中的可信来源。例如,我们可能使用一个容器镜像而不知道该镜像包含的所有软件组件或依赖项。但是,如果我们相信容器镜像的生产者对该镜像中存在的软件做出的声明,我们就可以使用生产者的证明来依赖该构件。这意味着我们可以继续安全地在我们的工作流中使用该构件,而不必自己进行分析。

- 可以使用[AWS Signer](https://docs.aws.amazon.com/signer/latest/developerguide/Welcome.html)或[Sigstore cosign](https://github.com/sigstore/cosign/blob/main/doc/cosign_attest.md)创建证明。
- [Kyverno](https://kyverno.io/)等Kubernetes准入控制器可用于[验证证明](https://kyverno.io/docs/writing-policies/verify-images/sigstore/)。
- 参考这个[研讨会](https://catalog.us-east-1.prod.workshops.aws/workshops/49343bb7-2cc5-4001-9d3b-f6a33b3c4442/en-US/0-introduction)了解更多关于在AWS上使用开源工具进行软件供应链管理最佳实践的信息,包括创建和附加证明到容器镜像。

### 为ECR存储库创建IAM策略

如今,一个组织中通常会有多个独立运营的开发团队共享一个AWS帐户。如果这些团队不需要共享资产,您可能希望创建一组IAM策略来限制每个团队可以与之交互的存储库。实现这一目标的一个好方法是使用ECR [命名空间](https://docs.aws.amazon.com/AmazonECR/latest/userguide/Repositories.html#repository-concepts)。命名空间是将相似的存储库分组在一起的一种方式。例如,团队A的所有注册表可以以team-a/为前缀,而团队B的注册表可以使用team-b/前缀。限制访问的策略可能如下所示:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowPushPull",
      "Effect": "Allow",
      "Action": [
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": [
        "arn:aws:ecr:<region>:<account_id>:repository/team-a/*"
      ]
    }
  ]
}
```

### 考虑使用ECR私有端点

ECR API有一个公共端点。因此,只要请求已通过IAM进行身份验证和授权,ECR注册表就可以从Internet访问。对于需要在没有Internet网关(IGW)的VPC中运行的沙盒环境,您可以配置ECR的私有端点。创建私有端点可让您通过私有IP地址而不是通过Internet路由流量来私下访问ECR API。有关此主题的更多信息,请参见[Amazon ECR接口VPC端点](https://docs.aws.amazon.com/AmazonECR/latest/userguide/vpc-endpoints.html)。

### 为ECR实施端点策略

ECR的默认端点策略允许访问某个区域内的所有ECR存储库。这可能允许攻击者/内部人员通过将数据打包为容器镜像并将其推送到另一个AWS帐户中的注册表来窃取数据。缓解此风险的方法是创建一个端点策略,该策略限制对ECR存储库的API访问。例如,以下策略允许您帐户中的所有AWS主体对您自己的ECR存储库执行所有操作:

```json
{
  "Statement": [
    {
      "Sid": "LimitECRAccess",
      "Principal": "*",
      "Action": "*",
      "Effect": "Allow",
      "Resource": "arn:aws:ecr:<region>:<account_id>:repository/*"
    }
  ]
}
```

您可以通过设置一个条件,使用新的`PrincipalOrgID`属性来进一步增强它,这将阻止不属于您的AWS组织的IAM主体推送/拉取镜像。有关更多详细信息,请参见[aws:PrincipalOrgID](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html#condition-keys-principalorgid)。
我们建议将相同的策略应用于`com.amazonaws.<region>.ecr.dkr`和`com.amazonaws.<region>.ecr.api`端点。
由于EKS从ECR拉取kube-proxy、coredns和aws-node的镜像,您需要将注册表的帐户ID(例如`602401143452.dkr.ecr.us-west-2.amazonaws.com/*`)添加到端点策略的资源列表中,或者修改策略以允许从"*"拉取,并限制推送到您的帐户ID。下表显示了EKS镜像从哪些AWS帐户提供的映射到集群区域。

|帐户号 |区域 |
|--- |--- |
|602401143452 |除下面列出的区域外的所有商业区域 |
|--- |--- |
|800184023465 |ap-east-1 - 亚太地区(香港) |
|558608220178 |me-south-1 - 中东(巴林) |
|918309763551 |cn-north-1 - 中国(北京) |
|961992271922 |cn-northwest-1 - 中国(宁夏) |

有关使用端点策略的更多信息,请参见[使用VPC端点策略控制Amazon ECR访问](https://aws.amazon.com/blogs/containers/using-vpc-endpoint-policies-to-control-amazon-ecr-access/)。

### 为ECR实施生命周期策略

[NIST应用容器安全指南](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-190.pdf)警告"注册表中的陈旧镜像"的风险,指出随着时间的推移,含有易受攻击、过时软件包的旧镜像应该被删除,以防止意外部署和暴露。
每个ECR存储库都可以有一个生命周期策略,该策略设置镜像过期的规则。[AWS官方文档](https://docs.aws.amazon.com/AmazonECR/latest/userguide/LifecyclePolicies.html)描述了如何设置测试规则、评估它们,然后应用它们。官方文档中有几个[生命周期策略示例](https://docs.aws.amazon.com/AmazonECR/latest/userguide/lifecycle_policy_examples.html),展示了在存储库中过滤镜像的不同方式:

- 按镜像年龄或数量过滤
- 按已标记或未标记的镜像过滤
- 按镜像标签过滤,可以在多个规则中或单个规则中进行

???+ warning
    如果长期运行应用程序的镜像从ECR中删除,当应用程序重新部署或水平扩展时,可能会导致拉取镜像错误。使用镜像生命周期策略时,请确保您有良好的CI/CD实践来保持部署和它们引用的镜像最新,并始终创建考虑您执行发布/部署频率的[镜像]过期规则。

### 创建一组经过审核的镜像

与允许开发人员创建自己的镜像相比,不如考虑为组织中不同的应用程序堆栈创建一组经过审核的镜像。通过这样做,开发人员可以免去学习如何编写Dockerfiles的麻烦,专注于编写代码。随着更改合并到主分支,CI/CD管道可以自动编译资产,将其存储在工件存储库中,并在将其推送到像ECR这样的Docker注册表之前将工件复制到适当的镜像中。至少,您应该创建一组基础镜像,供开发人员创建自己的Dockerfiles。理想情况下,您希望避免从Dockerhub拉取镜像,因为1/您并不总是知道镜像中包含什么,2/约[五分之一](https://www.kennasecurity.com/blog/one-fifth-of-the-most-used-docker-containers-have-at-least-one-critical-vulnerability/)的前1000个镜像都有漏洞。这些镜像及其漏洞的列表可以在[这里](https://vulnerablecontainers.org/)找到。

### 在Dockerfiles中添加USER指令以以非root用户运行

正如在pod安全部分提到的,您应该避免以root身份运行容器。虽然您可以将此配置为podSpec的一部分,但在Dockerfiles中使用`USER`指令也是一个好习惯。`USER`指令设置在`RUN`、`ENTRYPOINT`或`CMD`指令出现之后使用的UID。

### Lint您的Dockerfiles

Linting可用于验证您的Dockerfiles是否遵守一组预定义的准则,例如包含`USER`指令或要求所有镜像都要标记。[dockerfile_lint](https://github.com/projectatomic/dockerfile_lint)是RedHat的一个开源项目,它可以验证常见的最佳实践,并包括一个规则引擎,您可以使用它来构建自己的Dockerfile linting规则。它可以集成到CI管道中,构建违反规则的Dockerfiles将自动失败。

### 从Scratch构建镜像

减少容器镜像的攻击面应该是构建镜像时的首要目标。实现这一目标的理想方式是创建没有可用于利用漏洞的二进制文件的最小化镜像。幸运的是,Docker有一种机制可以从[`scratch`](https://docs.docker.com/develop/develop-images/baseimages/#create-a-simple-parent-image-using-scratch)创建镜像。对于像Go这样的语言,您可以创建一个静态链接的二进制文件,并在Dockerfile中引用它,如下例所示:

```docker
############################
# STEP 1 build executable binary
############################
FROM golang:alpine AS builder# Install git.
# Git is required for fetching the dependencies.
RUN apk update && apk add --no-cache gitWORKDIR $GOPATH/src/mypackage/myapp/COPY . . # Fetch dependencies.
# Using go get.
RUN go get -d -v# Build the binary.
RUN go build -o /go/bin/hello

############################
# STEP 2 build a small image
############################
FROM scratch# Copy our static executable.
COPY --from=builder /go/bin/hello /go/bin/hello# Run the hello binary.
ENTRYPOINT ["/go/bin/hello"]
```

这创建了一个仅包含您的应用程序的容器镜像,使其极其安全。

### 在ECR中使用不可变标签

[不可变标签](https://aws.amazon.com/about-aws/whats-new/2019/07/amazon-ecr-now-supports-immutable-image-tags/)强制您在每次将镜像推送到镜像存储库时更新镜像标签。这可以阻止攻击者在不更改镜像标签的情况下用恶意版本覆盖镜像。此外,它为您提供了一种简单有效地唯一标识镜像的方法。

### 对您的镜像、SBOM、管道运行和漏洞报告进行签名

当Docker刚刚推出时,没有用于验证容器镜像的加密模型。在v2中,Docker添加了摘要到镜像清单。这允许对镜像配置进行哈希处理,并使用哈希来生成镜像的ID。启用镜像签名后,Docker引擎会验证清单的签名,确保内容是由可信来源生成的,并且没有发生篡改。在下载每一层后,引擎都会验证该层的摘要,确保内容与清单中指定的内容匹配。镜像签名实际上允许您创建一个安全的供应链,通过验证与镜像相关的数字签名。

我们可以使用[AWS Signer](https://docs.aws.amazon.com/signer/latest/developerguide/Welcome.html)或[Sigstore Cosign](https://github.com/sigstore/cosign)来签署容器镜像,为SBOM、漏洞扫描报告和管道运行报告创建证明。这些证明保证了镜像的可信度和完整性,即它确实是由受信任的管道创建的,没有任何干扰或篡改,并且只包含文档中记录的(在SBOM中)、镜像发布者信任的软件组件。这些证明可以附加到容器镜像并推送到存储库。

在下一节中,我们将看到如何使用经过认证的工件进行审核和准入控制器验证。

### 使用Kubernetes准入控制器验证镜像完整性

我们可以以自动化的方式在将镜像部署到目标Kubernetes集群之前,使用[动态准入控制器](https://kubernetes.io/blog/2019/03/21/a-guide-to-kubernetes-admission-controllers/)验证镜像签名和经过认证的工件,并仅在工件的安全元数据符合准入控制器策略时才允许部署。

例如,我们可以编写一个策略,以加密方式验证镜像的签名、经过认证的SBOM、经过认证的管道运行报告或经过认证的CVE扫描报告。我们可以在策略中编写条件来检查报告中的数据,例如CVE扫描不应该有任何严重的CVE。只有满足这些条件的镜像部署才会被准入控制器允许,所有其他部署都会被拒绝。

准入控制器的示例包括:

- [Kyverno](https://kyverno.io/)
- [OPA Gatekeeper](https://github.com/open-policy-agent/gatekeeper)
- [Portieris](https://github.com/IBM/portieris)
- [Ratify](https://github.com/deislabs/ratify)
- [Kritis](https://github.com/grafeas/kritis)
- [Grafeas教程](https://github.com/kelseyhightower/grafeas-tutorial)
- [Voucher](https://github.com/Shopify/voucher)

### 更新容器镜像中的软件包

您应该在Dockerfiles中包含`RUN apt-get update && apt-get upgrade`来升级镜像中的软件包。尽管升级需要以root身份运行,但这发生在镜像构建阶段。应用程序不需要以root身份运行。您可以安装更新,然后切换到另一个用户,使用USER指令。如果您的基础镜像以非root用户身份运行,请切换到root并切换回来;不要完全依赖基础镜像的维护者来安装最新的安全更新。

运行`apt-get clean`以删除`/var/cache/apt/archives/`中的安装程序文件。您还可以在安装软件包后运行`rm -rf /var/lib/apt/lists/*`。这会删除索引文件或可安装软件包的列表。请注意,这些命令可能因每个包管理器而有所不同。例如:

```docker
RUN apt-get update && apt-get install -y \
    curl \
    git \
    libsqlite3-dev \
    && apt-get clean && rm -rf /var/lib/apt/lists/*
```

## 工具和资源

- [Amazon EKS Security Immersion Workshop - Image Security](https://catalog.workshops.aws/eks-security-immersionday/en-US/12-image-security)
- [docker-slim](https://github.com/docker-slim/docker-slim) 构建安全的最小化镜像
- [dockle](https://github.com/goodwithtech/dockle) 验证您的Dockerfile是否符合创建安全镜像的最佳实践
- [dockerfile-lint](https://github.com/projectatomic/dockerfile_lint) 用于Dockerfiles的基于规则的linter
- [hadolint](https://github.com/hadolint/hadolint) 一个智能的Dockerfile linter
- [Gatekeeper和OPA](https://github.com/open-policy-agent/gatekeeper) 基于策略的准入控制器
- [Kyverno](https://kyverno.io/) 一个原生Kubernetes的策略引擎
- [in-toto](https://in-toto.io/) 允许用户验证供应链中的步骤是否按预期执行,以及步骤是否由正确的参与者执行
- [Notary](https://github.com/theupdateframework/notary) 一个用于签署容器镜像的项目
- [Notary v2](https://github.com/notaryproject/nv2)
- [Grafeas](https://grafeas.io/) 一个开放的工件元数据API,用于审核和管理您的软件供应链
- [NeuVector by SUSE](https://www.suse.com/neuvector/) 开源的零信任容器安全平台,提供容器、镜像和注册表扫描以检测漏洞、机密和合规性。