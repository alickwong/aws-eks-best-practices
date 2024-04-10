# 运行时安全

运行时安全为正在运行的容器提供主动保护。其目的是检测和/或阻止容器内部的恶意活动。这可以通过 Kubernetes 集成的 Linux 内核或内核扩展中的多种机制来实现,例如 Linux 功能、安全计算 (seccomp)、AppArmor 或 SELinux。还有一些选项,如 Amazon GuardDuty 和第三方工具,可以帮助建立基线并检测异常活动,而无需手动配置 Linux 内核机制。

!!! attention
    Kubernetes 目前不提供任何本地机制来加载 seccomp、AppArmor 或 SELinux 配置文件到节点上。它们必须手动加载或在节点引导时安装。这必须在 Pod 中引用它们之前完成,因为调度器不知道哪些节点有配置文件。请参见下文,了解 Security Profiles Operator 等工具如何帮助自动将配置文件部署到节点。

## 安全上下文和内置 Kubernetes 控制

许多 Linux 运行时安全机制都与 Kubernetes 紧密集成,可通过 Kubernetes [安全上下文](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)进行配置。一个选项是 `privileged` 标志,默认为 `false`,如果启用,它实质上等同于主机上的 root。在生产工作负载中启用特权模式几乎总是不合适的,但还有许多其他控制可以根据需要为容器提供更细粒度的权限。

### Linux 功能

Linux 功能允许您向 Pod 或容器授予某些功能,而无需提供 root 用户的所有功能。例如,`CAP_NET_ADMIN` 允许配置网络接口或防火墙,`CAP_SYS_TIME` 允许操作系统时钟。

### Seccomp

使用安全计算 (seccomp),您可以阻止容器化应用程序对底层主机操作系统内核进行某些系统调用。虽然 Linux 操作系统有几百个系统调用,但其中大部分对于运行容器并不必要。通过限制容器可以进行的系统调用,您可以有效地减少应用程序的攻击面。

Seccomp 通过拦截系统调用并仅允许通过白名单的系统调用来工作。Docker 有一个[默认](https://github.com/moby/moby/blob/master/profiles/seccomp/default.json) seccomp 配置文件,适用于大多数通用工作负载,其他容器运行时如 containerd 也提供类似的默认值。您可以通过在 Pod 规范的 `securityContext` 部分添加以下内容来配置容器或 Pod 使用容器运行时的默认 seccomp 配置文件:

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
```

从 1.22 版本开始(在 alpha 版本中,从 1.27 版本开始稳定),可以使用 [单个 kubelet 标志](https://kubernetes.io/docs/tutorials/security/seccomp/#enable-the-use-of-runtimedefault-as-the-default-seccomp-profile-for-all-workloads) `--seccomp-default` 在节点上对所有 Pod 使用 `RuntimeDefault`。然后,`securityContext` 中指定的配置文件仅用于其他配置文件。

您也可以为需要额外权限的情况创建自己的配置文件。这手动完成可能非常繁琐,但有工具如 [Inspektor Gadget](https://github.com/inspektor-gadget/inspektor-gadget)(也在[网络安全部分](../network/)中推荐用于生成网络策略)和 [Security Profiles Operator](https://github.com/inspektor-gadget/inspektor-gadget)支持使用 eBPF 或日志记录来记录基线权限要求作为 seccomp 配置文件。Security Profiles Operator 还允许自动将记录的配置文件部署到节点,供 Pod 和容器使用。

### AppArmor 和 SELinux

AppArmor 和 SELinux 被称为[强制访问控制或 MAC 系统](https://en.wikipedia.org/wiki/Mandatory_access_control)。它们在概念上与 seccomp 相似,但 API 和功能不同,允许对特定文件系统路径或网络端口进行访问控制。对这些工具的支持取决于 Linux 发行版,Debian/Ubuntu 支持 AppArmor,RHEL/CentOS/Bottlerocket/Amazon Linux 2023 支持 SELinux。另请参见[基础设施安全部分](../hosts/#run-selinux),了解有关 SELinux 的更多讨论。

AppArmor 和 SELinux 都与 Kubernetes 集成,但截至 Kubernetes 1.28,AppArmor 配置文件必须通过[注释](https://kubernetes.io/docs/tutorials/security/apparmor/#securing-a-pod)指定,而 SELinux 标签可以通过安全上下文上的 [SELinuxOptions](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/#selinuxoptions-v1-core)字段直接设置。

与 seccomp 配置文件一样,上述提到的 Security Profiles Operator 可以帮助将配置文件部署到集群中的节点。(未来,该项目还计划为 AppArmor 和 SELinux 生成配置文件,就像它为 seccomp 所做的那样。)

## 建议

### 使用 Amazon GuardDuty 进行运行时监控和检测 EKS 环境中的威胁

如果您目前没有解决方案持续监控 EKS 运行时并分析 EKS 审核日志,扫描恶意软件和其他可疑活动,Amazon 强烈建议使用 [Amazon GuardDuty](https://aws.amazon.com/guardduty/) 作为客户想要一种简单、快速、安全、可扩展和经济高效的一键式方式来保护其 AWS 环境。Amazon GuardDuty 是一种安全监控服务,它分析和处理基础数据源,如 AWS CloudTrail 管理事件、AWS CloudTrail 事件日志、VPC 流日志(来自 Amazon EC2 实例)、Kubernetes 审核日志和 DNS 日志。它还包括 EKS 运行时监控。它使用不断更新的威胁情报源(如恶意 IP 地址和域名列表)和机器学习来识别您的 AWS 环境中意外、可能未经授权和恶意的活动。这可能包括权限升级、使用暴露的凭据、与恶意 IP 地址或域名通信、Amazon EC2 实例和 EKS 容器工作负载上存在恶意软件,或发现可疑的 API 活动。GuardDuty 通过生成安全发现来告知您 AWS 环境的状态,您可以在 GuardDuty 控制台或通过 Amazon EventBridge 查看这些发现。GuardDuty 还支持将您的发现导出到 Amazon Simple Storage Service (S3) 存储桶,并与其他服务(如 AWS Security Hub 和 Detective)集成。

观看这个 AWS 在线技术讨论["使用 Amazon GuardDuty 增强对 Amazon EKS 的威胁检测 - AWS 在线技术讨论"](https://www.youtube.com/watch?v=oNHGRRroJuE),了解如何在几分钟内启用这些额外的 EKS 安全功能。

### 可选:使用第三方解决方案进行运行时监控

如果您不熟悉 Linux 安全性,创建和管理 seccomp 和 Apparmor 配置文件可能很困难。如果您没有时间成为专家,请考虑使用第三方商业解决方案。许多解决方案已经超越了 Apparmor 和 seccomp 等静态配置文件,开始使用机器学习来阻止或警告可疑活动。下面的[工具和资源](#tools-and-resources)部分列出了一些这样的解决方案。您可以在 [AWS Marketplace for Containers](https://aws.amazon.com/marketplace/features/containers) 上找到更多选项。

### 在编写 seccomp 策略之前,请考虑添加/删除 Linux 功能

功能涉及内核函数中的各种检查,这些函数可通过系统调用访问。如果检查失败,系统调用通常会返回错误。检查可以在特定系统调用的开头进行,也可以在可通过多个不同系统调用访问的内核区域(如写入特定特权文件)更深处进行。另一方面,seccomp 是一个系统调用过滤器,在系统调用运行之前应用。进程可以设置一个过滤器,允许它们撤销运行某些系统调用或特定参数的权限。

在使用 seccomp 之前,请考虑添加/删除 Linux 功能是否能满足您的需求。有关更多信息,请参见[为容器设置功能](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-capabilities-for-a-container)。

### 查看是否可以通过使用 Pod 安全策略 (PSP) 来实现您的目标

Pod 安全策略提供了许多不同的方式来改善您的安全态势,而无需引入过多的复杂性。在进入构建 seccomp 和 Apparmor 配置文件之前,请先探索 PSP 中可用的选项。

!!! warning
    从 Kubernetes 1.25 版本开始,PSP 已被删除,取而代之的是 [Pod 安全准入](https://kubernetes.io/docs/concepts/security/pod-security-admission/)控制器。第三方替代品包括 OPA/Gatekeeper 和 Kyverno。可以从 GitHub 上的 [Gatekeeper 库](https://github.com/open-policy-agent/gatekeeper-library/tree/master/library/pod-security-policy)仓库中拉取一组 Gatekeeper 约束和约束模板,用于实现常见的 PSP 策略。Kyverno 政策库中也有许多 PSP 的替代品,包括完整的 [Pod 安全标准](https://kubernetes.io/docs/concepts/security/pod-security-standards/)集合。

## 工具和资源

- [7 things you should know before you start](https://itnext.io/seccomp-in-kubernetes-part-i-7-things-you-should-know-before-you-even-start-97502ad6b6d6)
- [AppArmor Loader](https://github.com/kubernetes/kubernetes/tree/master/test/images/apparmor-loader)
- [Setting up nodes with profiles](https://kubernetes.io/docs/tutorials/clusters/apparmor/#setting-up-nodes-with-profiles)
- [Security Profiles Operator](https://github.com/kubernetes-sigs/security-profiles-operator) 是一个 Kubernetes 增强功能,旨在使用户更容易在 Kubernetes 集群中使用 SELinux、seccomp 和 AppArmor。它提供了从正在运行的工作负载生成配置文件以及将配置文件加载到 Kubernetes 节点供 Pod 使用的功能。
- [Inspektor Gadget](https://github.com/inspektor-gadget/inspektor-gadget) 允许检查、跟踪和分析 Kubernetes 上许多方面的运行时行为,包括协助生成 seccomp 配置文件。
- [Aqua](https://www.aquasec.com/products/aqua-cloud-native-security-platform/)
- [Qualys](https://www.qualys.com/apps/container-security/)
- [Stackrox](https://www.stackrox.com/use-cases/threat-detection/)
- [Sysdig Secure](https://sysdig.com/products/kubernetes-security/)
- [Prisma](https://docs.paloaltonetworks.com/cn-series)
- [NeuVector by SUSE](https://www.suse.com/neuvector/) 开源、零信任容器安全平台,提供进程配置文件规则和文件访问规则。