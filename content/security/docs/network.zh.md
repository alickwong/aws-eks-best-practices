网络安全

网络安全有几个方面。第一个涉及应用限制网络流量在服务之间流动的规则。第二个涉及在传输过程中对流量进行加密。在EKS上实施这些安全措施的机制各不相同,但通常包括以下内容:

## 流量控制

- 网络策略
- 安全组

## 网络加密

- 服务网格
- 容器网络接口(CNIs)
- 入口控制器和负载均衡器
- Nitro实例
- 带有cert-manager的ACM私有CA

## 网络策略

在Kubernetes集群中,默认情况下允许所有Pod到Pod的通信。虽然这种灵活性可能有助于促进实验,但它并不被认为是安全的。Kubernetes网络策略为您提供了一种机制,用于限制Pod之间(通常称为东/西向流量)以及Pod和外部服务之间的网络流量。Kubernetes网络策略在OSI模型的第3层和第4层运行。网络策略使用pod、命名空间选择器和标签来识别源和目标pod,但也可以包括IP地址、端口号、协议或这些的组合。网络策略可以应用于pod的入站或出站连接,通常称为入口和出口规则。

通过Amazon VPC CNI插件的原生网络策略支持,您可以实施网络策略来保护Kubernetes集群中的网络流量。这与上游Kubernetes网络策略API集成,确保兼容性和遵守Kubernetes标准。您可以使用上游API支持的不同[标识符](https://kubernetes.io/docs/concepts/services-networking/network-policies/)定义策略。默认情况下,允许所有入口和出口流量到pod。当指定了带有policyType Ingress的网络策略时,只允许从pod的节点和入口规则允许的连接进入pod。出口规则也是如此。如果定义了多个规则,则在做出决策时会考虑所有规则的并集。因此,评估顺序不会影响策略结果。

!!! attention
    当您首次配置EKS集群时,VPC CNI网络策略功能默认情况下是禁用的。确保您部署了受支持的VPC CNI插件版本,并在vpc-cni插件上将`ENABLE_NETWORK_POLICY`标志设置为`true`以启用此功能。请参考[Amazon EKS用户指南](https://docs.aws.amazon.com/eks/latest/userguide/managing-vpc-cni.html)了解详细说明。

## 建议

### 开始使用网络策略 - 遵循最小权限原则

#### 创建默认拒绝策略

与RBAC策略一样,建议遵循最小权限访问原则来使用网络策略。首先创建一个拒绝所有策略,限制命名空间内的所有入站和出站流量。

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

![default-deny](./images/default-deny.jpg)

!!! tip
    上图由[Tufin](https://orca.tufin.io/netpol/)的网络策略查看器创建。

#### 创建允许DNS查询的规则

在设置默认拒绝所有规则后,您可以开始添加其他规则,例如允许pod查询CoreDNS进行名称解析的规则。

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-access
  namespace: default
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

![allow-dns-access](./images/allow-dns-access.jpg)

#### 逐步添加规则,选择性地允许命名空间/pod之间的流量

了解应用程序的要求,并根据需要创建细粒度的入口和出口规则。下面的示例展示了如何限制端口80上从`client-one`到`app-one`的入口流量。这有助于最小化攻击面并降低未经授权访问的风险。

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-app-one
  namespace: default
spec:
  podSelector:
    matchLabels:
      k8s-app: app-one
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          k8s-app: client-one
    ports:
    - protocol: TCP
      port: 80
```

![allow-ingress-app-one](./images/allow-ingress-app-one.png)

### 监控网络策略执行

- **使用网络策略编辑器**
  - [网络策略编辑器](https://networkpolicy.io/)提供可视化、安全评分、从网络流量日志自动生成
  - 以交互方式构建网络策略
- **审核日志**
  - 定期审查EKS集群的审核日志
  - 审核日志提供大量有关在集群上执行的操作的信息,包括对网络策略的更改
  - 使用这些信息跟踪随时间的网络策略变化,并检测任何未经授权或意外的变化
- **自动化测试**
  - 实施自动化测试,创建一个与生产环境相似的测试环境,并定期部署试图违反网络策略的工作负载。
- **监控指标**
  - 配置您的可观察性代理,以刮取VPC CNI节点代理的Prometheus指标,这允许监控代理健康状况和SDK错误。
- **定期审核网络策略**
  - 定期审核您的网络策略,以确保它们满足当前的应用程序要求。随着您的应用程序的发展,审核可以让您有机会删除冗余的入口和出口规则,并确保您的应用程序没有过多的权限。
- **使用Open Policy Agent(OPA)确保网络策略存在**
  - 使用如下所示的OPA策略,确保在引入应用程序pod之前网络策略始终存在。如果没有相应的网络策略,此策略将拒绝带有标签`k8s-app: sample-app`的k8s pod的引入。

```javascript
package kubernetes.admission
import data.kubernetes.networkpolicies

deny[msg] {
    input.request.kind.kind == "Pod"
    pod_label_value := {v["k8s-app"] | v := input.request.object.metadata.labels}
    contains_label(pod_label_value, "sample-app")
    np_label_value := {v["k8s-app"] | v := networkpolicies[_].spec.podSelector.matchLabels}
    not contains_label(np_label_value, "sample-app")
    msg:= sprintf("The Pod %v could not be created because it is missing an associated Network Policy.", [input.request.object.metadata.name])
}
contains_label(arr, val) {
    arr[_] == val
}
```

### 故障排查

#### 监控vpc-network-policy-controller、node-agent日志

启用EKS控制平面控制器管理器日志以诊断网络策略功能。您可以将控制平面日志流式传输到CloudWatch日志组,并使用[CloudWatch日志洞察](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html)执行高级查询。从日志中,您可以查看哪些pod端点对象被解析为网络策略,策略的协调状态,以及调试策略是否按预期工作。

此外,Amazon VPC CNI允许您从EKS工作节点启用策略执行日志的收集和导出到[Amazon Cloudwatch](https://aws.amazon.com/cloudwatch/)。启用后,您可以利用[CloudWatch容器洞察](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights.html)提供有关网络策略使用情况的见解。

Amazon VPC CNI还提供了一个SDK,提供了一个与节点上的eBPF程序交互的接口。在部署`aws-node`时会安装SDK。您可以在节点上的`/opt/cni/bin`目录下找到SDK二进制文件。在启动时,SDK提供对基本功能的支持,如检查eBPF程序和映射。

```shell
sudo /opt/cni/bin/aws-eks-na-cli ebpf progs
```

#### 记录网络流量元数据

[AWS VPC流日志](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html)捕获通过VPC流动的流量的元数据,如源和目标IP地址和端口以及接受/丢弃的数据包。这些信息可以被分析,以查找VPC内部资源之间的可疑或异常活动。但是,由于pod的IP地址经常随着它们被替换而改变,流日志本身可能不足以解决这个问题。Calico Enterprise通过pod标签和其他元数据扩展了流日志,使得解读pod之间的流量更加容易。

## 安全组

EKS使用[AWS VPC安全组](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html)(SGs)来控制Kubernetes控制平面和集群工作节点之间的流量。安全组也用于控制工作节点、其他VPC资源和外部IP地址之间的流量。当您配置一个EKS集群(Kubernetes版本1.14-eks.3或更高版本)时,会自动为您创建一个集群安全组。这个安全组允许EKS控制平面与托管节点组之间的无限制通信。为了简单起见,建议您将集群SG添加到所有节点组,包括非托管节点组。

在Kubernetes版本1.14和EKS版本eks.3之前,EKS控制平面和节点组配置了单独的安全组。控制平面和节点组安全组的最小和建议规则可以在[https://docs.aws.amazon.com/eks/latest/userguide/sec-group-reqs.html](https://docs.aws.amazon.com/eks/latest/userguide/sec-group-reqs.html)找到。_控制平面安全组_的最小规则允许端口443的入站流量来自工作节点SG。这个规则是允许kubelet与Kubernetes API服务器通信的。它还包括端口10250的出站流量到工作节点SG;10250是kubelet监听的端口。类似地,_节点组_的最小规则允许端口10250的入站流量来自控制平面SG,以及端口443的出站流量到控制平面SG。最后,还有一条允许节点组内部节点之间无限制通信的规则。

如果您需要控制在集群内运行并为外部服务(如RDS数据库)提供服务的服务之间的通信,请考虑[pod的安全组](https://docs.aws.amazon.com/eks/latest/userguide/security-groups-for-pods.html)。使用pod的安全组,您可以将现有的安全组分配给一组pod。

!!! warning
    如果在创建pod之前引用一个不存在的安全组,pod将无法调度。

您可以通过创建一个`SecurityGroupPolicy`对象并指定一个`PodSelector`或`ServiceAccountSelector`来控制哪些pod被分配到一个安全组。将选择器设置为`{}`将把`SecurityGroupPolicy`中引用的SG分配给某个命名空间中的所有pod或所有服务帐户。在实现pod的安全组之前,请务必熟悉所有[注意事项](https://docs.aws.amazon.com/eks/latest/userguide/security-groups-for-pods.html#security-groups-pods-considerations)。

!!! important
    如果您使用pod的SG,**必须**创建允许端口53出站到集群安全组的SG。同样,**必须**更新集群安全组,以接受来自pod安全组的端口53入站流量。

!!! important
    [安全组的限制](https://docs.aws.amazon.com/vpc/latest/userguide/amazon-vpc-limits.html#vpc-limits-security-groups)仍然适用于pod的安全组,所以要谨慎使用。

!!! important
    您**必须**为pod配置的所有探测创建来自集群安全组(kubelet)的入站流量规则。

!!! important
    pod的安全组依赖于一个称为[ENI trunking](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/container-instance-eni.html)的功能,该功能是为了增加EC2实例的ENI密度。当一个pod被分配到一个SG时,VPC控制器会将一个来自节点组的分支ENI与该pod关联。如果在pod被调度时节点组中没有足够的分支ENI可用,pod将保持pending状态。一个实例可以支持的分支ENI数量因实例类型/系列而异。请参见[https://docs.aws.amazon.com/eks/latest/userguide/security-groups-for-pods.html#supported-instance-types](https://docs.aws.amazon.com/eks/latest/userguide/security-groups-for-pods.html#supported-instance-types)了解更多详情。

虽然pod的安全组提供了一种AWS原生的方式来控制集群内部和外部的网络流量,而无需依赖策略守护进程,但也有其他选择。例如,Cilium策略引擎允许您在网络策略中引用DNS名称。Calico Enterprise包括一个将网络策略映射到AWS安全组的选项。如果您实现了像Istio这样的服务网格,您可以使用出口网关来限制网络出口到特定的完全限定域名或IP地址。有关此选项的更多信息,请阅读关于[Istio中出口流量控制](https://istio.io/blog/2019/egress-traffic-control-in-istio-part-1/)的三篇系列文章。

## 何时使用网络策略vs pod的安全组?

### 何时使用Kubernetes网络策略

- **控制pod到pod的流量**
  - 适用于控制集群内pod之间的网络流量(东西向流量)
- **在IP地址或端口级别控制流量(OSI第3层或第4层)**

### 何时使用AWS pod的安全组(SGP)

- **利用现有的AWS配置**
  - 如果您已经有一组复杂的EC2安全组来管理对AWS服务的访问,并且正在将应用程序从EC2实例迁移到EKS,SGPs可能是一个非常好的选择,允许您重用安全组资源并将其应用于您的pod。
- **控制对AWS服务的访问**
  - 您在EKS集群中运行的应用程序想要与其他AWS服务(RDS数据库)进行通信,使用SGPs作为一种有效的机制来控制从pod到AWS服务的流量。
- **隔离pod和节点流量**
  - 如果您想完全分离pod流量和其他节点流量,请在`POD_SECURITY_GROUP_ENFORCING_MODE=strict`模式下使用SGP。

### 使用`pod的安全组`和`网络策略`的最佳实践

- **分层安全**
  - 结合使用SGP和Kubernetes网络策略,采取分层安全方法
  - 使用SGPs来限制对不属于集群的AWS服务的网络级访问,而Kubernetes网络策略可以限制集群内pod之间的网络流量
- **最小权限原则**
  - 只允许pod或命名空间之间必要的流量
- **细分您的应用程序**
  - 尽可能按网络策略细分应用程序,以减少应用程序被入侵时的损害范围
- **保持策略简单明了**
  - Kubernetes网络策略可能非常细粒度和复杂,最好保持尽可能简单,以减少配置错误的风险并降低管理开销
- **减少攻击面**
  - 通过限制应用程序的暴露来最小化攻击面

!!! attention
    pod的安全组提供了两种执行模式:`strict`和`standard`。当在EKS集群中同时使用网络策略和pod的安全组功能时,您必须使用`standard`模式。

在网络安全方面,分层方法通常是最有效的解决方案。结合使用Kubernetes网络策略和SGP可以为您在EKS中运行的应用程序提供一个强大的纵深防御策略。

## 服务网格策略执行或Kubernetes网络策略

`服务网格`是一个专门的基础设施层,您可以将其添加到应用程序中。它允许您透明地添加诸如可观察性、流量管理和安全性等功能,而无需将它们添加到您自己的代码中。

服务网格在OSI模型的第7层(应用程序)执行策略,而Kubernetes网络策略在第3层(网络)和第4层(传输)运行。这个领域有很多产品,如AWS AppMesh、Istio、Linkerd等。

### 何时使用服务网格进行策略执行

- 已经投资了服务网格
- 需要更高级的功能,如流量管理、可观察性和安全性
  - 流量控制、负载均衡、断路、速率限制、超时等
  - 详细了解您的服务性能(延迟、错误率、每秒请求数、请求量等)
  - 您想实现并利用服务网格的安全功能,如mTLS

### 选择Kubernetes网络策略用于更简单的用例

- 限制哪些pod可以相互通信
- 网络策略需要的资源比服务网格少,这使它们成为较小集群或更简单用例的良好选择

!!! tip
    网络策略和服务网格也可以一起使用。使用网络策略提供基本的安全性和隔离,然后使用服务网格添加流量管理、可观察性和安全性等其他功能。

## 第三方网络策略引擎

当您有高级策略需求,如全局网络策略、支持基于DNS主机名的规则、第7层规则、基于服务帐户的规则以及显式拒绝/日志操作等时,请考虑使用第三方网络策略引擎。[Calico](https://docs.projectcalico.org/introduction/)是来自[Tigera](https://tigera.io)的开源策略引擎,可以很好地与EKS配合使用。除了实现完整的Kubernetes网络策略功能外,Calico还支持扩展的网络策略,具有更丰富的功能,包括在与Istio集成时支持第7层规则,如HTTP。Calico策略可以针对命名空间、pod或服务帐户进行范围界定。当策略的范围限定为服务帐户时,它会将一组入口/出口规则与该服务帐户关联。在适当的RBAC规则位置,您可以防止团队覆盖这些规则,允许IT安全专业人员安全地委托管理命名空间。Isovalent(Cilium的维护者)也扩展了网络策略,包括对第7层规则(如HTTP)的部分支持。Cilium还支持DNS主机名,这对于限制Kubernetes服务/pod与VPC内部或外部运行的资源之间的流量很有用。相比之下,Calico Enterprise包括一个允许您将Kubernetes网络策略映射到AWS安全组的功能,以及DNS主机名。

您可以在[https://github.com/ahmetb/kubernetes-network-policy-recipes](https://github.com/ahmetb/kubernetes-network-policy-recipes)找到常见的Kubernetes网络策略列表。Calico的类似规则集可在[https://docs.projectcalico.org/security/calico-network-policy](https://docs.projectcalico.org/security/calico-network-policy)获得。

### 迁移到Amazon VPC CNI网络策略引擎

为了保持一致性并避免意外的pod通信行为,建议在集群中只部署一个网络策略引擎。如果您想从第三方迁移到VPC CNI网络策略引擎,我们建议在启用VPC CNI网络策略支持之前,将现有的第三方NetworkPolicy CRD转换为Kubernetes NetworkPolicy资源。并在应用于生产环境之前,在单独的测试集群中测试迁移后的策略。这样可以帮助您识别和解决pod通信行为中的任何潜在问题或不一致。

#### 迁移工具

为了协助您的迁移过程,我们开发了一个名为[K8s Network Policy Migrator](https://github.com/awslabs/k8s-network-policy-migrator)的工具,可以将您现有的Calico/Cilium网络策略CRD转换为Kubernetes原生网络策略。转换后,您可以直接在运行VPC CNI网络策略控制器的新集群上测试转换后的网络策略。该工具旨在帮助您简化迁移过程,确保顺利过渡。

!!! Important
    迁移工具只会转换与原生Kubernetes网络策略API兼容的第三方策略。如果您使用第三方插件提供的高级网络策略功能,迁移工具将跳过并报告它们。

请注意,迁移工具目前不受AWS VPC CNI网络策略工程团队支持,它是在尽力而为的基础上提供给客户使用的。我们鼓励您利用这个工具来促进您的迁移过程。如果您遇到任何问题或错误,我们诚挚地请您在[GitHub问题](https://github.com/awslabs/k8s-network-policy-migrator/issues)中创建一个问题。您的反馈对我们很宝贵,将有助于我们不断改进我们的服务。

### 其他资源

- [Kubernetes & Tigera: Network Policies, Security, and Audit](https://youtu.be/lEY2WnRHYpg)
- [Calico Enterprise](https://www.tigera.io/tigera-products/calico-enterprise/)
- [Cilium](https://cilium.readthedocs.io/en/stable/intro/)
- [NetworkPolicy编辑器](https://cilium.io/blog/2021/02/10/network-policy-editor)来自Cilium的交互式策略编辑器
- [Inspektor Gadget advise network-policy gadget](https://www.inspektor-gadget.io/docs/latest/gadgets/advise/network-policy/)根据网络流量分析建议网络策略

## 传输中的加密

需要符合PCI、HIPAA或其他法规的应用程序可能需要在传输过程中加密数据。如今,TLS是加密网络流量的事实标准。TLS就像它的前身SSL一样,提供了使用加密协议在网络上进行安全通信。TLS使用对称加密,其中用于加密数据的密钥是基于在会话开始时协商的共享秘密生成的。以下是一些在Kubernetes环境中加密数据的方法。

### Nitro实例

在以下Nitro实例类型之间交换的流量,如C5n、G4、I3en、M5dn、M5n、P3dn、R5dn和R5n,默认情况下是自动加密的。当有中间跳转,如传输网关或负载均衡器,流量就不会被加密。有关传输中的加密以及支持网络加密的完整实例类型列表,请参见[传输中的加密](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/data-protection.html#encryption-transit)。

### 容器网络接口(CNIs)

[WeaveNet](https://www.weave.works/oss/net/)可以配置为使用NaCl加密自动加密所有sleeve流量,并使用IPsec ESP加密快速数据路径流量。

### 服务网格

使用服务网格,如App Mesh、Linkerd v2和Istio,也可以实现传输中的加密。AppMesh支持使用X.509证书或Envoy的Secret Discovery Service(SDS)的[mTLS](https://docs.aws.amazon.com/app-mesh/latest/userguide/mutual-tls.html)。Linkerd和Istio都支持mTLS。

[aws-app-mesh-examples](https://github.com/aws/aws-app-mesh-examples)GitHub存储库提供了使用X.509证书和SPIRE作为SDS提供程序配置mTLS的演练:

- [使用X.509证书配置mTLS](https://github.com/aws/aws-app-mesh-examples/tree/main/walkthroughs/howto-k8s-mtls-file-based)
- [使用SPIRE(SDS)配置TLS](https://github.com/aws/aws-app-mesh-examples/tree/main/walkthroughs/howto-k8s-mtls-sds-based)

App Mesh还支持使用[AWS Certificate Manager](https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html)(ACM)颁发的私有证书或存储在虚拟节点本地文件系统上的证书的[TLS加密](https://docs.aws.amazon.com/app-mesh/latest/userguide/virtual-node-tls.html)。

[aws-app-mesh-examples](https://github.com/aws/aws-app-mesh-examples)GitHub存储库提供了使用ACM颁发的证书和打包在Envoy容器中的证书配置TLS的演练:

- [使用文件提供的TLS证书配置TLS](https://github.com/aws/aws-app-mesh-examples/tree/master/walkthroughs/howto-tls-file-provided)
- [使用AWS Certificate Manager配置TLS](https://github.com/aws/aws-app-mesh-examples/tree/master/walkthroughs/tls-with-acm)

### 入口控制器和负载均衡器

入口控制器是一种让您智能地路由来自集群外部的HTTP/S流量到集群内部服务的方式。这些入口通常由第4层负载均衡器(如经典负载均衡器或网络负载均衡器(NLB))提供前端。加密的流量可以在网络中的不同位置终止,例如在负载均衡器、入口资源或pod处。您终止SSL连接的方式和位置最终将由您组织的网络安全政策决定。例如,如果您有一个要求端到端加密的政策,您将不得不在pod处解密流量。这将给您的pod带来额外的负担,因为它将不得不花费周期来建立初始握手。总的来说,SSL/TLS处理是非常CPU密集的。因此,如果您有灵活性,请尝试在入口或负载均衡器处执行SSL卸载。

#### 与AWS弹性负载均衡器一起使用加密

[AWS应用程序负载均衡器](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html)(ALB)和[网络负载均衡器](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html)(NLB)都支持传输加密(SSL和TLS)。ALB的`alb.ingress.kubernetes.io/certificate-arn`注解允许您指定要添加到ALB的证书。如果省略注解,控制器将尝试使用匹配主机字段的可用[AWS Certificate Manager (ACM)](https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html)证书添加证书到需要它的侦听器。从EKS v1.15开始,您可以使用如下示例中所示的`service.beta.kubernetes.io/aws-load-balancer-ssl-cert`注解与NLB一起使用。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-app
  namespace: default
  labels:
    app: demo-app
  annotations:
     service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
     service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "<certificate ARN>"
     service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
     service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
spec:
  type: LoadBalancer
  ports:
  - port: 443
    targetPort: 80
    protocol: TCP
  selector:
    app: demo-app
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nginx
  namespace: default
  labels:
    app: demo-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
        - name: nginx
          image: nginx
          ports:
            - containerPort: 443
              protocol: TCP
            - containerPort: 80
              protocol: TCP
```

以下是SSL/TLS终止的其他示例。

- [使用Contour和Let's Encrypt以GitOps方式保护EKS入口](https://aws.amazon.com/blogs/containers/securing-eks-ingress-contour-lets-encrypt-gitops/)
- [如何使用ACM终止EKS工作负载上的HTTPS流量?](https://aws.amazon.com/premiumsupport/knowledge-center/terminate-https-traffic-eks-acm/)

!!! attention
    一些入口,如AWS LB控制器,使用注解而不是作为入口规范的一部分来实现SSL/TLS。

### ACM私有CA与cert-manager

您可以使用ACM私有证书颁发机构(CA)和[cert-manager](https://cert-manager.io/)(一个流行的Kubernetes插件,用于分发、续订和撤销证书)来启用入口、pod和pod之间的TLS和mTLS,以保护您的EKS应用程序工作负载。ACM私有CA是一个高可用、安全的托管CA,无需承担管理自己的CA的前期和维护成本。如果您使用默认的Kubernetes证书颁发机构,有机会通过ACM私有CA来提高安全性并满足合规性要求。与默认CA将密钥存储在内存中(较不安全)相比,ACM私有CA在FIPS 140-2 Level 3硬件安全模块(非常安全)中保护私钥。集中式CA还可以让您更好地控制和审核私有证书,无论是在Kubernetes环境内还是外部。

#### 用于工作负载之间的相互TLS的短期CA模式

在EKS中使用ACM私有CA进行mTLS时,建议使用_短期CA模式_颁发短期证书。虽然在通用CA模式下也可以颁发短期证书,但使用短期CA模式更加经济高效(比通用模式便宜约75%)。此外,您应该尽量将私有证书的有效期与EKS集群中pod的生命周期保持一致。[在此了解更多关于ACM私有CA及其优势的信息](https://aws.amazon.com/certificate-manager/private-certificate-authority/)。

#### ACM设置说明

首先,按照[ACM私有CA技术文档](https://docs.aws.amazon.com/acm-pca/latest/userguide/create-CA.html)中提供的步骤创建一个私有CA。创建私有CA后,按照[常规安装说明](https://cert-manager.io/docs/installation/)安装cert-manager。安装cert-manager后,按照[GitHub上的设置说明](https://github.com/cert-manager/aws-privateca-issuer#setup)安装私有CA Kubernetes cert-manager插件。该插件允许cert-manager向ACM私有CA请求私有证书。

现在您已经有了一个私有CA和一个配备了cert-manager和插件的EKS集群,是时候设置权限并创建issuer了。使用以下IAM权限更新EKS节点角色,以允许访问ACM私有CA。将`<CA_ARN>`替换为您的私有CA值:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "awspcaissuer",
            "Action": [
                "acm-pca:DescribeCertificateAuthority",
                "acm-pca:GetCertificate",
                "acm-pca:IssueCertificate"
            ],
            "Effect": "Allow",
            "Resource": "<CA_ARN>"
        }
    ]
}
```

[IAM帐户的服务角色(IRSA)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)也可以使用。请参阅下面的其他资源部分以获取完整的示例。

创建一个名为cluster-issuer.yaml的自定义资源定义文件,其中包含以下文本,并将`<CA_ARN>`和`<Region>`信息替换为您的私有CA信息。

```yaml
apiVersion: awspca.cert-manager.io/v1beta1
kind: AWSPCAClusterIssuer
metadata:
          name: demo-test-root-ca
spec:
          arn: <CA_ARN>
          region: <Region>
```

部署您创建的Issuer。

```bash
kubectl apply -f cluster-issuer.yaml
```

您的EKS集群现已配置为从私有CA请求证书。您现在可以使用cert-manager的`Certificate`资源来发出证书,方法是将`issuerRef`字段的值更改为您创建的私有CA Issuer。有关如何指定和请求Certificate资源的更多详细信息,请查看cert-manager的[Certificate资源指南](https://cert-manager.io/docs/usage/certificate/)。[在此查看示例](https://github.com/cert-manager/aws-privateca-issuer/tree/main/config/samples/)。

### 带有Istio和cert-manager的ACM私有CA

如果您在EKS集群中运行Istio,您可以禁用Istio控制平面(特别是`istiod`)作为根证书颁发机构(CA),并将ACM私有CA配置为工作负载之间mTLS的根CA。如果您选择这种方法,请考虑在ACM私有CA中使用_短期CA模式_。请参阅[前一节](#short-lived-ca-mode-for-mutual-tls-between-workloads)和这篇[博客文章](https://aws.amazon.com/blogs/security/how-to-use-aws-private-certificate-authority-short-lived-certificate-mode)了解更多详细信息。

#### Istio中的证书签名工作原理(默认)

Kubernetes中的工作负载使用服务帐户进行标识。如果您没有指定服务帐户,Kubernetes将自动为您的工作负载分配一个。服务帐户还会自动挂载一个关联的令牌。此令牌由工作负载用于对Kubernetes API进行身份验证。服务帐户可能足以作为Kubernetes中工作负载的身份,但Istio有自己的身份管理系统和CA。

当一个带有envoy边车代理的工作负载启动时,它需要从Istio获得一个标识,以便被视为可信并被允许与网格中的其他服务进行通信。为此,`istio-agent`向Istio控制平面发送一个称为证书签名请求(或CSR)的请求。此CSR包含服务帐户令牌,以便在处理之前验证工作负载的身份。此验证过程由`istiod`处理,`istiod`充当注册机构(RA)和CA。RA充当网关,确保只有经过验证的CSR才能进入CA。一旦CSR得到验证,它将被转发到CA,CA将颁发一个包含[SPIFFE](https://spiffe.io/)身份的证书,称为SPIFFE可验证身份文件(或SVID)。此SVID被分配给请求服务,用于标识和加密与其他服务之间的传输流量。

![Istio默认证书签名请求流程](./images/default-istio-csr-flow.png)

#### Istio中的证书签名工作原理与ACM私有CA

您可以使用一个名为Istio Certificate Signing Request agent([istio-csr](https://cert-manager.io/docs/projects/istio-csr/))的cert-manager插件将Istio与ACM私有CA集成。此代理允许Istio工作负载和控制平面组件使用cert-manager发行人(在本例中为ACM私有CA)进行安全。_istio-csr_代理公开了与_istiod_在默认配置中提供的相同的服务,用于验证传入的CSR。但是,在验证之后,它将请求转换为cert-manager支持的资源(即与外部CA发行人的集成)。

每当有来自工作负载的CSR时,它都会被转发到_istio-csr_,_istio-csr_将从ACM私有CA请求证书。_istio-csr_与ACM私有CA之间的这种通信是通过[AWS Private CA发行人插件](https://github.com/cert-manager/aws-privateca-issuer)启用的。cert-manager使用此插件向ACM私有CA请求TLS证书。发行人插件将与ACM私有CA服务进行通信,请求为工作负载签署证书。一旦证书被签署,它将被返回给_istio-csr_,_istio-csr_将读取签名的请求,并将其返回给发起CSR的工作负载。

![带有istio-csr的Istio证书签名请求流程](./images/istio-csr-with-acm-private-ca.png)

#### Istio与私有CA的设置说明

1. 首先,按照本节中的[ACM私有CA与cert-manager](#acm-private-ca-with-cert-manager)部分的说明完成以下步骤:
2. 创建一个私有CA
3. 安装cert-manager
4. 安装发行人插件
5. 设置权限并创建一个issuer。该issuer代表CA,用于签署`istiod`和网格工作负载证书。它将与ACM私有CA进行通信。
6. 创建一个`istio-system`命名空间。这是部署`istiod证书`和其他Istio资源的位置。
7. 安装配置有AWS私有CA发行人插件的Istio CSR。您可以保留工作负载的证书签名请求,以验证它们是否得到批准和签名(`preserveCertificateRequests=true`)。

    ```bash
    helm install -n cert-manager cert-manager-istio-csr jetstack/cert-manager-istio-csr \
    --set "app.certmanager.issuer.group=awspca.cert-manager.io" \
    --set "app.certmanager.issuer.kind=AWSPCAClusterIssuer" \
    --set "app.certmanager.issuer.name=<the-name-of-the-issuer-you-created>" \
    --set "app.certmanager.preserveCertificateRequests=true" \
    --set "app.server.maxCertificateDuration=48h" \
    --set "app.tls.certificateDuration=24h" \
    --set "app.tls.istiodCertificateDuration=24h" \
    --set "app.tls.rootCAFile=/var/run/secrets/istio-csr/ca.pem" \
    --set "volumeMounts[0].name=root-ca" \
    --set "volumeMounts[0].mountPath=/var/run/secrets/istio-csr" \
    --set "volumes[0].name=root-ca" \
    --set "volumes[0].secret.secretName=istio-root-ca"
    ```

8. 使用自定义配置安装Istio,以将`istiod`替换为`cert-manager istio-csr`作为网格的证书提供程序。此过程可以使用[Istio Operator](https://tetrate.io/blog/what-is-istio-operator/)来完成。

    ```yaml
    apiVersion: install.istio.io/v1alpha1
    kind: IstioOperator
    metadata:
      name: istio
      namespace: istio-system
    spec:
      profile: "demo"
      hub: gcr.io/istio-release
      values:
      global:
        # Change certificate provider to cert-manager istio agent for istio agent
        caAddress: cert-manager-istio-csr.cert-manager.svc:443
      components:
        pilot:
          k8s:
            env:
              # Disable istiod CA Sever functionality
            - name: ENABLE_CA_SERVER
              value: "false"
            overlays:
            - apiVersion: apps/v1
              kind: Deployment
              name: istiod
              patches:

                # Mount istiod serving and webhook certificate from Secret mount
              - path: spec.template.spec.containers.[name:discovery].args[7]
                value: "--tlsCertFile=/etc/cert-manager/tls/tls.crt"
              - path: spec.template.spec.containers.[name:discovery].args[8]
                value: "--tlsKeyFile=/etc/cert-manager/tls/tls.key"
              - path: spec.template.spec.containers.[name:discovery].args[9]
                value: "--caCertFile=/etc/cert-manager/ca/root-cert.pem"

              - path: spec.template.spec.containers.[name:discovery].volumeMounts[6]
                value:
                  name: cert-manager
                  mountPath: "/etc/cert-manager/tls"
                  readOnly: true
              - path: spec.template.spec.containers.[name:discovery].volumeMounts[7]
                value:
                  name: ca-root-cert
                  mountPath: "/etc/cert-manager/ca"
                  readOnly: true

              - path: spec.template.spec.volumes[6]
                value:
                  name: cert-manager
                  secret:
                    secretName: istiod-tls
              - path: spec.template.spec.volumes[7]
                value:
                  name: ca-root-cert
                  configMap:
                    defaultMode: 420
                    name: istio-ca-root-cert
    ```

9. 部署您创建的自定义资源。

    ```bash
    istioctl operator init
    kubectl apply -f istio-custom-config.yaml
    ```

10. 现在您可以在EKS集群中部署一个工作负载到网格,并[强制执行mTLS](https://istio.io/latest/docs/reference/config/security/peer_authentication/)。

![Istio证书签名请求](./images/istio-csr-requests.png)

## 工具和资源

- [Amazon EKS安全沉浸式研讨会 - 网络安全](https://catalog.workshops.aws/eks-security-immersionday/en-US/6-network-security)
- [如何实现cert-manager和ACM私有CA插件,在EKS上启用TLS](https://aws.amazon.com/blogs/security/tls-enabled-kubernetes-clusters-with-acm-private-ca-and-amazon-eks-2/)。
- [使用新的AWS Load Balancer控制器和ACM私有CA在Amazon EKS上设置端到端TLS加密](https://aws.amazon.com/blogs/containers/setting-up-end-to-end-tls-encryption-on-amazon-eks-with-the-new-aws-load-balancer-controller/)。
- [私有CA Kubernetes cert-manager插件在GitHub上](https://github.com/cert-manager/aws-privateca-issuer)。
- [私有CA Kubernetes cert-manager插件用户指南](https://docs.aws.amazon.com/acm-pca/latest/userguide/PcaKubernetes.html)。
- [如何使用AWS私有证书颁发机构短期证书模式](https://aws.amazon.com/blogs/security/how-to-use-aws-private-certificate-authority-short-lived-certificate-mode)
- [使用ksniff和Wireshark验证Kubernetes中的服务网格TLS](https://itnext.io/verifying-service-mesh-tls-in-kubernetes-using-ksniff-and-wireshark-2e993b26bf95)
- [ksniff](https://github.com/eldadru/ksniff)
- [egress-operator](https://github.com/monzo/egress-operator) 一个操作符和DNS插件,用于控制集群外的出口流量,而无需进行协议检查
- [SUSE的NeuVector](https://www.suse.com/neuvector/) 开源的零信任容器安全平台,提供策略网络规则、数据丢失预防(DLP)、Web应用防火墙(WAF)和网络威胁签名。