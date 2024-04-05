!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。
# 图像安全

您应该将容器镜像视为防御攻击的第一道防线。不安全、构建不当的镜像可能会让攻击者逃脱容器的限制并访问主机。一旦进入主机,攻击者就可以访问敏感信息或在集群或您的AWS帐户内横向移动。以下最佳实践将有助于降低此类事件发生的风险。

## 建议

### 创建最小化镜像

首先从容器镜像中删除所有多余的二进制文件。如果您使用的是来自Dockerhub的陌生镜像,请使用[Dive](https://github.com/wagoodman/dive)等应用程序检查镜像,它可以显示容器每一层的内容。删除所有具有SETUID和SETGID位的二进制文件,因为它们可用于提升权限,并考虑删除所有shell和诸如nc和curl之类的实用程序,因为它们可用于不正当目的。您可以使用以下命令找到具有SETUID和SETGID位的文件:
```bash
find / -perm /6000 -type f -exec ls -ld {} \;
```

要从这些文件中删除特殊权限,请在您的容器镜像中添加以下指令:
```docker
RUN find / -xdev -perm /6000 -type f -exec chmod a-s {} \; || true
```


通俗地说，这被称为"去毒"你的图像。

### 使用多阶段构建

使用多阶段构建是创建最小映像的一种方式。通常,多阶段构建用于自动化持续集成周期的某些部分。例如,多阶段构建可用于检查源代码或执行静态代码分析。这为开发人员提供了获得即时反馈的机会,而不必等待管道执行。从安全的角度来看,多阶段构建很有吸引力,因为它们允许您最小化推送到容器注册表的最终映像的大小。没有构建工具和其他多余二进制文件的容器映像通过减少映像的攻击面来改善您的安全状况。有关多阶段构建的更多信息,请参见[Docker的多阶段构建文档](https://docs.docker.com/develop/develop-images/multistage-build/)。

### 为您的容器映像创建软件材料清单(SBOM)

"软件材料清单"(SBOM)是构成您的容器映像的软件构件的嵌套清单。
SBOM是软件安全和软件供应链风险管理的关键构建块。[在中央存储库中生成和存储SBOM,并扫描SBOM以发现漏洞](https://anchore.com/sbom/)有助于解决以下问题:

- **可见性**:了解构成您的容器映像的组件。存储在中央存储库中允许随时审核和扫描SBOM,即使在部署后也可以检测和响应新的漏洞,例如零日漏洞。
- **来源验证**:确保现有的关于构件来源和创建方式的假设是正确的,并且构件或其随附的元数据在构建或交付过程中没有被篡改。
- **可信度**:确保给定的构件及其内容可以被信任去执行它声称要做的事情,即适合某个目的。这涉及到判断代码是否安全可执行,并对执行代码的风险做出明智的决策。通过创建经过认证的管道执行报告、经过认证的SBOM和经过认证的CVE扫描报告来确保可信度,以确保图像的使用者这个图像确实是通过安全的方式(管道)创建的,使用了安全的组件。
- **依赖关系信任验证**:递归检查构件依赖树的可信度和来源的可信度。SBOM的漂移可以帮助检测恶意活动,包括未经授权的、不受信任的依赖项,以及渗透尝试。

可以使用以下工具生成SBOM:

- [Amazon Inspector](https://docs.aws.amazon.com/inspector)可用于[创建和导出SBOM](https://docs.aws.amazon.com/inspector/latest/user/sbom-export.html)。

- [Syft from Anchore](https://github.com/anchore/syft)也可用于SBOM生成。为了更快的漏洞扫描,可以将为容器镜像生成的SBOM用作输入进行扫描。然后将SBOM和扫描报告[附加和证明](https://github.com/sigstore/cosign/blob/main/doc/cosign_attach_attestation.md)到镜像中,然后将镜像推送到像Amazon ECR这样的中央OCI存储库,以供审查和审核。

通过查看[CNCF软件供应链最佳实践指南](https://project.linuxfoundation.org/hubfs/CNCF_SSCP_v1.pdf)了解更多关于保护您的软件供应链的信息。

### 定期扫描镜像以发现漏洞

与虚拟机对应物一样,容器镜像也可能包含带有漏洞的二进制文件和应用程序库,或者随时间而产生漏洞。防范利用的最佳方法是定期使用镜像扫描器扫描您的镜像。存储在Amazon ECR中的镜像可以在推送时或按需(每24小时一次)进行扫描。ECR目前支持[两种类型的扫描 - 基本和增强](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning.html)。基本扫描利用开源镜像扫描解决方案[Clair](https://github.com/quay/clair),无需付费。[增强扫描](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning-enhanced.html)使用Amazon Inspector提供自动持续扫描,[需要额外付费](https://aws.amazon.com/inspector/pricing/)。镜像扫描后,结果将记录到EventBridge中的ECR事件流。您也可以在ECR控制台中查看扫描结果。应删除或重建存在高或严重漏洞的镜像。如果已部署的镜像出现漏洞,应尽快替换。

知道哪些镜像存在漏洞并部署在哪里对于保持环境安全至关重要。虽然您可以自行构建镜像跟踪解决方案,但已经有几种商业产品提供这些以及其他高级功能,包括:

- [Grype](https://github.com/anchore/grype)
- [Palo Alto - Prisma Cloud (twistcli)](https://docs.paloaltonetworks.com/prisma/prisma-cloud/prisma-cloud-admin-compute/tools/twistcli_scan_images)
- [Aqua](https://www.aquasec.com/)
- [Kubei](https://github.com/Portshift/kubei)
- [Trivy](https://github.com/aquasecurity/trivy)
- [Snyk](https://support.snyk.io/hc/en-us/articles/360003946917-Test-images-with-the-Snyk-Container-CLI)

Kubernetes 验证 webhook 也可用于验证映像是否不存在关键漏洞。验证 webhook 在 Kubernetes API 之前被调用。它们通常用于拒绝不符合 webhook 中定义的验证标准的请求。[这](https://aws.amazon.com/blogs/containers/building-serverless-admission-webhooks-for-kubernetes-with-aws-sam/)是一个无服务器 webhook 的示例,它调用 ECR describeImageScanFindings API 来确定是否有 pod 正在拉取存在关键漏洞的映像。如果发现漏洞,pod 将被拒绝,并且包含 CVE 列表的消息将作为事件返回。

### 使用证明来验证工件完整性

证明是一个经过加密签名的"声明",声称某些事情 - 一个"谓词"(例如管道运行或 SBOM 或漏洞扫描报告)是关于另一件事 - "主体"(即容器映像)的真实情况。

证明有助于用户验证工件来自软件供应链中的可信来源。例如,我们可能使用一个容器映像而不知道该映像中包含的所有软件组件或依赖项。但是,如果我们相信容器映像生产者对所存在的软件的说法,我们就可以使用生产者的证明来依赖该工件。这意味着我们可以安全地在工作流程中使用该工件,而不必自己进行分析。

- 可以使用 [AWS Signer](https://docs.aws.amazon.com/signer/latest/developerguide/Welcome.html) 或 [Sigstore cosign](https://github.com/sigstore/cosign/blob/main/doc/cosign_attest.md) 创建证明。
- [Kyverno](https://kyverno.io/) 等 Kubernetes 准入控制器可用于[验证证明](https://kyverno.io/docs/writing-policies/verify-images/sigstore/)。
- 请参考此[研讨会](https://catalog.us-east-1.prod.workshops.aws/workshops/49343bb7-2cc5-4001-9d3b-f6a33b3c4442/en-US/0-introduction)了解有关在 AWS 上使用开源工具进行软件供应链管理最佳实践的更多信息,包括创建和附加容器映像的证明。

### 为 ECR 存储库创建 IAM 策略

如今,一个组织中通常会有多个独立运营的开发团队共享一个 AWS 账户。如果这些团队不需要共享资产,您可能希望创建一组 IAM 策略来限制每个团队可以与之交互的存储库的访问权限。实现这一目标的一个好方法是使用 ECR [命名空间](https://docs.aws.amazon.com/AmazonECR/latest/userguide/Repositories.html#repository-concepts)。命名空间是一种将相似的存储库分组在一起的方式。例如,团队 A 的所有注册表都可以以 team-a/ 为前缀,而团队 B 的注册表可以使用 team-b/ 前缀。限制访问的策略可能如下所示:
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

考虑使用 ECR 私有端点

ECR API 有一个公共端点。因此,只要请求已通过 IAM 进行身份验证和授权,ECR 注册表就可以从互联网访问。对于需要在没有互联网网关 (IGW) 的集群 VPC 中运行的人来说,您可以为 ECR 配置一个私有端点。创建私有端点可让您通过私有 IP 地址私下访问 ECR API,而不是通过互联网路由流量。有关此主题的更多信息,请参见[Amazon ECR 接口 VPC 端点](https://docs.aws.amazon.com/AmazonECR/latest/userguide/vpc-endpoints.html)。

为 ECR 实施端点策略

默认端点策略允许访问某个区域内的所有 ECR 存储库。这可能允许攻击者/内部人员通过将数据打包为容器镜像并将其推送到另一个 AWS 帐户中的注册表来窃取数据。缓解此风险的方法是创建一个限制对 ECR 存储库的 API 访问的端点策略。例如,以下策略允许您帐户中的所有 AWS 主体对您自己的 ECR 存储库执行所有操作:
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


您可以通过设置使用新的 `PrincipalOrgID` 属性的条件来进一步增强这一点,这将防止 IAM 主体不属于您的 AWS 组织的情况下推送/拉取图像。有关更多详细信息,请参见 [aws:PrincipalOrgID](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html#condition-keys-principalorgid)。

我们建议将相同的策略应用于 `com.amazonaws.<region>.ecr.dkr` 和 `com.amazonaws.<region>.ecr.api` 端点。

由于 EKS 从 ECR 拉取 kube-proxy、coredns 和 aws-node 的镜像,您需要将注册表的帐户 ID 添加到端点策略中的资源列表中,例如 `602401143452.dkr.ecr.us-west-2.amazonaws.com/*`,或者修改策略以允许从"*"拉取并将推送限制为您的帐户 ID。下表显示了 EKS 镜像来源 AWS 帐户与集群区域之间的映射。

|帐户号码|区域|
|--- |--- |
|602401143452|除下列地区外的所有商业区域|
|--- |--- |
|800184023465|ap-east-1 - 亚太地区(香港)|
|558608220178|me-south-1 - 中东(巴林)|
|918309763551|cn-north-1 - 中国(北京)|
|961992271922|cn-northwest-1 - 中国(宁夏)|

有关使用端点策略的更多信息,请参见[使用 VPC 端点策略控制 Amazon ECR 访问](https://aws.amazon.com/blogs/containers/using-vpc-endpoint-policies-to-control-amazon-ecr-access/)。

### 为 ECR 实施生命周期策略

[NIST 应用程序容器安全指南](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-190.pdf)警告了"注册表中陈旧镜像"的风险,指出随着时间的推移,应该删除包含易受攻击、过时软件包的旧镜像,以防止意外部署和暴露。

每个 ECR 存储库都可以有一个生命周期策略,该策略设置图像过期的规则。[AWS 官方文档](https://docs.aws.amazon.com/AmazonECR/latest/userguide/LifecyclePolicies.html)描述了如何设置测试规则、评估它们,然后应用它们。官方文档中有几个[生命周期策略示例](https://docs.aws.amazon.com/AmazonECR/latest/userguide/lifecycle_policy_examples.html),展示了过滤存储库中图像的不同方式:

- 按图像年龄或数量过滤
- 按已标记或未标记的图像过滤
- 按图像标签过滤,可以是多个规则或单个规则

???+ warning
    如果长期运行应用程序的镜像从 ECR 中清除,当应用程序重新部署或水平扩展时,可能会导致拉取镜像错误。在使用镜像生命周期策略时,请确保您有良好的 CI/CD 实践来保持部署和它们引用的镜像最新,并始终创建考虑您执行发布/部署频率的[镜像]过期规则。

### 创建一组精选镜像

不要让开发人员创建自己的镜像,而是考虑为您组织中的不同应用程序堆栈创建一组经过审核的镜像。通过这样做,开发人员可以不必学习如何编写Dockerfiles,而是专注于编写代码。随着更改被合并到主分支,CI/CD管道可以自动编译资产,将其存储在制品存储库中,并在将其推送到像ECR这样的Docker注册表之前,将制品复制到适当的镜像中。至少您应该创建一组基础镜像,供开发人员创建自己的Dockerfiles。理想情况下,您希望避免从Docker Hub拉取镜像,因为1/您并不总是知道镜像中包含什么,2/前1000个最常用镜像中约有五分之一存在漏洞。这些镜像及其漏洞的列表可以在[这里](https://vulnerablecontainers.org/)找到。

### 在您的Dockerfiles中添加USER指令以以非root用户运行

正如在pod安全部分提到的,您应该避免以root用户运行容器。虽然您可以将此配置为podSpec的一部分,但使用`USER`指令将是一个好习惯。`USER`指令设置在执行出现在USER指令之后的`RUN`、`ENTRYPOINT`或`CMD`指令时使用的UID。

### 检查您的Dockerfiles

Linting可用于验证您的Dockerfiles是否遵守一组预定义的准则,例如包含`USER`指令或要求所有镜像都打上标签。[dockerfile_lint](https://github.com/projectatomic/dockerfile_lint)是RedHat的一个开源项目,它可以验证常见的最佳实践,并包含一个规则引擎,您可以使用它来构建自己的Dockerfile检查规则。它可以纳入CI管道中,这样违反规则的构建就会自动失败。

### 从头开始构建镜像

减少容器镜像的攻击面应该是构建镜像时的首要目标。实现这一目标的理想方式是创建最小化的镜像,不包含可用于利用漏洞的二进制文件。幸运的是,Docker有一种机制可以从[`scratch`](https://docs.docker.com/develop/develop-images/baseimages/#create-a-simple-parent-image-using-scratch)创建镜像。对于像Go这样的语言,您可以创建一个静态链接的二进制文件,并在Dockerfile中引用它,如下例所示:
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


这创建了一个仅包含您的应用程序而没有其他内容的容器镜像,使其极其安全。

### 在 ECR 中使用不可变标签

[不可变标签](https://aws.amazon.com/about-aws/whats-new/2019/07/amazon-ecr-now-supports-immutable-image-tags/)强制您在每次将镜像推送到镜像存储库时更新镜像标签。这可以阻止攻击者在不更改镜像标签的情况下用恶意版本覆盖镜像。此外,它为您提供了一种简单有效的方式来唯一标识镜像。

### 对您的镜像、SBOM、管道运行和漏洞报告进行签名

当 Docker 首次推出时,没有用于验证容器镜像的加密模型。在 v2 中,Docker 为镜像清单添加了摘要。这允许对镜像配置进行哈希处理,并使用该哈希值生成镜像的 ID。启用镜像签名后,Docker 引擎会验证清单的签名,确保内容是由可信来源生成的,且未发生篡改。在下载每一层后,引擎都会验证该层的摘要,确保内容与清单中指定的内容匹配。镜像签名实际上允许您创建一个安全的供应链,通过验证与镜像关联的数字签名。

我们可以使用 [AWS Signer](https://docs.aws.amazon.com/signer/latest/developerguide/Welcome.html) 或 [Sigstore Cosign](https://github.com/sigstore/cosign) 对容器镜像进行签名,为 SBOM、漏洞扫描报告和管道运行报告创建证明。这些证明可确保镜像的可信度和完整性,即它确实是由受信任的管道创建的,没有任何干扰或篡改,并且只包含文档中记录的软件组件(在 SBOM 中),这些组件已由镜像发布者验证和信任。这些证明可以附加到容器镜像并推送到存储库。

在下一节中,我们将了解如何使用经过证明的工件进行审核和准入控制器验证。

### 使用 Kubernetes 准入控制器进行镜像完整性验证

我们可以在将镜像部署到目标 Kubernetes 集群之前,通过[动态准入控制器](https://kubernetes.io/blog/2019/03/21/a-guide-to-kubernetes-admission-controllers/)以自动化方式验证镜像签名和经过证明的工件,并仅在安全元数据的工件符合准入控制器策略时才允许部署。

例如,我们可以编写一个策略,以加密方式验证镜像的签名、经过证明的 SBOM、经过证明的管道运行报告或经过证明的 CVE 扫描报告。我们可以在策略中编写条件,检查报告中的数据,例如 CVE 扫描不应该有任何严重的 CVE。只有满足这些条件的镜像才能被允许部署,其他所有部署都将被准入控制器拒绝。
[Kyverno]
[OPA Gatekeeper]
[Portieris]
[Ratify]
[Kritis]
[Grafeas tutorial]
[Voucher]

更新容器镜像中的软件包

您应该在Dockerfiles中包含RUN `apt-get update && apt-get upgrade`来升级镜像中的软件包。尽管升级需要以root身份运行,但这发生在镜像构建阶段。应用程序不需要以root身份运行。您可以安装更新,然后使用USER指令切换到其他用户。如果基础镜像以非root用户运行,请切换到root并切换回来;不要完全依赖基础镜像的维护者来安装最新的安全更新。

运行`apt-get clean`从`/var/cache/apt/archives/`中删除安装程序文件。您还可以在安装软件包后运行`rm -rf /var/lib/apt/lists/*`。这将删除索引文件或可安装软件包的列表。请注意,这些命令可能因每个包管理器而有所不同。例如:
```docker
RUN apt-get update && apt-get install -y \
    curl \
    git \
    libsqlite3-dev \
    && apt-get clean && rm -rf /var/lib/apt/lists/*
```

## 工具和资源

- [亚马逊 EKS 安全沉浸式研讨会 - 镜像安全](https://catalog.workshops.aws/eks-security-immersionday/en-US/12-image-security)
- [docker-slim](https://github.com/docker-slim/docker-slim) 构建安全的最小化镜像
- [dockle](https://github.com/goodwithtech/dockle) 验证您的 Dockerfile 是否符合创建安全镜像的最佳实践
- [dockerfile-lint](https://github.com/projectatomic/dockerfile_lint) 基于规则的 Dockerfile 检查器
- [hadolint](https://github.com/hadolint/hadolint) 一个智能的 Dockerfile 检查器
- [Gatekeeper 和 OPA](https://github.com/open-policy-agent/gatekeeper) 一个基于策略的准入控制器
- [Kyverno](https://kyverno.io/) 一个原生于 Kubernetes 的策略引擎
- [in-toto](https://in-toto.io/) 允许用户验证供应链中的步骤是否按预期执行,以及步骤是否由正确的参与者执行
- [Notary](https://github.com/theupdateframework/notary) 一个用于签署容器镜像的项目
- [Notary v2](https://github.com/notaryproject/nv2)
- [Grafeas](https://grafeas.io/) 一个开放的工件元数据 API,用于审核和管理您的软件供应链
- [SUSE 的 NeuVector](https://www.suse.com/neuvector/) 开源的零信任容器安全平台,提供容器、镜像和注册表扫描,以发现漏洞、密钥和合规性问题。
