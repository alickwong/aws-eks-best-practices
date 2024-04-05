!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

合规性

合规性是 AWS 和其服务消费者之间的共同责任。一般来说,AWS 负责"云端的安全性",而用户负责"云中的安全性"。AWS 和用户各自负责的范围会因服务而有所不同。例如,对于 Fargate,AWS 负责管理数据中心的物理安全性、硬件、虚拟基础设施(Amazon EC2)和容器运行时(Docker)。Fargate 用户负责保护容器镜像和应用程序的安全性。了解谁负责什么是在必须遵守合规标准的情况下运行工作负载时需要考虑的重要因素。

下表显示了不同容器服务符合的合规性计划。

| 合规性计划 | Amazon ECS 编排器 | Amazon EKS 编排器 | ECS Fargate | Amazon ECR |
| ------------------ |:----------:|:----------:|:-----------:|:----------:|
| PCI DSS 1 级 | 1 | 1 | 1 | 1 |
| HIPAA 合格 | 1 | 1 | 1 | 1 |
| SOC I | 1 | 1 | 1 | 1 |
| SOC II | 1 | 1 | 1 | 1 |
| SOC III | 1 | 1 | 1 | 1 |
| ISO 27001:2013 | 1 | 1 | 1 | 1 |
| ISO 9001:2015 | 1 | 1 | 1 | 1 |
| ISO 27017:2015 | 1 | 1 | 1 | 1 |
| ISO 27018:2019 | 1 | 1 | 1 | 1 |
| IRAP | 1 | 1 | 1 | 1 |
| FedRAMP 中等(东/西) | 1 | 1 | 0 | 1 |
| FedRAMP 高(GovCloud) | 1 | 1 | 0 | 1 |
| DOD CC SRG | 1 | DISA 审查(IL5) | 0 | 1 |
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

左移的概念涉及在软件开发生命周期的早期阶段捕捉策略违规和错误。从安全的角度来看,这可能非常有益。例如,开发人员可以在应用程序部署到集群之前修复其配置中的问题。尽早发现这些错误将有助于防止违反您的策略的配置被部署。

### 代码即策略

政策可以被视为管理行为的一组规则,即允许的行为或禁止的行为。例如,您可能有一项政策,规定所有Dockerfiles都应包含一个USER指令,使容器以非root用户身份运行。作为一个文件,这样的政策可能很难发现和执行。随着您的需求发生变化,它也可能过时。使用"代码即政策"(PaC)解决方案,您可以自动化安全性、合规性和隐私控制,以检测、预防、减少和抵御已知和持续的威胁。此外,它们为您提供了一种机制,可以将您的政策编码并像管理其他代码工件一样进行管理。这种方法的好处是,您可以重复使用您的DevOps和GitOps策略,在Kubernetes集群群中一致地管理和应用政策。有关PaC选项和PSP未来的信息,请参考[Pod安全性](https://aws.github.io/aws-eks-best-practices/security/docs/pods/#pod-security)。

### 在管道中使用代码即政策工具来检测部署前的违规行为

- [OPA](https://www.openpolicyagent.org/)是一个开源的政策引擎,属于CNCF的一部分。它用于做出政策决策,可以以各种不同的方式运行,例如作为语言库或服务。OPA政策是用一种称为Rego的特定领域语言(DSL)编写的。尽管它通常作为[Gatekeeper](https://github.com/open-policy-agent/gatekeeper)项目的一部分运行在Kubernetes动态准入控制器中,但OPA也可以并入您的CI/CD管道。这允许开发人员在发布周期的早期获得有关他们配置的反馈,从而有助于他们在进入生产环境之前解决问题。可以在GitHub [repository](https://github.com/aws/aws-eks-best-practices/tree/master/policies/opa)中找到一些常见的OPA政策集合。
- [Conftest](https://github.com/open-policy-agent/conftest)建立在OPA之上,为开发人员提供了一种测试Kubernetes配置的体验。
- [Kyverno](https://kyverno.io/)是一个专为Kubernetes设计的政策引擎。使用Kyverno,政策作为Kubernetes资源进行管理,编写政策不需要新的语言。这允许使用熟悉的工具,如kubectl、git和kustomize来管理政策。Kyverno政策可以验证、变更和生成Kubernetes资源,并确保OCI镜像供应链安全。[Kyverno CLI](https://kyverno.io/docs/kyverno-cli/)可用于测试政策并作为CI/CD管道的一部分验证资源。所有Kyverno社区政策都可以在[Kyverno网站](https://kyverno.io/policies/)上找到,有关在管道中使用Kyverno CLI编写测试的示例,请参见[policies repository](https://github.com/kyverno/policies)。

## 工具和资源
- [亚马逊 EKS 安全沉浸式研讨会 - 监管合规性](https://catalog.workshops.aws/eks-security-immersionday/en-US/10-regulatory-compliance)
- [kube-bench](https://github.com/aquasecurity/kube-bench)
- [docker-bench-security](https://github.com/docker/docker-bench-security)
- [AWS Inspector](https://aws.amazon.com/inspector/)
- [Kubernetes 安全审查](https://github.com/kubernetes/community/blob/master/sig-security/security-audit-2019/findings/Kubernetes%20Final%20Report.pdf) 对 Kubernetes 1.13.4 (2019) 的第三方安全评估
- [SUSE 的 NeuVector](https://www.suse.com/neuvector/) 开源的零信任容器安全平台,提供合规性报告和自定义合规性检查
