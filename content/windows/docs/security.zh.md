!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

# Pod 安全上下文

**Pod 安全策略 (PSP)** 和 **Pod 安全标准 (PSS)** 是在 Kubernetes 中实施安全性的两种主要方式。请注意,PodSecurityPolicy 已在 Kubernetes v1.21 中被弃用,将在 v1.25 中删除,而 Pod 安全标准 (PSS) 是 Kubernetes 推荐的未来实施安全性的方法。

Pod 安全策略 (PSP) 是 Kubernetes 中的一个本地解决方案,用于实施安全策略。PSP 是集群级资源,用于控制 Pod 规范中的安全敏感方面。使用 Pod 安全策略,您可以定义 Pod 必须满足的一组条件,才能被集群接受。
PSP 功能从 Kubernetes 的早期就开始提供,旨在阻止在给定集群上创建配置错误的 Pod。

有关 Pod 安全策略的更多信息,请参考 Kubernetes [文档](https://kubernetes.io/docs/concepts/policy/pod-security-policy/)。根据 [Kubernetes 弃用政策](https://kubernetes.io/docs/reference/using-api/deprecation-policy/),旧版本将在该功能弃用后九个月内停止获得支持。

另一方面,Pod 安全标准 (PSS) 是推荐的安全方法,通常通过安全上下文在 Pod 清单中的 Pod 和容器规范中定义。PSS 是 Kubernetes 项目团队定义的官方标准,用于解决 Pod 的安全相关最佳实践。它定义了基线(最小限制,默认)、特权(无限制)和受限(最严格)等策略。

我们建议从基线配置文件开始。PSS 基线配置文件在安全性和潜在摩擦之间提供了良好的平衡,只需要最少的例外列表,可作为工作负载安全的良好起点。如果您目前正在使用 PSP,我们建议切换到 PSS。有关 PSS 策略的更多详细信息,请参阅 Kubernetes [文档](https://kubernetes.io/docs/concepts/security/pod-security-standards/)。这些策略可以使用包括 [OPA](https://www.openpolicyagent.org/) 和 [Kyverno](https://kyverno.io/) 在内的多种工具强制执行。例如,Kyverno 在[此处](https://kyverno.io/policies/pod-security/)提供了完整的 PSS 策略集合。

安全上下文设置允许您为选定的进程授予特权,使用程序配置文件限制个别程序的功能,允许特权升级,过滤系统调用等。

Kubernetes 中的 Windows Pod 在安全上下文方面与标准基于 Linux 的工作负载存在一些限制和差异。
Windows 使用每个容器的作业对象和系统命名空间过滤器来包含容器中的所有进程,并提供与主机的逻辑隔离。在没有命名空间过滤的情况下,无法运行 Windows 容器。这意味着无法在主机的上下文中断言系统特权,因此 Windows 上不提供特权容器。

以下 `windowsOptions` 是唯一记录的 [Windows 安全上下文选项](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.20/#windowssecuritycontextoptions-v1-core),其余为一般 [安全上下文选项](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.21/#securitycontext-v1-core)。

有关 Windows 与 Linux 中支持的安全上下文属性的列表,请参考官方文档[此处](https://kubernetes.io/docs/setup/production-environment/windows/_print/#v1-container)。

Pod 特定设置应用于所有容器。如果未指定,将使用 PodSecurityContext 中的选项。如果同时在 SecurityContext 和 PodSecurityContext 中设置,则 SecurityContext 中指定的值优先。

例如,Pods 和容器的 runAsUserName 设置是 Linux 特定的 runAsUser 设置的粗略等价物,在以下清单中,pod 特定的安全上下文应用于所有容器。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: run-as-username-pod-demo
spec:
  securityContext:
    windowsOptions:
      runAsUserName: "ContainerUser"
  containers:
  - name: run-as-username-demo
    ...
  nodeSelector:
    kubernetes.io/os: windows
```

在下面的情况下，容器级别的安全上下文会覆盖 pod 级别的安全上下文。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: run-as-username-container-demo
spec:
  securityContext:
    windowsOptions:
      runAsUserName: "ContainerUser"
  containers:
  - name: run-as-username-demo
    ..
    securityContext:
        windowsOptions:
            runAsUserName: "ContainerAdministrator"
  nodeSelector:
    kubernetes.io/os: windows
```

容器用户的可接受值示例: ContainerAdministrator、ContainerUser、NT AUTHORITY\NETWORK SERVICE、NT AUTHORITY\LOCAL SERVICE

通常情况下,建议使用ContainerUser运行Windows pod中的容器。容器和主机之间的用户是不共享的,但ContainerAdministrator在容器内具有额外的权限。请注意,需要注意[用户名限制](https://kubernetes.io/docs/tasks/configure-pod-container/configure-runasusername/#windows-username-limitations)。

使用ContainerAdministrator的一个好例子是设置PATH。您可以使用USER指令来实现,如下所示:
```bash
USER ContainerAdministrator
RUN setx /M PATH "%PATH%;C:/your/path"
USER ContainerUser
```

也请注意,在节点的卷上以明文形式写入密码(与 Linux 上的 tmpfs/内存相比)。这意味着您必须做两件事:

* 使用文件 ACL 来保护密码文件位置
* 使用 [BitLocker](https://docs.microsoft.com/en-us/windows/security/information-protection/bitlocker/bitlocker-how-to-deploy-on-windows-server) 进行卷级加密
