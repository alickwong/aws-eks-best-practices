!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。
[运行时安全性]

运行时安全性为正在运行的容器提供主动保护。其目的是检测和/或防止容器内部的恶意活动。这可以通过Linux内核或与Kubernetes集成的内核扩展中的多种机制来实现,例如Linux功能、安全计算(seccomp)、AppArmor或SELinux。还有一些选项,如Amazon GuardDuty和第三方工具,可以帮助建立基线并检测异常活动,而无需过多手动配置Linux内核机制。

!!! attention
    Kubernetes目前没有提供任何本地机制来加载seccomp、AppArmor或SELinux配置文件到节点上。它们要么需要手动加载,要么在节点引导时安装。这必须在您的Pod中引用它们之前完成,因为调度器不知道哪些节点有配置文件。请参见下文,了解Security Profiles Operator等工具如何帮助自动将配置文件配置到节点上。

## 安全上下文和内置的Kubernetes控制

许多Linux运行时安全机制都与Kubernetes紧密集成,可以通过Kubernetes[安全上下文](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)进行配置。一个这样的选项是`privileged`标志,默认为`false`,如果启用,它实质上相当于主机上的root。在生产工作负载中启用特权模式几乎总是不合适的,但还有许多其他控制可以根据需要为容器提供更细粒度的权限。

### Linux功能

Linux功能允许您授予某些功能给Pod或容器,而无需提供root用户的所有功能。例如,`CAP_NET_ADMIN`允许配置网络接口或防火墙,`CAP_SYS_TIME`允许操作系统时钟。

### Seccomp

使用安全计算(seccomp),您可以防止容器化应用程序对底层主机操作系统内核进行某些系统调用。虽然Linux操作系统有几百个系统调用,但其中大部分对于运行容器并不必要。通过限制容器可以进行的系统调用,您可以有效地减少应用程序的攻击面。

Seccomp通过拦截系统调用并只允许通过白名单的系统调用来工作。Docker有一个[默认](https://github.com/moby/moby/blob/master/profiles/seccomp/default.json)的seccomp配置文件,适用于大多数通用工作负载,其他容器运行时如containerd也提供类似的默认配置。您可以通过在Pod规范的`securityContext`部分添加以下内容来配置容器或Pod使用容器运行时的默认seccomp配置文件:
```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
```


从 1.22 版本(目前处于 alpha 阶段,1.27 版本已稳定)开始,上述 `RuntimeDefault` 可以通过使用 [单个 kubelet 标志](https://kubernetes.io/docs/tutorials/security/seccomp/#enable-the-use-of-runtimedefault-as-the-default-seccomp-profile-for-all-workloads)，`--seccomp-default`，在节点上应用于所有 Pod。然后，在 `securityContext` 中指定的配置文件仅用于其他配置文件。

您也可以为需要额外权限的情况创建自己的配置文件。手动执行这一操作可能非常繁琐,但是有一些工具可以帮助您完成这项工作,例如 [Inspektor Gadget](https://github.com/inspektor-gadget/inspektor-gadget)(也在[网络安全部分](../network/)中推荐用于生成网络策略)和 [Security Profiles Operator](https://github.com/inspektor-gadget/inspektor-gadget),它们支持使用 eBPF 或日志来记录基线权限要求作为 seccomp 配置文件。Security Profiles Operator 还允许自动将记录的配置文件部署到节点,供 Pod 和容器使用。

### AppArmor 和 SELinux

AppArmor 和 SELinux 被称为[强制访问控制或 MAC 系统](https://en.wikipedia.org/wiki/Mandatory_access_control)。它们在概念上与 seccomp 相似,但具有不同的 API 和功能,允许对例如特定文件系统路径或网络端口进行访问控制。这些工具的支持情况因 Linux 发行版而异,Debian/Ubuntu 支持 AppArmor,而 RHEL/CentOS/Bottlerocket/Amazon Linux 2023 支持 SELinux。有关 SELinux 的更多讨论,请参见[基础设施安全部分](../hosts/#run-selinux)。

AppArmor 和 SELinux 都与 Kubernetes 集成,但在 Kubernetes 1.28 版本中,AppArmor 配置文件必须通过[注解](https://kubernetes.io/docs/tutorials/security/apparmor/#securing-a-pod)指定,而 SELinux 标签可以直接通过[安全上下文](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/#selinuxoptions-v1-core)字段设置。

与 seccomp 配置文件一样,上述提到的 Security Profiles Operator 可以帮助您将配置文件部署到集群中的节点。(未来,该项目还计划为 AppArmor 和 SELinux 生成配置文件,就像它为 seccomp 所做的那样。)

## 建议

### 使用 Amazon GuardDuty 对 EKS 环境进行运行时监控和威胁检测

如果您目前没有持续监控 EKS 运行时和分析 EKS 审核日志、扫描恶意软件和其他可疑活动的解决方案,Amazon 强烈建议使用 [Amazon GuardDuty](https://aws.amazon.com/guardduty/) 作为客户想要一种简单、快速、安全、可扩展和经济高效的一键式方式来保护其 AWS 环境。Amazon GuardDuty 是一项安全监控服务,它分析和处理基础数据源,如 AWS CloudTrail 管理事件、AWS CloudTrail 事件日志、VPC 流日志(来自 Amazon EC2 实例)、Kubernetes 审核日志和 DNS 日志。它还包括 EKS 运行时监控。它使用不断更新的威胁情报源,如恶意 IP 地址和域名列表,以及机器学习来识别您 AWS 环境中的意外、可能未经授权和恶意活动。这可能包括权限升级、使用暴露的凭据或与恶意 IP 地址、域名通信、Amazon EC2 实例和 EKS 容器工作负载上存在恶意软件,或发现可疑 API 活动等问题。GuardDuty 通过生成您可以在 GuardDuty 控制台或通过 Amazon EventBridge 查看的安全发现来通知您 AWS 环境的状态。GuardDuty 还支持您将发现导出到 Amazon Simple Storage Service (S3) 存储桶,并与其他服务(如 AWS Security Hub 和 Detective)集成。

观看这个 AWS 在线技术讨论会["使用 Amazon GuardDuty 增强 Amazon EKS 的威胁检测 - AWS 在线技术讨论会"](https://www.youtube.com/watch?v=oNHGRRroJuE),了解如何在几分钟内启用这些额外的 EKS 安全功能。

### 可选:使用第三方解决方案进行运行时监控

如果您不熟悉 Linux 安全性,创建和管理 seccomp 和 Apparmor 配置文件可能很困难。如果您没有时间成为专家,请考虑使用第三方商业解决方案。许多解决方案已经超越了 Apparmor 和 seccomp 等静态配置文件,开始使用机器学习来阻止或警告可疑活动。下面的[工具](#tools-and-resources)部分列出了一些这样的解决方案。您可以在 [AWS Marketplace for Containers](https://aws.amazon.com/marketplace/features/containers) 上找到更多选择。

### 在编写 seccomp 策略之前,请考虑添加/删除 Linux 功能

能力涉及内核函数中可通过系统调用访问的各种检查。如果检查失败,通常系统调用会返回一个错误。检查可以在特定系统调用的开头进行,也可以在可通过多个不同系统调用访问的内核区域更深入地进行(例如写入特定的特权文件)。另一方面,Seccomp是一种系统调用过滤器,在执行系统调用之前应用于所有系统调用。进程可以设置一个过滤器,允许它们撤销运行某些系统调用或某些系统调用的特定参数的权利。

在使用Seccomp之前,请考虑是否通过添加/删除Linux功能可以获得所需的控制。有关更多信息,请参见[为容器设置功能](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#set-capabilities-for-a-container)。

### 查看是否可以通过使用Pod安全策略(PSP)来实现您的目标

Pod安全策略提供了许多不同的方式来提高您的安全态势,而不会引入过度的复杂性。在进入构建Seccomp和Apparmor配置文件之前,请先探索PSP中可用的选项。

!!! warning
    从Kubernetes 1.25开始,PSP已被删除,并被[Pod安全准入](https://kubernetes.io/docs/concepts/security/pod-security-admission/)控制器所取代。现有的第三方替代方案包括OPA/Gatekeeper和Kyverno。可以从GitHub上的[Gatekeeper库](https://github.com/open-policy-agent/gatekeeper-library/tree/master/library/pod-security-policy)仓库中拉取一组Gatekeeper约束和约束模板,用于实施通常在PSP中找到的策略。并且可以在[Kyverno策略库](https://main.kyverno.io/policies/)中找到许多替代PSP的方案,包括完整的[Pod安全标准](https://kubernetes.io/docs/concepts/security/pod-security-standards/)集合。

## 工具和资源

- [在开始之前您应该知道的7件事](https://itnext.io/seccomp-in-kubernetes-part-i-7-things-you-should-know-before-you-even-start-97502ad6b6d6)
- [AppArmor Loader](https://github.com/kubernetes/kubernetes/tree/master/test/images/apparmor-loader)
- [设置带有配置文件的节点](https://kubernetes.io/docs/tutorials/clusters/apparmor/#setting-up-nodes-with-profiles)
- [Security Profiles Operator](https://github.com/kubernetes-sigs/security-profiles-operator)是一个Kubernetes增强功能,旨在使用户更容易在Kubernetes集群中使用SELinux、Seccomp和AppArmor。它提供了从正在运行的工作负载生成配置文件以及将配置文件加载到Kubernetes节点以供Pod使用的功能。
- [Inspektor Gadget](https://github.com/inspektor-gadget/inspektor-gadget)允许检查、跟踪和分析Kubernetes上许多运行时行为方面,包括协助生成Seccomp配置文件。
- [水]
- [Qualys]
- [Stackrox]
- [Sysdig Secure]
- [Prisma]
- [SUSE的NeuVector]开源、零信任容器安全平台,提供进程配置文件规则和文件访问规则。
