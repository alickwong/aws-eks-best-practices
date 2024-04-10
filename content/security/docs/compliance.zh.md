合规性

合规性是 AWS 和其服务消费者之间的共同责任。一般来说,AWS 负责"云端的安全性",而用户负责"云中的安全性"。AWS 和用户各自负责的界限会因服务而有所不同。例如,对于 Fargate,AWS 负责管理其数据中心的物理安全性、硬件、虚拟基础设施(Amazon EC2)和容器运行时(Docker)。Fargate 用户负责保护容器镜像和应用程序的安全性。了解谁负责什么是在运行必须遵守合规标准的工作负载时需要考虑的重要因素。

下表显示了不同容器服务符合的合规性计划。

| 合规性计划 | Amazon ECS 编排器 | Amazon EKS 编排器 | ECS Fargate | Amazon ECR |
| ------------------ |:----------:|:----------:|:-----------:|:----------:|
| PCI DSS Level 1 | 1 | 1 | 1 | 1 |
| HIPAA 合格 | 1 | 1 | 1 | 1 |
| SOC I | 1 | 1 | 1 | 1 |
| SOC II | 1 | 1 | 1 | 1 |
| SOC III | 1 | 1 | 1 | 1 |
| ISO 27001:2013 | 1 | 1 | 1 | 1 |
| ISO 9001:2015 | 1 | 1 | 1 | 1 |
| ISO 27017:2015 | 1 | 1 | 1 | 1 |
| ISO 27018:2019 | 1 | 1 | 1 | 1 |
| IRAP | 1 | 1 | 1 | 1 |
| FedRAMP Moderate (East/West) | 1 | 1 | 0 | 1 |
| FedRAMP High (GovCloud) | 1 | 1 | 0 | 1 |
| DOD CC SRG | 1 | DISA Review (IL5) | 0 | 1 |
| HIPAA BAA | 1 | 1 | 1 | 1 |
| MTCS | 1 | 1 | 0 | 1 |
| C5 | 1 | 1 | 0 | 1 |
| K-ISMS | 1 | 1 | 0 | 1 |
| ENS High | 1 | 1 | 0 | 1 |
| OSPAR | 1 | 1 | 0 | 1 |
| HITRUST CSF | 1 | 1 | 1 | 1 |

合规性状态随时间而变化。有关最新状态,请参阅 [https://aws.amazon.com/compliance/services-in-scope/](https://aws.amazon.com/compliance/services-in-scope/)。

有关云认证模型和最佳实践的更多信息,请参阅 AWS 白皮书[面向安全云采用的认证模型](https://d1.awsstatic.com/whitepapers/accreditation-models-for-secure-cloud-adoption.pdf)。

## 左移

左移的概念涉及在软件开发生命周期的早期捕获策略违规和错误。从安全的角度来看,这可能会非常有益。例如,开发人员可以在将应用程序部署到集群之前修复其配置中的问题。尽早发现这些错误将有助于防止违反您的策略的配置被部署。

### 代码即策略

策略可以被视为一组管理行为的规则,即允许的行为或禁止的行为。例如,您可能有一个策略,要求所有 Dockerfile 都包含一个 USER 指令,使容器以非 root 用户身份运行。作为一个文档,这样的策略可能很难发现和执行。随着您的需求发生变化,它也可能过时。使用代码即策略(PaC)解决方案,您可以自动化安全性、合规性和隐私控制,以检测、预防、减少和抵御已知和持续的威胁。此外,它们为您提供了一种机制,可以将您的策略编码并像管理其他代码工件一样管理它们。这种方法的好处是,您可以重复使用您的 DevOps 和 GitOps 策略来管理和一致地应用跨 Kubernetes 集群的策略。有关 PaC 选项和 PSP 未来的信息,请参阅 [Pod 安全](https://aws.github.io/aws-eks-best-practices/security/docs/pods/#pod-security)。

### 在管道中使用代码即策略工具来检测部署前的违规行为

- [OPA](https://www.openpolicyagent.org/) 是 CNCF 的一个开源策略引擎。它用于做出策略决策,可以以各种不同的方式运行,例如作为语言库或服务。OPA 策略是用一种称为 Rego 的特定领域语言(DSL)编写的。虽然它通常作为 [Gatekeeper](https://github.com/open-policy-agent/gatekeeper) 项目的一部分在 Kubernetes 动态准入控制器中运行,但 OPA 也可以集成到您的 CI/CD 管道中。这允许开发人员在发布周期的早期获得有关他们配置的反馈,从而有助于他们在部署到生产环境之前解决问题。可以在 GitHub [存储库](https://github.com/aws/aws-eks-best-practices/tree/master/policies/opa)中找到一些常见的 OPA 策略集合。
- [Conftest](https://github.com/open-policy-agent/conftest) 建立在 OPA 之上,为开发人员提供了一种测试 Kubernetes 配置的体验。
- [Kyverno](https://kyverno.io/) 是一个专为 Kubernetes 设计的策略引擎。使用 Kyverno,策略作为 Kubernetes 资源进行管理,无需学习新的语言来编写策略。这允许使用熟悉的工具,如 kubectl、git 和 kustomize 来管理策略。Kyverno 策略可以验证、变更和生成 Kubernetes 资源,并确保 OCI 镜像供应链安全性。[Kyverno CLI](https://kyverno.io/docs/kyverno-cli/) 可用于测试策略并验证作为 CI/CD 管道一部分的资源。所有 Kyverno 社区策略都可以在 [Kyverno 网站](https://kyverno.io/policies/)上找到,有关在管道中使用 Kyverno CLI 编写测试的示例,请参见 [policies 存储库](https://github.com/kyverno/policies)。

## 工具和资源

- [Amazon EKS 安全沉浸式研讨会 - 监管合规性](https://catalog.workshops.aws/eks-security-immersionday/en-US/10-regulatory-compliance)
- [kube-bench](https://github.com/aquasecurity/kube-bench)
- [docker-bench-security](https://github.com/docker/docker-bench-security)
- [AWS Inspector](https://aws.amazon.com/inspector/)
- [Kubernetes 安全审查](https://github.com/kubernetes/community/blob/master/sig-security/security-audit-2019/findings/Kubernetes%20Final%20Report.pdf) Kubernetes 1.13.4(2019)的第三方安全评估
- [SUSE 的 NeuVector](https://www.suse.com/neuvector/) 开源的零信任容器安全平台,提供合规性报告和自定义合规性检查