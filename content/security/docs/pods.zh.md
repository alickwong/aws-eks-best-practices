# Pod 安全

Pod 规范包括各种不同的属性,可以增强或削弱您的整体安全态势。作为 Kubernetes 从业者,您的主要关注点应该是防止在容器中运行的进程逃脱容器运行时的隔离边界,并访问底层主机。

## Linux 功能

在容器内运行的进程默认情况下在 \[Linux\] 根用户的上下文中运行。虽然容器运行时分配给容器的一组 Linux 功能部分约束了根用户在容器内的操作,但这些默认特权可能允许攻击者升级其特权和/或访问绑定到主机的敏感信息,包括 Secrets 和 ConfigMaps。以下是分配给容器的默认功能列表。有关每个功能的更多信息,请参见 [http://man7.org/linux/man-pages/man7/capabilities.7.html](http://man7.org/linux/man-pages/man7/capabilities.7.html)。

`CAP_AUDIT_WRITE, CAP_CHOWN, CAP_DAC_OVERRIDE, CAP_FOWNER, CAP_FSETID, CAP_KILL, CAP_MKNOD, CAP_NET_BIND_SERVICE, CAP_NET_RAW, CAP_SETGID, CAP_SETUID, CAP_SETFCAP, CAP_SETPCAP, CAP_SYS_CHROOT`

!!! Info

  EC2 和 Fargate pod 默认分配了上述功能。此外,Linux 功能只能从 Fargate pod 中删除。

以特权方式运行的 pod 继承了主机上根用户关联的所有 Linux 功能。尽可能避免这种情况。

### 节点授权

所有 Kubernetes 工作节点都使用一种称为 [节点授权](https://kubernetes.io/docs/reference/access-authn-authz/node/) 的授权模式。节点授权授权所有源自 kubelet 的 API 请求,并允许节点执行以下操作:

读取操作:

- services
- endpoints
- nodes
- pods
- 与 kubelet 节点绑定的 pods 相关的 secrets、configmaps、persistent volume claims 和 persistent volumes

写操作:

- nodes 和 node 状态(启用 `NodeRestriction` 准入插件以限制 kubelet 修改其自身节点)
- pods 和 pod 状态(启用 `NodeRestriction` 准入插件以限制 kubelet 修改绑定到自身的 pods)
- events

身份验证相关操作:

- 对 CSR API 的读/写访问权限,用于 TLS 引导
- 创建 TokenReview 和 SubjectAccessReview 的能力,用于委托身份验证/授权检查

EKS 使用 [节点限制准入控制器](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction),它只允许节点修改有限的节点属性集和绑定到节点的 pod 对象。然而,攻击者如果设法访问主机,仍然可以从 Kubernetes API 中获取有关环境的敏感信息,这可能允许他们在集群内部横向移动。

## Pod 安全解决方案

### Pod 安全策略 (PSP)

过去,[Pod 安全策略 (PSP)](https://kubernetes.io/docs/concepts/policy/pod-security-policy/) 资源用于指定 pod 必须满足的一组要求,然后才能创建。从 Kubernetes 版本 1.21 开始,PSP 已被弃用。它们计划在 Kubernetes 版本 1.25 中删除。

!!! Attention

  [PSP 在 Kubernetes 版本 1.21 中被弃用](https://kubernetes.io/blog/2021/04/06/podsecuritypolicy-deprecation-past-present-and-future/)。您将拥有直到版本 1.25 或大约 2 年的时间过渡到替代方案。[此文档](https://github.com/kubernetes/enhancements/blob/master/keps/sig-auth/2579-psp-replacement/README.md#motivation)解释了此弃用的动机。

### 迁移到新的 pod 安全解决方案

由于 PSP 已在 Kubernetes v1.25 中被删除,集群管理员和操作员必须替换这些安全控制。两种解决方案可以满足这一需求:

- Kubernetes 生态系统中的策略即代码 (PAC) 解决方案
- Kubernetes [Pod 安全标准 (PSS)](https://kubernetes.io/docs/concepts/security/pod-security-standards/)

PAC 和 PSS 解决方案可以与 PSP 共存;它们可以在 PSP 被删除之前在集群中使用。这有助于在从 PSP 迁移时采用。在考虑从 PSP 迁移到 PSS 时,请参阅[此文档](https://kubernetes.io/docs/tasks/configure-pod-container/migrate-from-psp/)。

Kyverno 是下面概述的 PAC 解决方案之一,在[博客文章](https://kyverno.io/blog/2023/05/24/podsecuritypolicy-migration-with-kyverno/)中概述了从 PSP 迁移到其解决方案的具体指南,包括类似的策略、功能比较和迁移过程。有关使用 Kyverno 迁移到 Pod 安全准入 (PSA) 的更多信息和指南,已发布在 AWS 博客[此处](https://aws.amazon.com/blogs/containers/managing-pod-security-on-amazon-eks-with-kyverno/)。

### 策略即代码 (PAC)

策略即代码 (PAC) 解决方案提供了护栏,通过规定的和自动化的控制来指导集群用户,并防止不想要的行为。PAC 使用 [Kubernetes 动态准入控制器](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)来拦截 Kubernetes API 服务器请求流,通过 webhook 调用,并根据存储为代码的策略来改变和验证请求有效负载。在 API 服务器请求导致对集群的更改之前,会进行变更和验证。PAC 解决方案使用策略来匹配和操作 API 服务器请求有效负载,基于分类和值。

Kubernetes 生态系统中有几种开源的 PAC 解决方案。这些解决方案不是 Kubernetes 项目的一部分;它们来自 Kubernetes 生态系统。以下列出了一些 PAC 解决方案。

- [OPA/Gatekeeper](https://open-policy-agent.github.io/gatekeeper/website/docs/)
- [Open Policy Agent (OPA)](https://www.openpolicyagent.org/)
- [Kyverno](https://kyverno.io/)
- [Kubewarden](https://www.kubewarden.io/)
- [jsPolicy](https://www.jspolicy.com/)

有关 PAC 解决方案的更多信息以及如何帮助您选择适合您需求的解决方案,请参见下面的链接。

- [Kubernetes 的基于策略的对策 - 第 1 部分](https://aws.amazon.com/blogs/containers/policy-based-countermeasures-for-kubernetes-part-1/)
- [Kubernetes 的基于策略的对策 - 第 2 部分](https://aws.amazon.com/blogs/containers/policy-based-countermeasures-for-kubernetes-part-2/)

### Pod 安全标准 (PSS) 和 Pod 安全准入 (PSA)

为了应对 PSP 弃用以及持续控制 pod 安全的需求,Kubernetes [Auth 特别兴趣小组](https://github.com/kubernetes/community/tree/master/sig-auth)创建了 [Pod 安全标准 (PSS)](https://kubernetes.io/docs/concepts/security/pod-security-standards/) 和 [Pod 安全准入 (PSA)](https://kubernetes.io/docs/concepts/security/pod-security-admission/)。PSA 工作包括一个[准入控制器 webhook 项目](https://github.com/kubernetes/pod-security-admission#pod-security-admission),它实现了 PSS 中定义的控制。这种准入控制器方法类似于 PAC 解决方案中使用的方法。

根据 Kubernetes 文档,PSS _"定义了三种不同的策略,广泛覆盖了安全范围。这些策略是累积的,范围从高度宽松到高度限制性。"_

这些策略定义如下:

- **特权:** 无限制(不安全)策略,提供最广泛的权限级别。此策略允许已知的特权升级。这是没有策略的情况。这对于日志代理、CNI、存储驱动程序和其他系统范围的应用程序很有用,它们需要特权访问。

- **基线:** 最小限制性策略,可防止已知的特权升级。允许默认(最小指定)的 Pod 配置。基线策略禁止使用 hostNetwork、hostPID、hostIPC、hostPath、hostPort,以及添加 Linux 功能的能力,以及其他几个限制。

- **受限:** 高度受限的策略,遵循当前 Pod 加固最佳实践。此策略继承自基线,并添加了更多限制,例如无法以 root 或 root 组运行。受限策略可能会影响应用程序的功能。它们主要针对运行关键安全应用程序。

这些策略定义了[pod 执行配置文件](https://kubernetes.io/docs/concepts/security/pod-security-standards/#profile-details),分为三个特权与受限访问级别。

为了实现 PSS 定义的控制,PSA 以三种模式运行:

- **enforce:** 策略违规将导致 pod 被拒绝。

- **audit:** 策略违规将触发在审核日志中添加审核注释,但否则允许。

- **warn:** 策略违规将触发用户可见的警告,但否则允许。

这些模式和配置文件(限制)级别在 Kubernetes 命名空间级别使用标签进行配置,如下例所示。

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: policy-test
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

单独使用时,这些操作模式有不同的响应,导致不同的用户体验。_enforce_ 模式将阻止创建违反配置的限制级别的 pod。但是,在此模式下,创建 pod 的 Kubernetes 对象(如 Deployment)将不会被阻止应用到集群,即使其中的 podSpec 违反了应用的 PSS。在这种情况下,Deployment 将被应用,而 pod(s) 将被阻止应用。

这是一个困难的用户体验,因为成功应用的 Deployment 对象掩盖了失败的 pod 创建。违反的 podSpec 将无法创建 pod。使用 `kubectl get deploy <DEPLOYMENT_NAME> -oyaml` 检查 Deployment 资源将暴露来自失败 pod(s) `.status.conditions` 元素的消息,如下所示。

```yaml
...
status:
  conditions:
    - lastTransitionTime: "2022-01-20T01:02:08Z"
      lastUpdateTime: "2022-01-20T01:02:08Z"
      message: 'pods "test-688f68dc87-tw587" is forbidden: violates PodSecurity "restricted:latest":
        allowPrivilegeEscalation != false (container "test" must set securityContext.allowPrivilegeEscalation=false),
        unrestricted capabilities (container "test" must set securityContext.capabilities.drop=["ALL"]),
        runAsNonRoot != true (pod or container "test" must set securityContext.runAsNonRoot=true),
        seccompProfile (pod or container "test" must set securityContext.seccompProfile.type
        to "RuntimeDefault" or "Localhost")'
      reason: FailedCreate
      status: "True"
      type: ReplicaFailure
...
```

在 _audit_ 和 _warn_ 模式下,pod 限制不会阻止违规 pod 被创建和启动。但是,在这些模式下,当 pod 以及创建 pod 的对象包含违规的 podSpec 时,会触发 API 服务器审核日志事件的审核注释和 API 服务器客户端(如 _kubectl_)的警告。下面可以看到一个 `kubectl` _Warning_ 消息。

```bash
Warning: would violate PodSecurity "restricted:latest": allowPrivilegeEscalation != false (container "test" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "test" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "test" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "test" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
deployment.apps/test created
```

PSA _audit_ 和 _warn_ 模式在不会对集群操作产生负面影响的情况下引入 PSS 很有用。

PSA 操作模式不是互斥的,可以以累积的方式使用。如下所示,多种模式可以在单个命名空间中配置。

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: policy-test
  labels:
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
```

在上面的示例中,当应用 Deployment 时,提供了用户友好的警告和审核注释,同时在 pod 级别也提供了违规的强制执行。事实上,多个 PSA 标签可以使用不同的配置文件级别,如下所示。

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: policy-test
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/warn: restricted
```

在上面的示例中,PSA 被配置为允许满足 _baseline_ 配置文件级别的所有 pod 的创建,并在 pod(和创建 pod 的对象)违反 _restricted_ 配置文件级别时发出警告。这是一种有用的方法,可以确定从 _baseline_ 到 _restricted_ 配置文件时可能产生的影响。

#### 现有 Pod

如果使用更严格的 PSS 配置文件修改了包含现有 pod 的命名空间,_audit_ 和 _warn_ 模式将产生适当的消息;但是,_enforce_ 模式不会删除 pod。可以看到以下警告消息。

```bash
Warning: existing pods in namespace "policy-test" violate the new PodSecurity enforce level "restricted:latest"
Warning: test-688f68dc87-htm8x: allowPrivilegeEscalation != false, unrestricted capabilities, runAsNonRoot != true, seccompProfile
namespace/policy-test configured
```

#### 豁免

PSA 使用 _Exemptions_ 来排除对本应被应用的 pod 的违规执行。这些豁免列举如下。

- **用户名:** 来自具有豁免认证(或模拟)用户名的请求被忽略。

- **RuntimeClassNames:** 指定豁免运行时类名的 pod 和工作负载资源被忽略。

- **命名空间:** 在豁免命名空间中的 pod 和工作负载资源被忽略。

这些豁免静态地应用在 [PSA 准入控制器配置](https://kubernetes.io/docs/tasks/configure-pod-container/enforce-standards-admission-controller/#configure-the-admission-controller)中,作为 API 服务器配置的一部分。

在 _Validating Webhook_ 实现中,可以在 Kubernetes [ConfigMap](https://github.com/kubernetes/pod-security-admission/blob/master/webhook/manifests/20-configmap.yaml) 资源中配置豁免,该资源作为卷挂载到 [pod-security-webhook](https://github.com/kubernetes/pod-security-admission/blob/master/webhook/manifests/50-deployment.yaml) 容器中。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: pod-security-webhook
  namespace: pod-security-webhook
data:
  podsecurityconfiguration.yaml: |
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: "restricted"
      enforce-version: "latest"
      audit: "restricted"
      audit-version: "latest"
      warn: "restricted"
      warn-version: "latest"
    exemptions:
      # Array of authenticated usernames to exempt.
      usernames: []
      # Array of runtime class names to exempt.
      runtimeClasses: []
      # Array of namespaces to exempt.
      namespaces: ["kube-system","policy-test1"]
```

如上面的 ConfigMap YAML 所示,集群范围的默认 PSS 级别已设置为所有 PSA 模式(_audit_、_enforce_ 和 _warn_)的 _restricted_。这会影响所有命名空间,除了被豁免的命名空间:"namespaces: ["kube-system","policy-test1"]"。此外,在下面看到的 _ValidatingWebhookConfiguration_ 资源中,_pod-security-webhook_ 命名空间也被豁免于配置的 PSS。

```yaml
...
webhooks:
  # Audit annotations will be prefixed with this name
  - name: "pod-security-webhook.kubernetes.io"
    # Fail-closed admission webhooks can present operational challenges.
    # You may want to consider using a failure policy of Ignore, but should 
    # consider the security tradeoffs.
    failurePolicy: Fail
    namespaceSelector:
      # Exempt the webhook itself to avoid a circular dependency.
      matchExpressions:
        - key: kubernetes.io/metadata.name
          operator: NotIn
          values: ["pod-security-webhook"]
...
```

!!! Attention

  Pod 安全准入在 Kubernetes v1.25 中升级为稳定版。如果您想在默认启用之前使用 Pod 安全准入功能,您需要安装动态准入控制器(变异 webhook)。安装和配置 webhook 的说明可以在[此处](https://github.com/kubernetes/pod-security-admission/tree/master/webhook)找到。

### 在策略即代码和 Pod 安全标准之间进行选择

Pod 安全标准 (PSS) 是为了取代 Pod 安全策略 (PSP) 而开发的,提供了一个内置于 Kubernetes 的解决方案,不需要 Kubernetes 生态系统的解决方案。也就是说,策略即代码 (PAC) 解决方案要灵活得多。

以下优缺点列表旨在帮助您做出更明智的 pod 安全解决方案决策。

#### 策略即代码 (与 Pod 安全标准相比)

优点:

- 更灵活,更细粒度(如果需要,可以到资源属性)
- 不仅关注于 pod,可以用于不同的资源和操作
- 不仅应用于命名空间级别
- 比 Pod 安全标准更成熟
- 决策可以基于 API 服务器请求有效负载中的任何内容,以及现有集群资源和外部数据(取决于解决方案)
- 支持在验证之前改变 API 服务器请求(取决于解决方案)
- 可以生成补充性策略和 Kubernetes 资源(取决于解决方案 - 从 pod 策略,Kyverno 可以[自动生成](https://kyverno.io/docs/writing-policies/autogen/)针对更高级别控制器(如 Deployment)的策略。Kyverno 还可以通过使用[生成规则](https://kyverno.io/docs/writing-policies/generate/)在创建新资源或源更新时生成额外的 Kubernetes 资源)
- 可用于向左移动,进入 CICD 管道,在调用 Kubernetes API 服务器之前(取决于解决方案)
- 可用于实现不一定与安全相关的行为,如最佳实践、组织标准等
- 可用于非 Kubernetes 用例(取决于解决方案)
- 由于灵活性,用户体验可以根据用户需求进行调整

缺点:

- 不内置于 Kubernetes
- 学习、配置和支持更加复杂
- 策略编写可能需要新的技能/语言/功能

#### Pod 安全准入 (与策略即代码相比)

优点:

- 内置于 Kubernetes
- 配置更简单
- 无需使用新语言或编写策略
- 如果集群默认准入级别配置为 _privileged_,可以使用命名空间标签将命名空间选择加入 pod 安全配置文件。

缺点:

- 不如策略即代码灵活或细粒度
- 只有 3 个限制级别
- 主要关注于 pod

#### 总结

如果您目前没有 pod 安全解决方案,除了 PSP 之外,并且您所需的 pod 安全态势符合 Pod 安全标准 (PSS) 定义的模型,那么采用 PSS 可能是一条更容易的路径,而不是使用策略即代码解决方案。但是,如果您的 pod 安全态势不符合 PSS 模型,或者您设想添加超出 PSS 定义的其他控制,那么策略即代码解决方案可能更适合您。

## 建议

### 使用多种 Pod 安全准入 (PSA) 模式以获得更好的用户体验

如前所述,PSA _enforce_ 模式会阻止违反 PSS 的 pod 被应用,但不会阻止更高级别的控制器(如 Deployment)。事实上,Deployment 将成功应用,而没有任何表明 pod 失败的迹象。虽然您可以使用 `kubectl` 检查 Deployment 对象,并从 PSA 发现失败的 pod 消息,但用户体验可能会更好。为了提供更好的用户体验,应该使用多种 PSA 模式(审核、强制、警告)。

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: policy-test
  labels:
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
```

在上面的示例中,使用了 _enforce_ 模式,当尝试将包含 PSS 违规的 podSpec 的 Deployment 清单应用到 Kubernetes API 服务器时,Deployment 将成功应用,但 pod 将不会。而且,由于还启用了 _audit_ 和 _warn_ 模式,API 服务器客户端将收到警告消息,API 服务器审核日志事件也将被注释。

### 限制可以以特权方式运行的容器

如前所述,以特权方式运行的容器继承了主机上根用户分配的所有 Linux 功能。很少有容器需要这种特权才能正常运行。有多种方法可用于限制容器的权限和功能。

!!! Attention

  Fargate 是一种启动类型,它使您能够运行"无服务器"容器,其中 pod 的容器在 AWS 管理的基础设施上运行。使用 Fargate,您无法运行特权容器或配置 pod 使用 hostNetwork 或 hostPort。

### 不要在容器中以 root 身份运行进程

所有容器默认以 root 身份运行。如果攻击者能够利用应用程序中的漏洞并获得容器运行时的 shell 访问权限,这可能会造成问题。您可以通过多种方式来缓解这种风险。首先,从容器镜像中删除 shell。其次,在 Dockerfile 中添加 USER 指令,或在 pod 中以非 root 用户运行容器。Kubernetes podSpec 包括一组字段,在 `spec.securityContext` 下,允许您指定应用程序运行的用户和/或组。这些字段分别是 `runAsUser` 和 `runAsGroup`。

为了强制执行 Kubernetes podSpec 中的 `spec.securityContext` 及其相关元素,可以在集群中添加策略即代码或 Pod 安全标准。这些解决方案允许您编写和/或使用可以验证入站 Kubernetes API 服务器请求有效负载的策略或配置文件,然后将其持久化到 etcd 中。此外,策略即代码解决方案可以改变入站请求,在某些情况下还可以生成新的请求。

### 永远不要在容器中运行 Docker 或挂载套接字

虽然这样可以方便地在 Docker 容器中构建/运行镜像,但您基本上是将节点的完全控制权交给了容器中运行的进程。如果您需要在 Kubernetes 上构建容器镜像,请使用 [Kaniko](https://github.com/GoogleContainerTools/kaniko)、[buildah](https://github.com/containers/buildah) 或构建服务如 [CodeBuild](https://docs.aws.amazon.com/codebuild/latest/userguide/welcome.html)。

!!! Tip

  用于 CICD 处理(如构建容器镜像)的 Kubernetes 集群应与运行更一般工作负载的集群隔离。

### 限制使用 hostPath,如果需要使用 hostPath,请限制可以使用的前缀并将卷配置为只读

`hostPath` 是一个将主机上的目录直接挂载到容器的卷。很少有 pod 需要这种访问,但如果需要,您需要注意风险。默认情况下,以 root 身份运行的 pod 将对 hostPath 公开的文件系统具有写访问权限。这可能允许攻击者修改 kubelet 设置、创建指向未直接公开的目录或文件(如 /etc/shadow)的符号链接、安装 ssh 密钥、读取挂载到主机的 secret 等恶意操作。为了缓解 hostPath 的风险,请配置 `spec.containers.volumeMounts` 为 `readOnly`:

```yaml
volumeMounts:
- name: hostPath-volume
    readOnly: true
    mountPath: /host-path
```

您还应该使用策略即代码解决方案来限制可以使用的 `hostPath` 卷的目录,或者完全禁止使用 `hostPath`。您可以使用 Pod 安全标准的 _Baseline_ 或 _Restricted_ 策略来防止使用 `hostPath`。

有关特权升级危险的更多信息,请阅读 Seth Art 的博客[Bad Pods: Kubernetes Pod Privilege Escalation](https://labs.bishopfox.com/tech-blog/bad-pods-kubernetes-pod-privilege-escalation)。

### 为每个容器设置请求和限制,以避免资源争用和 DoS 攻击

没有请求或限制的 pod 理论上可以消耗节点上可用的所有资源。随着更多 pod 被调度到节点上,节点可能会经历 CPU 或内存压力,这可能导致 Kubelet 终止或驱逐节点上的 pod。虽然您无法完全防止这种情况发生,但设置请求和限制将有助于最小化资源争用,并缓解由于编写不当而消耗过多资源的应用程序的风险。

`podSpec` 允许您为 CPU 和内存指定请求和限制。CPU 被视为可压缩资源,因为它可以被过度订阅。内存是不可压缩的,即不能在多个容器之间共享。

当您指定 _requests_ 的 CPU 或内存时,您实际上是在指定容器将获得的 _内存_ 量。Kubernetes 会聚合 pod 中所有容器的请求,以确定将 pod 调度到哪个节点上。如果容器超过请求的内存量,如果节点出现内存压力,它可能会被终止。

_Limits_ 是容器允许消耗的 CPU 和内存资源的最大值,直接对应于为容器创建的 cgroup 的 `memory.limit_in_bytes` 值。容器如果超过内存限制将被 OOM 杀死。如果容器超过 CPU 限制,它将被节流。

!!! Tip

  使用容器 `resources.limits` 时,强烈建议基于负载测试的数据驱动和准确的容器资源使用情况(即资源足迹)。在没有准确和可信的资源足迹的情况下,可以对容器 `resources.limits` 进行填充。例如,`resources.limits.memory` 可以填充 20-30% 高于可观察的最大值,以应对潜在的内存资源限制不准确。

Kubernetes 使用三种服务质量(QoS)类来优先处理节点上运行的工作负载。它们包括:

- guaranteed
- burstable
- best-effort

如果未设置限制和请求,pod 将配置为 _best-effort_(最低优先级)。当内存不足时,首先会杀死 best-effort pod。如果 pod 中所有容器都设置了限制,或者请求和限制设置为相同的值且不等于 0,pod 将配置为 _guaranteed_(最高优先级)。Guaranteed pod 不会被杀死,除非它们超过配置的内存限制。如果限制和请求配置为不同的值且不等于 0,或者 pod 中的一个容器设置了限制而其他容器没有或设置了不同的资源限制,pod 将配置为 _burstable_(中等优先级)。这些 pod 有一些资源保证,但如果超过请求的内存,也可能被杀死。

!!! Attention

  请求不会影响容器 cgroup 的 `memory_limit_in_bytes` 值;cgroup 限制设置为主机上可用的内存量。但是,如果将请求值设置得太低,如果节点出现内存压力,pod 可能会被 kubelet 选中终止。

| 类别 | 优先级 | 条件 | 杀死条件 |
| :-- | :-- | :-- | :-- |
| Guaranteed | 最高 | limit = request != 0  | 仅超过内存限制 |
| Burstable  | 中等  | limit != request != 0 | 如果超过请求内存可能被杀死 |
| Best-Effort| 最低  | limit & request 未设置 | 内存不足时首先被杀死 |

有关资源 QoS 的更多信息,请参考 [Kubernetes 文档](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)。

您可以通过在命名空间上设置[资源配额](https://kubernetes.io/docs/concepts/policy/resource-quotas/)或创建[限制范围](https://kubernetes.io/docs/concepts/policy/limit-range/)来强制使用请求和限制。资源配额允许您指定分配给命名空间的资源总量,如 CPU 和 RAM。当应用于命名空间时,它会强制您为部署到该命名空间的所有容器指定请求和限制。相比之下,限制范围让您更细粒度地控制资源分配。使用限制范围,您可以为 pod 或命名空间中的每个容器设置 CPU 和内存资源的最小/最大值。您还可以使用它们来设置默认的请求/限制值(如果未提供)。

策略即代码解决方案可用于强制执行请求和限制,或在创建命名空间时创建资源配额和限制范围。

### 不允许特权升级

特权升级允许进程更改其正在运行的安全上下文。sudo 就是一个很好的例子,二进制文件具有 SUID 或 SGID 位也是如此。特权升级基本上是用户执行文件的一种方式,以另一个用户或组的权限执行。您可以通过实现一个改变策略的策略即代码策略来防止容器使用特权升级,该策略将 `allowPrivilegeEscalation` 设置为 `false`,或者在 `podSpec` 中设置 `securityContext.allowPrivilegeEscalation`。策略即代码策略还可用于在检测到不正确的设置时防止 API 服务器请求成功。Pod 安全标准也可用于防止 pod 使用特权升级。

### 禁用 ServiceAccount 令牌挂载

对于不需要访问 Kubernetes API 的 pod,您可以禁用在 pod 规范上自动挂载 ServiceAccount 令牌。

!!! Attention

  禁用 ServiceAccount 挂载不会阻止 pod 访问 Kubernetes API。要完全阻止 pod 访问 Kubernetes API,您需要修改 [EKS 集群端点访问](https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html),并使用 [NetworkPolicy](../network/#network-policy) 阻止 pod 访问。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-no-automount
spec:
  automountServiceAccountToken: false
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-no-automount
automountServiceAccountToken: false
```

### 禁用服务发现

对于不需要查找或调用集群内服务的 pod,您可以减少提供给 pod 的信息量。您可以将 Pod 的 DNS 策略设置为不使用 CoreDNS,并且不将命名空间中的服务公开为环境变量。有关环境变量的更多信息,请参见 [Kubernetes 文档](https://kubernetes.io/docs/concepts/services-networking/service/#environment-variables)。pod 的默认 DNS 策略值为 "ClusterFirst",使用集群内 DNS,而非默认值 "Default" 使用底层节点的 DNS 解析。有关 Pod DNS 策略的更多信息,请参见 [Kubernetes 文档](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy)。

!!! Attention

  禁用服务链接和更改 pod 的 DNS 策略不会阻止 pod 访问集群内 DNS 服务。攻击者仍然可以通过访问集群内 DNS 服务来枚举集群中的服务(例如: `dig SRV *.*.svc.cluster.local @$CLUSTER_DNS_IP`)。要阻止集群内服务发现,请使用 [NetworkPolicy](../network/#network-policy) 阻止 pod 访问。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-no-service-info
spec:
    dnsPolicy: Default # "Default" 不是真正的默认值
    enableServiceLinks: false
```

### 配置您的镜像使用只读根文件系统

配置您的镜像使用只读根文件系统可以防止攻击者覆盖您的应用程序使用的二进制文件。如果您的应用程序需要写入文件系统,请考虑写入临时目录或附加和挂载卷。您可以通过设置 pod 的 SecurityContext 来强制执行这一行为:

```yaml
...
securityContext:
  readOnlyRootFilesystem: true
...
```

策略即代码和 Pod 安全标准可用于强制执行此行为。

!!! Info

  根据 [Windows 容器在 Kubernetes 中的使用](https://kubernetes.io/docs/concepts/windows/intro/),`securityContext.readOnlyRootFilesystem` 不能设置为 `true`,因为 Windows 容器需要写访问权限才能运行注册表和系统进程。

## 工具和资源

- [Amazon EKS 安全沉浸式研讨会 - Pod 安全](https://catalog.workshops.aws/eks-security-immersionday/en-US/3-pod-security)
- [open-policy-agent/gatekeeper-library: The OPA Gatekeeper policy library](https://github.com/open-policy-agent/gatekeeper-library) 一个 OPA/Gatekeeper 策略库,可用作 PSP 的替代品。
- [Kyverno 策略库](https://kyverno.io/policies/)
- 一些常见的 OPA 和 Kyverno [策略](https://github.com/aws/aws-eks-best-practices/tree/master/policies)用于 EKS。
- [基于策略的对策:第 1 部分](https://aws.amazon.com/blogs/containers/policy-based-countermeasures-for-kubernetes-part-1/)
- [基于策略的对策:第 2 部分](https://aws.amazon.com/blogs/containers/policy-based-countermeasures-for-kubernetes-part-2/)
- [Pod 安全策略迁移器](https://appvia.github.io/psp-migration/) 一个将 PSP 转换为 OPA/Gatekeeper、KubeWarden 或 Kyverno 策略的工具
- [NeuVector by SUSE](https://www.suse.com/neuvector/) 开源的零信任容器安全平台,提供进程和文件系统策略以及准入控制规则。