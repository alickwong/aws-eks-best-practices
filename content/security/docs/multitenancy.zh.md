# 租户隔离

当我们考虑多租户时,我们通常希望将用户或应用程序与在共享基础设施上运行的其他用户或应用程序隔离开来。

Kubernetes是一个_单租户编排器_,即控制平面的单个实例在集群中的所有租户之间共享。但是,您可以使用各种Kubernetes对象来创建多租户的外观。例如,可以实施命名空间和基于角色的访问控制(RBAC)来在逻辑上隔离租户。同样,配额和限制范围可用于控制每个租户可以消耗的集群资源量。尽管如此,集群仍然是提供强大安全边界的唯一构造。这是因为能够访问集群中主机的攻击者可以检索该主机上安装的所有_机密_、ConfigMaps和卷。他们还可以冒充Kubelet,这将允许他们操纵节点的属性和/或在集群内部横向移动。

以下部分将解释如何在缓解使用单租户编排器(如Kubernetes)的风险的同时实施租户隔离。

## 软多租户

通过软多租户,您使用本机Kubernetes构造(例如命名空间、角色和角色绑定以及网络策略)来创建租户之间的逻辑分离。例如,RBAC可以防止租户访问或操纵彼此的资源。配额和限制范围控制每个租户可以消耗的集群资源量,而网络策略可帮助防止部署到不同命名空间的应用程序相互通信。

但是,这些控制措施并不能防止来自不同租户的pod共享节点。如果需要更强的隔离,您可以使用节点选择器、反亲和性规则和/或污点和容忍来强制来自不同租户的pod被调度到单独的节点;通常称为_独占租户节点_。这可能会变得相当复杂和成本高昂,尤其是在有许多租户的环境中。

!!! attention
    使用命名空间实现的软多租户不允许您为租户提供经过过滤的命名空间列表,因为命名空间是一种全局作用域的类型。如果租户有权查看特定的命名空间,则可以查看集群中的所有命名空间。

!!! warning
    在软多租户中,租户默认保留查询CoreDNS以获取集群中运行的所有服务的能力。攻击者可以利用这一点,从集群中的任何pod运行`dig SRV *.*.svc.cluster.local`。如果您需要限制对集群中运行的服务的DNS记录的访问,请考虑使用CoreDNS的防火墙或策略插件。有关更多信息,请参见[https://github.com/coredns/policy#kubernetes-metadata-multi-tenancy-policy](https://github.com/coredns/policy#kubernetes-metadata-multi-tenancy-policy)。

[Kiosk](https://github.com/kiosk-sh/kiosk)是一个开源项目,可以帮助实施软多租户。它实现为一系列CRD和控制器,提供以下功能:

- **帐户和帐户用户**以在共享Kubernetes集群中分隔租户
- **自助服务命名空间供应**供帐户用户使用
- **帐户限制**以确保共享集群时的服务质量和公平性
- **命名空间模板**用于安全的租户隔离和自助服务命名空间初始化

[Loft](https://loft.sh)是Kiosk和[DevSpace](https://github.com/devspace-cloud/devspace)维护者提供的商业产品,增加了以下功能:

- **多集群访问**用于授予对不同集群中空间的访问权限
- **睡眠模式**在不活动期间缩减空间中的部署
- **单点登录**使用GitHub等OIDC身份验证提供程序

可以通过软多租户解决三个主要用例。

### 企业环境

第一个是在企业环境中,其中"租户"是半受信任的,因为他们是员工、承包商或否则被组织授权。每个租户通常都与行政部门(如部门或团队)保持一致。

在这种环境中,集群管理员通常负责创建命名空间和管理策略。他们还可能实施委托管理模型,其中某些个人被授予对命名空间的监督权,允许他们执行与策略无关的对象(如部署、服务、pod、作业等)的CRUD操作。

容器运行时提供的隔离可能在此环境中可以接受,或者可能需要通过其他控制措施进行增强,以实现更严格的pod安全性。如果需要更严格的隔离,也可能需要限制不同命名空间中服务之间的通信。

### Kubernetes即服务

相比之下,软多租户可用于提供Kubernetes即服务(KaaS)的环境。在KaaS中,您的应用程序与一组控制器和CRD一起托管在共享集群中,这些控制器和CRD提供了一组PaaS服务。租户直接与Kubernetes API服务器交互,并被允许对非策略对象执行CRUD操作。这种环境中也存在自助服务元素,因为租户可能被允许创建和管理自己的命名空间。在这种环境中,租户被假定在运行不受信任的代码。

为了隔离这种环境中的租户,您可能需要实施严格的网络策略以及_pod沙箱化_。沙箱化是指在微型虚拟机(如Firecracker)或用户空间内核中运行pod的容器。如今,您可以使用EKS Fargate创建沙箱化pod。

### 软件即服务(SaaS)

软多租户的最后一个用例是在软件即服务(SaaS)环境中。在这种环境中,每个租户都与集群中运行的特定_实例_的应用程序相关联。每个实例通常都有自己的数据,并使用通常独立于Kubernetes RBAC的单独访问控制。

与其他用例不同,SaaS环境中的租户不会直接与Kubernetes API交互。相反,SaaS应用程序负责与Kubernetes API交互,以创建支持每个租户所需的必要对象。

## Kubernetes构造

在这些情况下,使用以下构造来隔离租户:

### 命名空间

命名空间是实施软多租户的基础。它们允许您将集群划分为逻辑分区。配额、网络策略、服务帐户和其他实现多租户所需的对象都作用于命名空间。

### 网络策略

默认情况下,Kubernetes集群中的所有pod都可以相互通信。可以使用网络策略改变这种行为。

网络策略使用标签或IP地址范围限制pod之间的通信。在需要严格的网络隔离的多租户环境中,我们建议从拒绝pod之间的通信的默认规则开始,再添加一条允许所有pod查询DNS服务器以进行名称解析的规则。有了这些,您就可以开始添加更宽松的规则,允许在命名空间内进行通信。这可以根据需要进一步细化。

!!! note
    Amazon [VPC CNI现在支持Kubernetes网络策略](https://aws.amazon.com/blogs/containers/amazon-vpc-cni-now-supports-kubernetes-network-policies/)来创建可隔离敏感工作负载并在AWS上运行Kubernetes时保护它们免受未经授权访问的策略。这意味着您可以在Amazon EKS集群中使用网络策略API的所有功能。这种细粒度控制使您能够实施最小特权原则,确保只有授权的pod才能相互通信。

!!! attention
    网络策略是必要的但不足够的。网络策略的执行需要策略引擎,如Calico或Cilium。

### 基于角色的访问控制(RBAC)

角色和角色绑定是Kubernetes用于在Kubernetes中实施基于角色的访问控制(RBAC)的对象。**角色**包含可对集群中的对象执行的操作列表。**角色绑定**指定角色适用的个人或组。在企业和KaaS环境中,RBAC可用于允许选定的组或个人管理对象。

### 配额

配额用于定义托管在集群中的工作负载的限制。使用配额,您可以指定pod可以消耗的最大CPU和内存量,或者您可以限制可在集群或命名空间中分配的资源数量。**限制范围**允许您声明每个限制的最小值、最大值和默认值。

在共享集群中过度承诺资源通常是有益的,因为它允许您最大化资源利用率。但是,对集群的无限访问可能会导致资源耗尽,从而导致性能下降和应用程序可用性损失。如果pod的请求设置得太低,而实际资源利用率超过了节点的容量,节点将开始经历CPU或内存压力。发生这种情况时,pod可能会被重新启动和/或从节点中驱逐。

为了防止这种情况发生,您应该计划在多租户环境中对命名空间施加配额,以迫使租户在将pod调度到集群上时指定请求和限制。它还将缓解潜在的拒绝服务攻击,因为它会限制pod可以消耗的资源量。

您还可以使用配额来分配集群资源,使其与租户的支出保持一致。这在KaaS场景中特别有用。

### Pod优先级和抢占

Pod优先级和抢占在您希望相对于其他Pod提供更高重要性的Pod时很有用。例如,通过pod优先级,您可以将来自客户A的pod配置为比来自客户B的pod具有更高的优先级。当可用容量不足时,调度程序将驱逐来自客户B的较低优先级pod,以容纳来自客户A的较高优先级pod。这在SaaS环境中特别有用,在该环境中愿意支付溢价的客户将获得更高的优先级。

!!! attention
    Pod优先级可能会对其他较低优先级的Pod产生不利影响。例如,尽管受害pod是以优雅的方式终止的,但不能保证PodDisruptionBudget,这可能会破坏依赖于pod仲裁的较低优先级应用程序,请参见[抢占的局限性](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#limitations-of-preemption)。

## 缓解控制措施

作为多租户环境的管理员,您最关心的是防止攻击者访问底层主机。应考虑以下控制措施来缓解这种风险:

### 容器的沙箱执行环境

沙箱化是一种技术,通过该技术每个容器都在自己的隔离虚拟机中运行。执行pod沙箱化的技术包括[Firecracker](https://firecracker-microvm.github.io/)和Weave的[Firekube](https://www.weave.works/blog/firekube-fast-and-secure-kubernetes-clusters-using-weave-ignite)。

有关使Firecracker成为EKS支持的运行时的工作的更多信息,请参见
[https://threadreaderapp.com/thread/1238496944684597248.html](https://threadreaderapp.com/thread/1238496944684597248.html)。

### 开放策略代理(OPA)和Gatekeeper

[Gatekeeper](https://github.com/open-policy-agent/gatekeeper)是一个Kubernetes准入控制器,它使用[OPA](https://www.openpolicyagent.org/)创建的策略进行强制执行。使用OPA,您可以创建一个策略,该策略在单独的实例上运行来自不同租户的pod,或者将它们的优先级设置得高于其他租户。可以在GitHub [repository](https://github.com/aws/aws-eks-best-practices/tree/master/policies/opa)中找到一组常见的OPA策略。

还有一个实验性的[OPA插件for CoreDNS](https://github.com/coredns/coredns-opa),允许您使用OPA过滤/控制CoreDNS返回的记录。

### Kyverno

[Kyverno](https://kyverno.io)是一个本地Kubernetes策略引擎,可以使用策略作为Kubernetes资源来验证、变异和生成配置。Kyverno使用Kustomize样式的覆盖进行验证,支持JSON补丁和策略合并补丁进行变异,并可以根据灵活的触发器在命名空间之间克隆资源。

您可以使用Kyverno隔离命名空间、执行pod安全性和其他最佳实践,并生成默认配置,如网络策略。GitHub [repository](https://github.com/aws/aws-eks-best-practices/tree/master/policies/kyverno)中包含了几个示例。Kyverno网站的[策略库](https://kyverno.io/policies/)中还包含了许多其他示例。

### 将租户工作负载隔离到特定节点

将租户工作负载限制在为各自租户配置的节点上可用于增强软多租户模型中的隔离。使用这种方法,租户特定的工作负载仅在为相应租户配置的节点上运行。为实现此隔离,使用本机Kubernetes属性(节点亲和性和污点与容忍)来针对特定节点进行pod调度,并防止来自其他租户的pod被调度到租户特定的节点上。

#### 第1部分 - 节点亲和性

Kubernetes [节点亲和性](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity)用于根据节点[标签](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)定位节点进行调度。使用节点亲和性规则,pod会被吸引到与选择器条件匹配的特定节点。在下面的pod规范中,应用了`requiredDuringSchedulingIgnoredDuringExecution`节点亲和性。结果是,pod将针对标有以下键/值的节点:`node-restriction.kubernetes.io/tenant: tenants-x`。

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

使用此节点亲和性,标签在调度期间是必需的,但在执行期间不是必需的;如果底层节点的标签发生变化,pod不会仅由于该标签变化而被驱逐。但是,未来的调度可能会受到影响。

!!! Warning
    `node-restriction.kubernetes.io/`前缀标签在Kubernetes中有特殊含义。[NodeRestriction](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#noderestriction)在EKS集群中启用,防止`kubelet`添加/删除/更新带有此前缀的标签。攻击者无法使用`kubelet`的凭据更新节点对象或修改系统设置以将这些标签传递给`kubelet`,因为`kubelet`不允许修改这些标签。如果此前缀用于所有pod到节点调度,它可以防止攻击者可能想要通过修改节点标签来吸引不同的工作负载到节点的场景。

!!! Info
    我们可以使用[节点选择器](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector)而不是节点亲和性。但是,节点亲和性更具表现力,允许在pod调度期间考虑更多条件。有关差异和更高级调度选择的更多信息,请参见CNCF博客文章[Advanced Kubernetes pod to node scheduling](https://www.cncf.io/blog/2021/07/27/advanced-kubernetes-pod-to-node-scheduling/)。

#### 第2部分 - 污点和容忍

吸引pod到节点只是这种三部分方法的第一部分。为了使这种方法奏效,我们必须阻止未经授权的pod被调度到节点上。为了阻止不需要或未经授权的pod,Kubernetes使用节点[污点](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)。污点用于在节点上设置条件,以防止pod被调度。下面的污点使用键值对`tenant: tenants-x`。

``` yaml
...
    taints:
      - key: tenant
        value: tenants-x
        effect: NoSchedule
...
```

给定上述节点`污点`,只有_容忍_污点的pod才会被允许调度到该节点上。为了允许授权的pod被调度到该节点,相应的pod规范必须包含对污点的`容忍`,如下所示。

``` yaml
...
  tolerations:
  - effect: NoSchedule
    key: tenant
    operator: Equal
    value: tenants-x
...
```

具有上述`容忍`的pod将不会因为该特定污点而被阻止调度到该节点上。污点也用于Kubernetes在某些条件下暂时停止pod调度,例如节点资源压力。通过节点亲和性和污点与容忍,我们可以有效地吸引所需的pod到特定节点并阻止不需要的pod。

!!! attention
    某些Kubernetes pod需要在所有节点上运行。这些pod的示例包括由[容器网络接口(CNI)](https://github.com/containernetworking/cni)和[kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/) [daemonsets](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)启动的pod。因此,这些pod的规范包含非常宽松的容忍,以容忍不同的污点。应小心不要更改这些容忍。更改这些容忍可能会导致集群操作不正确。此外,策略管理工具,如[OPA/Gatekeeper](https://github.com/open-policy-agent/gatekeeper)和[Kyverno](https://kyverno.io/)可用于编写验证策略,以防止未经授权的pod使用这些宽松的容忍。

#### 第3部分 - 基于策略的节点选择管理

有几种工具可用于帮助管理pod规范的节点亲和性和容忍,包括在CICD管道中执行规则。但是,隔离的执行也应该在Kubernetes集群级别完成。为此,可以使用策略管理工具来_变异_入站Kubernetes API服务器请求,基于请求有效负载应用上述提到的相应节点亲和性规则和容忍。

例如,目标为_tenants-x_命名空间的pod可以被_加上_正确的节点亲和性和容忍,以允许在_tenants-x_节点上进行调度。利用使用Kubernetes [Mutating Admission Webhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook)配置的策略管理工具,可以使用策略来变异入站pod规范。这些变异添加了所需的元素,以允许所需的调度。下面是一个添加节点亲和性的OPA/Gatekeeper策略示例。

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

上述策略应用于Kubernetes API服务器请求,以将pod应用于_tenants-x_命名空间。该策略添加了`requiredDuringSchedulingIgnoredDuringExecution`节点亲和性规则,以便pod被吸引到带有`tenant: tenants-x`标签的节点。

第二个策略(如下所示)使用与目标命名空间和组、种类和版本相同的匹配标准,将容忍添加到同一pod规范中。

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

上述策略特定于pod;这是由于策略的`location`元素中突变元素的路径。可以编写其他策略来处理创建pod的资源,如Deployment和Job资源。列出的策略和其他示例可以在此指南的伴随[GitHub项目](https://github.com/aws/aws-eks-best-practices/tree/master/policies/opa/gatekeeper/node-selector)中看到。

这两个变异的结果是,pod被吸引到所需的节点,同时也不会被特定节点污点所排斥。为了验证这一点,我们可以看到从两个`kubectl`调用获得的输出片段,一个用于获取标有`tenant=tenants-x`的节点,另一个用于获取`tenants-x`命名空间中的pod。

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

从上面的输出可以看出,所有pod都调度在标有`tenant=tenants-x`的节点上。简单地说,pod只会在所需的节点上运行,而其他pod(没有所需的亲和力和容忍)则不会。租户工作负载被有效隔离。

下面是一个变异的pod规范示例。

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
    使用变异和验证准入Webhook集成到Kubernetes API服务器请求流的策略管理工具旨在在指定的时间框内(通常为3秒或更短)响应API服务器的请求。如果Webhook调用未能在配置的时间内返回响应,则可能会发生或可能不会发生入站API服务器请求的变异和/或验证。这种行为取决于准入Webhook配置是设置为[Fail Open还是Fail Close](https://open-policy-agent.github.io/gatekeeper/website/docs/#admission-webhook-fail-open-by-default)。

在上面的示例中,我们使用了为OPA/Gatekeeper编写的策略。但是,也有其他策略管理工具可以处理我们的节点选择用例。例如,这个[Kyverno策略](https://kyverno.io/policies/other/add_node_affinity/add_node_affinity/)可用于处理节点亲和性变异。

!!! tip
    如果操作正确,变异策略将对入站API服务器请求有效负载进行所需的更改。但是,也应该包括验证策略来验证是否发生了所需的更改,然后才允许更改持久化。这在将这些策略用于租户到节点隔离时特别重要。定期检查集群是否存在不需要的配置的审核策略也是一个好主意。

### 参考

- [k-rail](https://github.com/cruise-automation/k-rail)旨在帮助您通过执行某些策略来保护多租户环境。

- [使用Amazon EKS的多租户SaaS应用程序的安全实践](https://d1.awsstatic.com/whitepapers/security-practices-for-multi-tenant-saas-apps-using-eks.pdf)

## 硬多租户

通过为每个租户配置单独的集群来实现硬多租户。虽然这提供了非常强大的租户隔离,但也有几个缺点。

首先,当您有许多租户时,这种方法可能会变得非常昂贵。您不仅需要为每个集群的控制平面成本付费,而且无法在集群之间共享计算资源。这最终会导致碎片化,其中一些集群利用不足,而其他集群过度利用。

其次,您可能需要购买或构建特殊工具来管理所有这些集群。随着时间的推移,管理数百或数千个集群可能会变得太过繁琐。

最后,每个租户创建一个集群的速度相对于创建一个命名空间要慢。尽管如此,在高度监管的行业或需要强隔离的SaaS环境中,硬租户方法可能是必要的。

## 未来方向

Kubernetes社区已经认识到软多租户的当前缺陷以及硬多租户的挑战。[多租户特殊兴趣小组(SIG)](https://github.com/kubernetes-sigs/multi-tenancy)正试图通过几个孵化项目解决这些缺陷,包括分层命名空间控制器(HNC)和虚拟集群。

HNC提案(KEP)描述了一种创建命名空间父子关系的方式,并沿着这种关系继承[策略]对象,同时允许租户管理员创建子命名空间。

虚拟集群提案描述了一种在集群内为每个租户创建单独的控制平面服务实例(包括API服务器、控制器管理器和调度程序)的机制(也称为"Kubernetes on Kubernetes")。

[多租户基准](https://github.com/kubernetes-sigs/multi-tenancy/blob/master/benchmarks/README.md)提案提供了使用命名空间进行隔离和分段的指南,以及一个命令行工具[kubectl-mtb](https://github.com/kubernetes-sigs/multi-tenancy/blob/master/benchmarks/kubectl-mtb/README.md)来验证是否符合这些指南。

## 多集群管理工具和资源

- [Banzai Cloud](https://banzaicloud.com/)
- [Kommander](https://d2iq.com/solutions/ksphere/kommander)
- [Lens](https://github.com/lensapp/lens)
- [Nirmata](https://nirmata.com)
- [Rafay](https://rafay.co/)
- [Rancher](https://rancher.com/products/rancher/)
- [Weave Flux](https://www.weave.works/oss/flux/)