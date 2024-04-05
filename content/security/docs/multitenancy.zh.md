!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

# 租户隔离

当我们考虑多租户时,我们通常希望将用户或应用程序与在共享基础设施上运行的其他用户或应用程序隔离开来。

Kubernetes 是一个_单租户编排器_,即控制平面的单个实例在集群中的所有租户之间共享。但是,您可以使用各种 Kubernetes 对象来创建多租户的假象。例如,可以实施命名空间和基于角色的访问控制(RBAC)来逻辑上隔离租户。同样,配额和限制范围也可用于控制每个租户可以消耗的集群资源量。但是,集群是提供强大安全边界的唯一构造。这是因为能够访问集群中主机的攻击者可以检索该主机上安装的所有_机密、配置映射和卷_。他们还可以模拟 Kubelet,这将允许他们操纵节点的属性和/或在集群内部横向移动。

以下部分将解释如何在减轻使用单租户编排器(如 Kubernetes)的风险的同时实施租户隔离。

## 软多租户

在软多租户中,您使用原生 Kubernetes 构造(例如命名空间、角色和角色绑定以及网络策略)来创建租户之间的逻辑分离。例如,RBAC 可以防止租户访问或操纵彼此的资源。配额和限制范围控制每个租户可以消耗的集群资源量,而网络策略可帮助防止部署到不同命名空间的应用程序相互通信。

但是,这些控制措施并不能防止来自不同租户的 pod 共享节点。如果需要更强的隔离,您可以使用节点选择器、反亲和性规则和/或污点和容忍来强制不同租户的 pod 被调度到单独的节点,通常称为_独占租户节点_。在拥有许多租户的环境中,这可能会变得非常复杂和成本高昂。

!!! attention
    使用命名空间实现的软多租户不允许您为租户提供经过过滤的命名空间列表,因为命名空间是一种全局作用域的类型。如果租户有权查看特定的命名空间,则可以查看集群中的所有命名空间。

!!! warning
    在软多租户中,租户默认保留查询 CoreDNS 以获取集群中运行的所有服务的能力。攻击者可以通过从集群中的任何 pod 运行 `dig SRV *.*.svc.cluster.local` 来利用这一点。如果您需要限制对集群中运行的服务的 DNS 记录的访问,请考虑使用 Firewall 或 Policy 插件for CoreDNS。有关更多信息,请参见 [https://github.com/coredns/policy#kubernetes-metadata-multi-tenancy-policy](https://github.com/coredns/policy#kubernetes-metadata-multi-tenancy-policy)。

[Kiosk](https://github.com/kiosk-sh/kiosk)是一个开源项目,可以帮助实现软多租户。它是通过一系列自定义资源定义(CRD)和控制器实现的,提供以下功能:

- **账户和账户用户**以分隔共享Kubernetes集群中的租户
- **自助服务命名空间供应**供账户用户使用
- **账户限制**以确保在共享集群时的服务质量和公平性
- **命名空间模板**用于安全的租户隔离和自助服务命名空间初始化

[Loft](https://loft.sh)是Kiosk和[DevSpace](https://github.com/devspace-cloud/devspace)维护者提供的商业产品,增加了以下功能:

- **多集群访问**,可以访问不同集群中的空间
- **休眠模式**,在不活跃期间缩减空间中的部署
- **单点登录**,支持GitHub等OIDC身份验证提供商

软多租户可以解决三种主要使用场景。

### 企业环境

第一种是在企业环境中,其中"租户"是半受信任的,他们是员工、承包商或经过组织授权的人。每个租户通常都与部门或团队等行政部门相对应。

在这种环境中,集群管理员通常负责创建命名空间和管理策略。他们还可以实施委托管理模型,让某些个人负责监管命名空间,允许他们执行与策略无关的对象(如部署、服务、Pod、作业等)的CRUD操作。

容器运行时提供的隔离可能在这种环境中已经足够,也可能需要通过其他控制措施来增强Pod安全性。如果需要更严格的隔离,可能还需要限制不同命名空间之间的服务通信。

### Kubernetes即服务

相比之下,软多租户可用于提供Kubernetes即服务(KaaS)的环境。在KaaS中,您的应用程序与一组控制器和CRD一起托管在共享集群中,这些控制器和CRD提供了一套PaaS服务。租户直接与Kubernetes API服务器交互,并被允许对非策略对象执行CRUD操作。租户还可以自行创建和管理自己的命名空间,这增加了自助服务的元素。在这种环境中,租户被假定运行不受信任的代码。

为了在这种环境中隔离租户,您可能需要实施严格的网络策略以及_Pod沙箱化_。沙箱化是指在微型虚拟机(如Firecracker)或用户空间内核中运行Pod中的容器。目前,您可以使用EKS Fargate创建沙箱化的Pod。

### 软件即服务(SaaS)
软多租户的最终用例是在软件即服务(SaaS)环境中。在这种环境中,每个租户都与集群中运行的应用程序的特定实例相关联。每个实例通常都有自己的数据,并使用通常独立于Kubernetes RBAC的单独访问控制。

与其他用例不同,SaaS环境中的租户不会直接与Kubernetes API交互。相反,SaaS应用程序负责与Kubernetes API交互,以创建支持每个租户所需的必要对象。

## Kubernetes构造

在这些实例中,使用以下构造来隔离租户:

### 命名空间

命名空间是实现软多租户的基础。它们允许您将集群划分为逻辑分区。配额、网络策略、服务帐户和实现多租户所需的其他对象都限定在命名空间范围内。

### 网络策略

默认情况下,Kubernetes集群中的所有pod都可以相互通信。这种行为可以使用网络策略进行更改。

网络策略使用标签或IP地址范围限制pod之间的通信。在需要严格的网络隔离的多租户环境中,我们建议从拒绝pod之间通信的默认规则开始,再添加一条允许所有pod查询DNS服务器进行名称解析的规则。有了这些,您就可以开始添加更宽松的规则,允许在命名空间内进行通信。根据需要,这可以进一步细化。

!!! note
    Amazon [VPC CNI现在支持Kubernetes网络策略](https://aws.amazon.com/blogs/containers/amazon-vpc-cni-now-supports-kubernetes-network-policies/)来创建可隔离敏感工作负载并在AWS上运行Kubernetes时保护它们免受未经授权访问的策略。这意味着您可以在Amazon EKS集群中使用网络策略API的所有功能。这种细粒度控制使您能够实施最小权限原则,确保只有授权的pod才能相互通信。

!!! attention
    网络策略是必要的但不足够的。网络策略的执行需要一个策略引擎,如Calico或Cilium。

### 基于角色的访问控制(RBAC)

角色和角色绑定是Kubernetes用于在Kubernetes中实施基于角色的访问控制(RBAC)的对象。**角色**包含可对集群中的对象执行的操作列表。**角色绑定**指定角色适用于哪些个人或组。在企业和KaaS设置中,RBAC可用于允许选定的组或个人管理对象。

### 配额

配额用于定义托管在集群中的工作负载的限制。通过配额,您可以指定 pod 可以消耗的 CPU 和内存的最大量,或者您可以限制可以在集群或命名空间中分配的资源数量。**限制范围**允许您为每个限制声明最小值、最大值和默认值。

在共享集群中过度承诺资源通常是有益的,因为它允许您最大化资源利用。然而,对集群的无限访问可能会导致资源短缺,从而导致性能下降和应用程序可用性损失。如果 pod 的请求设置过低,实际资源利用率超过节点的容量,节点将开始经历 CPU 或内存压力。发生这种情况时,pod 可能会被重新启动和/或从节点中驱逐。

为了防止这种情况发生,您应该计划在多租户环境中对命名空间施加配额,以迫使租户在将 pod 调度到集群上时指定请求和限制。这也将缓解潜在的拒绝服务攻击,通过限制 pod 可以消耗的资源量。

您还可以使用配额来分配集群资源,以与租户的支出保持一致。这在 KaaS 场景中特别有用。

### Pod 优先级和抢占

Pod 优先级和抢占在您希望相对于其他 Pod 为 Pod 提供更高重要性时很有用。例如,通过 pod 优先级,您可以将来自客户 A 的 pod 配置为比来自客户 B 的 pod 具有更高的优先级。当可用容量不足时,调度程序将驱逐来自客户 B 的较低优先级 pod,以容纳来自客户 A 的较高优先级 pod。在 SaaS 环境中,愿意支付溢价的客户获得更高优先级,这可能特别有用。

!!! attention
    Pod 优先级可能会对其他较低优先级的 Pod 产生不利影响。例如,尽管受害 pod 被体面地终止,但 PodDisruptionBudget 并不能得到保证,这可能会破坏依赖 Pod 法定人数的较低优先级应用程序,请参见[抢占的局限性](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#limitations-of-preemption)。

## 缓解控制措施

作为多租户环境的管理员,您最关心的是防止攻击者访问底层主机。应考虑采取以下控制措施来降低这种风险:

### 容器的沙箱执行环境

沙箱化是一种技术,每个容器都在自己的隔离虚拟机中运行。执行 pod 沙箱化的技术包括 [Firecracker](https://firecracker-microvm.github.io/) 和 Weave 的 [Firekube](https://www.weave.works/blog/firekube-fast-and-secure-kubernetes-clusters-using-weave-ignite)。

有关使 Firecracker 成为 EKS 支持的运行时的更多信息,请参见

# 开放策略代理(OPA)和Gatekeeper

[Gatekeeper](https://github.com/open-policy-agent/gatekeeper)是一个Kubernetes准入控制器,它使用[OPA](https://www.openpolicyagent.org/)创建的策略进行强制执行。使用OPA,您可以创建一个策略,该策略在单独的实例上运行租户的pod,或者以比其他租户更高的优先级运行。可以在GitHub[存储库](https://github.com/aws/aws-eks-best-practices/tree/master/policies/opa)中找到一组常见的OPA策略。

还有一个实验性的[OPA插件for CoreDNS](https://github.com/coredns/coredns-opa),允许您使用OPA过滤/控制CoreDNS返回的记录。

# Kyverno

[Kyverno](https://kyverno.io)是一个本地Kubernetes策略引擎,可以使用策略作为Kubernetes资源来验证、变更和生成配置。Kyverno使用Kustomize样式的覆盖进行验证,支持JSON Patch和策略性合并补丁进行变更,并可以根据灵活的触发器在命名空间之间克隆资源。

您可以使用Kyverno隔离命名空间,执行pod安全和其他最佳实践,并生成默认配置,如网络策略。在GitHub[存储库](https://github.com/aws/aws-eks-best-practices/tree/master/policies/kyverno)中包含了几个示例。Kyverno网站的[策略库](https://kyverno.io/policies/)中还包含了许多其他示例。

# 将租户工作负载隔离到特定节点

将租户工作负载限制在专门为其配置的节点上可用于增强软多租户模型中的隔离。使用这种方法,租户特定的工作负载只在为相应租户配置的节点上运行。为实现这种隔离,使用原生Kubernetes属性(节点亲和性、污点和容忍度)来针对特定节点进行pod调度,并防止来自其他租户的pod被调度到租户专用节点。

## 第1部分 - 节点亲和性

Kubernetes [节点亲和性](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)用于根据节点[标签](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)定位调度节点。使用节点亲和性规则,pod会被吸引到与选择器条件匹配的特定节点。在下面的pod规范中,应用了`requiredDuringSchedulingIgnoredDuringExecution`节点亲和性。结果是,pod将针对标有以下键/值的节点:`node-restriction.kubernetes.io/tenant: tenants-x`。
``` yaml
...
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-restriction.kubernetes.io/tenant
            operator: In
            values:
            - tenants-x
...
```

使用此节点亲和性,在调度期间需要标签,但在执行期间不需要;如果底层节点的标签发生变化,pod 不会仅因为该标签变化而被驱逐。但未来的调度可能会受到影响。

!!! 警告
    Kubernetes 中 `node-restriction.kubernetes.io/` 标签前缀有特殊含义。对于 EKS 集群启用的 [NodeRestriction](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction) 阻止 `kubelet` 添加/删除/更新带有此前缀的标签。攻击者无法使用 `kubelet` 的凭据更新节点对象或修改系统设置以将这些标签传递给 `kubelet`,因为 `kubelet` 不允许修改这些标签。如果此前缀用于所有 pod 到节点的调度,它可以防止攻击者通过修改节点标签来吸引不同的工作负载到节点的场景。

!!! 信息
    我们可以使用 [节点选择器](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector) 而不是节点亲和性。但是,节点亲和性更加表达性,允许在 pod 调度期间考虑更多条件。有关差异和更高级调度选择的更多信息,请参见 CNCF 博客文章 [Advanced Kubernetes pod to node scheduling](https://www.cncf.io/blog/2021/07/27/advanced-kubernetes-pod-to-node-scheduling/)。

#### 第二部分 - 污点和容忍

吸引 pod 到节点只是这种三部分方法的第一部分。为了使这种方法奏效,我们必须阻止 pod 被调度到未经授权的节点。为了阻止不需要或未经授权的 pod,Kubernetes 使用节点 [污点](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)。污点用于在节点上设置条件,以防止 pod 被调度。下面的污点使用 `tenant: tenants-x` 的键值对。
``` yaml
...
    taints:
      - key: tenant
        value: tenants-x
        effect: NoSchedule
...
```

鉴于上述节点 `taint`，只有能够 _容忍_ 该污点的 pod 才能被调度到该节点上。为了允许授权的 pod 被调度到该节点上，相应的 pod 规格必须包含对该污点的 `容忍`，如下所示。
``` yaml
...
  tolerations:
  - effect: NoSchedule
    key: tenant
    operator: Equal
    value: tenants-x
...
```

具有上述 `toleration` 的 Pod 不会因为该特定污点而被阻止调度到节点上。污点也被 Kubernetes 用于在某些条件下暂时停止 Pod 调度,例如节点资源压力。通过节点亲和性和污点及容忍,我们可以有效地吸引所需的 Pod 到特定节点并排斥不需要的 Pod。

!!! attention
    某些 Kubernetes Pod 需要在所有节点上运行。这些 Pod 的示例包括由 [Container Network Interface (CNI)](https://github.com/containernetworking/cni) 和 [kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/) [daemonsets](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) 启动的 Pod。为此,这些 Pod 的规范包含非常宽松的容忍,以容忍不同的污点。应该小心不要更改这些容忍。更改这些容忍可能会导致集群操作不正确。此外,可以使用诸如 [OPA/Gatekeeper](https://github.com/open-policy-agent/gatekeeper) 和 [Kyverno](https://kyverno.io/) 等策略管理工具编写验证策略,以防止未经授权的 Pod 使用这些宽松的容忍。

#### 第 3 部分 - 基于策略的节点选择管理

有几种工具可用于帮助管理 Pod 规范的节点亲和性和容忍,包括在 CICD 管道中执行规则。但是,隔离的执行也应该在 Kubernetes 集群级别完成。为此,可以使用策略管理工具基于请求有效负载来_变更_传入的 Kubernetes API 服务器请求,以应用上述相应的节点亲和性规则和容忍。

例如,目标为 _tenants-x_ 命名空间的 Pod 可以被_加上_正确的节点亲和性和容忍,以允许在 _tenants-x_ 节点上进行调度。利用使用 Kubernetes [Mutating Admission Webhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook) 配置的策略管理工具,可以使用策略来变更传入的 Pod 规范。这些变更添加了所需的元素以允许所需的调度。下面显示了一个添加节点亲和性的示例 OPA/Gatekeeper 策略。
``` yaml
apiVersion: mutations.gatekeeper.sh/v1alpha1
kind: Assign
metadata:
  name: mutator-add-nodeaffinity-pod
  annotations:
    aws-eks-best-practices/description: >-
      Adds Node affinity - https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity
spec:
  applyTo:
  - groups: [""]
    kinds: ["Pod"]
    versions: ["v1"]
  match:
    namespaces: ["tenants-x"]
  location: "spec.affinity.nodeAffinity.requiredDuringSchedulingIgnoredDuringExecution.nodeSelectorTerms"
  parameters:
    assign:
      value: 
        - matchExpressions:
          - key: "tenant"
            operator: In
            values:
            - "tenants-x"
```

以上政策适用于 Kubernetes API 服务器请求,以将 pod 应用于 _tenants-x_ 命名空间。该政策添加了 `requiredDuringSchedulingIgnoredDuringExecution` 节点亲和性规则,以便将 pod 吸引到具有 `tenant: tenants-x` 标签的节点。

下面看到的第二个政策,为同一 pod 规范添加了容忍,使用相同的目标命名空间和组、种类及版本的匹配标准。
``` yaml
apiVersion: mutations.gatekeeper.sh/v1alpha1
kind: Assign
metadata:
  name: mutator-add-toleration-pod
  annotations:
    aws-eks-best-practices/description: >-
      Adds toleration - https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/
spec:
  applyTo:
  - groups: [""]
    kinds: ["Pod"]
    versions: ["v1"]
  match:
    namespaces: ["tenants-x"]
  location: "spec.tolerations"
  parameters:
    assign:
      value: 
      - key: "tenant"
        operator: "Equal"
        value: "tenants-x"
        effect: "NoSchedule"
```

以上政策特定于 pods；这是由于策略的 `location` 元素中变异元素的路径。可以编写其他政策来处理创建 pods 的资源,如 Deployment 和 Job 资源。列出的政策和其他示例可以在此指南的配套 [GitHub 项目](https://github.com/aws/aws-eks-best-practices/tree/master/policies/opa/gatekeeper/node-selector)中看到。

这两个变异的结果是 pods 被吸引到所需的节点,同时也不会被特定的节点污点排斥。为了验证这一点,我们可以看到两个 `kubectl` 调用的输出片段,获取标有 `tenant=tenants-x` 的节点,以及获取 `tenants-x` 命名空间中的 pods。
``` bash
kubectl get nodes -l tenant=tenants-x
NAME                                        
ip-10-0-11-255...
ip-10-0-28-81...
ip-10-0-43-107...

kubectl -n tenants-x get pods -owide
NAME                                  READY   STATUS    RESTARTS   AGE   IP            NODE
tenant-test-deploy-58b895ff87-2q7xw   1/1     Running   0          13s   10.0.42.143   ip-10-0-43-107...
tenant-test-deploy-58b895ff87-9b6hg   1/1     Running   0          13s   10.0.18.145   ip-10-0-28-81...
tenant-test-deploy-58b895ff87-nxvw5   1/1     Running   0          13s   10.0.30.117   ip-10-0-28-81...
tenant-test-deploy-58b895ff87-vw796   1/1     Running   0          13s   10.0.3.113    ip-10-0-11-255...
tenant-test-pod                       1/1     Running   0          13s   10.0.35.83    ip-10-0-43-107...
```

从上述输出可以看出,所有 pod 都调度在标有 `tenant=tenants-x` 标签的节点上。简单地说,pod 只会在所需的节点上运行,而其他 pod（没有所需的亲和性和容忍度）则不会。租户工作负载得到了有效隔离。

下面是一个经过变更的 pod 规格示例。
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: tenant-test-pod
  namespace: tenants-x
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: tenant
            operator: In
            values:
            - tenants-x
...
  tolerations:
  - effect: NoSchedule
    key: tenant
    operator: Equal
    value: tenants-x
...
```


!!! attention
    集成到 Kubernetes API 服务器请求流程中的策略管理工具,使用变更和验证准入 Webhook,旨在在指定的时间框架内响应 API 服务器的请求。这通常是 3 秒或更短。如果 Webhook 调用未能在配置的时间内返回响应,则可能会发生或可能不会发生对传入 API 服务器请求的变更和/或验证。这种行为取决于准入 Webhook 配置是设置为[Fail Open 还是 Fail Close](https://open-policy-agent.github.io/gatekeeper/website/docs/#admission-webhook-fail-open-by-default)。

在上述示例中,我们使用了为 OPA/Gatekeeper 编写的策略。但是,还有其他策略管理工具可以处理我们的节点选择用例。例如,这个 [Kyverno 策略](https://kyverno.io/policies/other/add_node_affinity/add_node_affinity/)可用于处理节点亲和性变更。

!!! tip
    如果操作正确,变更策略将对传入的 API 服务器请求有效载荷产生所需的更改。但是,还应包括验证策略,以在允许更改持久化之前验证是否发生了所需的更改。这在将这些策略用于租户到节点隔离时尤其重要。定期检查集群是否存在不需要的配置的审核策略也是一个好主意。

### 参考

- [k-rail](https://github.com/cruise-automation/k-rail) 旨在通过执行某些策略来帮助您保护多租户环境。

- [使用 Amazon EKS 的多租户 SaaS 应用程序的安全实践](https://d1.awsstatic.com/whitepapers/security-practices-for-multi-tenant-saas-apps-using-eks.pdf)

## 硬性多租户

可以通过为每个租户配置单独的集群来实现硬性多租户。虽然这提供了非常强的租户隔离,但也存在一些缺点。

首先,当您有许多租户时,这种方法可能会变得非常昂贵。您不仅需要为每个集群的控制平面成本付费,而且无法在集群之间共享计算资源。这最终会导致碎片化,一部分集群资源利用不足,而另一部分集群资源过度利用。

其次,您可能需要购买或构建特殊的工具来管理所有这些集群。随着时间的推移,管理数百或数千个集群可能会变得太过繁琐。

最后,相比于创建命名空间,为每个租户创建集群的速度会更慢。尽管如此,在高度监管的行业或需要强隔离的 SaaS 环境中,硬性多租户方法可能是必要的。

## 未来方向

Kubernetes 社区已经认识到软多租户的当前缺陷以及硬多租户的挑战。[多租户特殊兴趣小组 (SIG)](https://github.com/kubernetes-sigs/multi-tenancy) 正试图通过几个孵化项目来解决这些缺陷,包括分层命名空间控制器 (HNC) 和虚拟集群。

HNC 提案 (KEP) 描述了一种在命名空间之间创建父子关系的方法,并具有 \[policy\] 对象继承功能,以及租户管理员创建子命名空间的能力。

虚拟集群提案描述了一种为集群中的每个租户创建控制平面服务(包括 API 服务器、控制器管理器和调度器)独立实例的机制(也称为"Kubernetes 上的 Kubernetes")。

[多租户基准](https://github.com/kubernetes-sigs/multi-tenancy/blob/master/benchmarks/README.md) 提案提供了使用命名空间进行隔离和分段的集群共享指南,以及一个命令行工具 [kubectl-mtb](https://github.com/kubernetes-sigs/multi-tenancy/blob/master/benchmarks/kubectl-mtb/README.md) 来验证是否符合这些指南。

## 多集群管理工具和资源

- [Banzai Cloud](https://banzaicloud.com/)
- [Kommander](https://d2iq.com/solutions/ksphere/kommander)
- [Lens](https://github.com/lensapp/lens)
- [Nirmata](https://nirmata.com)
- [Rafay](https://rafay.co/)
- [Rancher](https://rancher.com/products/rancher/)
- [Weave Flux](https://www.weave.works/oss/flux/)
