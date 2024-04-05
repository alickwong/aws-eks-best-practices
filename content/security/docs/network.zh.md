!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

# 网络安全

网络安全有几个方面。第一个涉及应用限制网络流量在服务之间流动的规则。第二个涉及在传输过程中对流量进行加密。在 EKS 上实施这些安全措施的机制各不相同,但通常包括以下内容:

## 流量控制

- 网络策略
- 安全组

## 网络加密

- 服务网格
- 容器网络接口 (CNI)
- 入口控制器和负载均衡器
- Nitro 实例
- 带有 cert-manager 的 ACM 私有 CA

## 网络策略

在 Kubernetes 集群中,默认情况下允许所有 Pod 到 Pod 的通信。虽然这种灵活性可能有助于促进实验,但它并不被认为是安全的。Kubernetes 网络策略为您提供了一种机制来限制 Pod 之间(通常称为东/西向流量)以及 Pod 与外部服务之间的网络流量。Kubernetes 网络策略在 OSI 模型的第 3 层和第 4 层运行。网络策略使用 pod、命名空间选择器和标签来识别源和目标 pod,但也可以包括 IP 地址、端口号、协议或这些的组合。网络策略可以应用于 pod 的入站或出站连接,通常称为入口和出口规则。

通过 Amazon VPC CNI 插件的原生网络策略支持,您可以实施网络策略来保护 kubernetes 集群中的网络流量。这与上游 Kubernetes 网络策略 API 集成,确保兼容性和遵守 Kubernetes 标准。您可以使用上游 API 支持的不同[标识符](https://kubernetes.io/docs/concepts/services-networking/network-policies/)定义策略。默认情况下,允许 pod 的所有入站和出站流量。当指定了带有 policyType Ingress 的网络策略时,只有来自 pod 节点和入口规则允许的连接才被允许进入 pod。出口规则也是如此。如果定义了多个规则,则在做出决策时会考虑所有规则的并集。因此,评估顺序不会影响策略结果。

!!! attention
    当您首次配置 EKS 集群时,VPC CNI 网络策略功能默认是禁用的。确保您部署了受支持的 VPC CNI 插件版本,并在 vpc-cni 插件上将 `ENABLE_NETWORK_POLICY` 标志设置为 `true` 以启用此功能。请参考 [Amazon EKS 用户指南](https://docs.aws.amazon.com/eks/latest/userguide/managing-vpc-cni.html)了解详细说明。

## 建议

### 从最小权限原则开始使用网络策略

#### 创建默认拒绝策略

与 RBAC 策略一样,建议遵循最小权限访问原则来使用网络策略。首先创建一个拒绝所有策略,限制命名空间内的所有入站和出站流量。
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

!!! 提示
    上图由 [Tufin](https://orca.tufin.io/netpol/) 的网络策略查看器创建。

#### 创建允许 DNS 查询的规则

一旦您设置了默认拒绝所有的规则,您就可以开始添加更多规则,例如允许 pod 查询 CoreDNS 进行名称解析的规则。
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

![允许 DNS 访问](./images/allow-dns-access.jpg)

#### 逐步添加规则以有选择地允许命名空间/pod 之间的流量

了解应用程序需求,并根据需要创建细粒度的入口和出口规则。下面的示例展示了如何限制 `client-one` 对 `app-one` 的 80 端口的入口流量。这有助于最小化攻击面,降低未经授权访问的风险。
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


# 监控网络策略执行

- **使用网络策略编辑器**
  - [网络策略编辑器](https://networkpolicy.io/)可帮助进行可视化、安全评分和根据网络流量日志自动生成
  - 以交互式方式构建网络策略
- **审核日志**
  - 定期审查您的 EKS 集群的审核日志
  - 审核日志提供了大量有关在您的集群上执行的操作的信息,包括对网络策略的更改
  - 使用此信息跟踪您的网络策略随时间的变化,并检测任何未经授权或意外的更改
- **自动化测试**
  - 通过创建一个与生产环境相似的测试环境,并定期部署试图违反您的网络策略的工作负载来实施自动化测试。
- **监控指标**
  - 配置您的可观察性代理,以从 VPC CNI 节点代理中抓取 Prometheus 指标,这样可以监控代理健康状况和 SDK 错误。
- **定期审核网络策略**
  - 定期审核您的网络策略,以确保它们满足您当前的应用程序要求。随着您的应用程序的发展,审核可以让您有机会删除冗余的入口和出口规则,并确保您的应用程序没有过多的权限。
- **使用 Open Policy Agent (OPA) 确保网络策略存在**
  - 使用如下所示的 OPA 策略,确保在载入应用程序 pod 之前网络策略已经存在。如果没有相应的网络策略,此策略将拒绝载入带有标签 `k8s-app: sample-app` 的 k8s pod。

![allow-ingress-app-one](./images/allow-ingress-app-one.png)
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

### 故障排除

#### 监控 vpc-network-policy-controller 和 node-agent 日志

启用 EKS 控制平面控制器管理器日志以诊断网络策略功能。您可以将控制平面日志流式传输到 CloudWatch 日志组,并使用 [CloudWatch 日志洞察](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html)执行高级查询。从日志中,您可以查看哪些 pod 端点对象被解析为网络策略,策略的协调状态,以及调试策略是否按预期工作。

此外,Amazon VPC CNI 允许您从 EKS 工作节点启用将策略执行日志收集和导出到 [Amazon Cloudwatch](https://aws.amazon.com/cloudwatch/)。启用后,您可以利用 [CloudWatch 容器洞察](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights.html)来提供有关网络策略使用情况的洞察。

Amazon VPC CNI 还提供了一个 SDK,可以提供与节点上的 eBPF 程序交互的接口。当部署 `aws-node` 到节点上时,就会安装该 SDK。您可以在节点上的 `/opt/cni/bin` 目录下找到 SDK 二进制文件。启动时,SDK 提供对基本功能的支持,如检查 eBPF 程序和映射。
```shell
sudo /opt/cni/bin/aws-eks-na-cli ebpf progs
```


#### 记录网络流量元数据

[AWS VPC 流日志](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html)捕获通过 VPC 流动的流量的元数据,例如源和目标 IP 地址和端口以及接受/丢弃的数据包。这些信息可用于查找 VPC 内资源之间的可疑或异常活动,包括 Pod。但是,由于 Pod 的 IP 地址随着它们的替换而频繁变化,流日志可能无法单独满足需求。Calico Enterprise 扩展了流日志,增加了 Pod 标签和其他元数据,使得解析 Pod 之间的流量更加容易。

## 安全组

EKS 使用 [AWS VPC 安全组](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html)(SG)来控制 Kubernetes 控制平面和集群工作节点之间的流量。安全组也用于控制工作节点之间、其他 VPC 资源和外部 IP 地址之间的流量。当您配置 EKS 集群(Kubernetes 版本为 1.14-eks.3 或更高)时,会自动为您创建一个集群安全组。此安全组允许 EKS 控制平面与托管节点组中的节点之间进行无限制的通信。为简单起见,建议您将集群 SG 添加到所有节点组,包括非托管节点组。

在 Kubernetes 版本 1.14 和 EKS 版本 eks.3 之前,EKS 控制平面和节点组配置了单独的安全组。控制平面和节点组安全组的最小和建议规则可以在 [https://docs.aws.amazon.com/eks/latest/userguide/sec-group-reqs.html](https://docs.aws.amazon.com/eks/latest/userguide/sec-group-reqs.html) 找到。_控制平面安全组_的最小规则允许来自工作节点 SG 的 443 端口入站。这个规则允许 kubelet 与 Kubernetes API 服务器通信。它还包括 10250 端口的出站流量到工作节点 SG;10250 是 kubelet 监听的端口。类似地,_节点组_的最小规则允许来自控制平面 SG 的 10250 端口入站和 443 端口出站到控制平面 SG。最后,还有一条允许节点组内节点之间进行无限制通信的规则。

如果您需要控制集群内部服务与集群外部服务(如 RDS 数据库)之间的通信,请考虑使用[针对 Pod 的安全组](https://docs.aws.amazon.com/eks/latest/userguide/security-groups-for-pods.html)。使用针对 Pod 的安全组,您可以将现有的安全组分配给一组 Pod。

!!! warning
    如果在创建 Pod 之前引用了一个不存在的安全组,那么 Pod 将无法调度。

您可以通过创建一个 `SecurityGroupPolicy` 对象并指定 `PodSelector` 或 `ServiceAccountSelector` 来控制哪些 pod 被分配到一个安全组。将选择器设置为 `{}` 将把 `SecurityGroupPolicy` 中引用的 SG 分配给某个命名空间中的所有 pod 或所有服务帐户。在为 pod 实施安全组之前,请务必熟悉所有[注意事项](https://docs.aws.amazon.com/eks/latest/userguide/security-groups-for-pods.html#security-groups-pods-considerations)。

!!! important
    如果您对 pod 使用 SG,则**必须**创建允许端口 53 出站到集群安全组的 SG。同样,您**必须**更新集群安全组以接受来自 pod 安全组的端口 53 入站流量。

!!! important
    [安全组的限制](https://docs.aws.amazon.com/vpc/latest/userguide/amazon-vpc-limits.html#vpc-limits-security-groups)在将安全组用于 pod 时仍然适用,因此请谨慎使用。

!!! important
    您**必须**为 pod 配置的所有探测创建来自集群安全组(kubelet)的入站流量规则。

!!! important
    pod 的安全组依赖于一个称为[ENI 中继](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/container-instance-eni.html)的功能,该功能是为了增加 EC2 实例的 ENI 密度而创建的。当一个 pod 被分配到一个 SG 时,VPC 控制器会将一个来自节点组的分支 ENI 与该 pod 关联。如果在调度 pod 时节点组中没有足够的分支 ENI 可用,pod 将保持待定状态。一个实例可以支持的分支 ENI 数量因实例类型/系列而异。请参阅[https://docs.aws.amazon.com/eks/latest/userguide/security-groups-for-pods.html#supported-instance-types](https://docs.aws.amazon.com/eks/latest/userguide/security-groups-for-pods.html#supported-instance-types)了解更多详细信息。

虽然 pod 的安全组提供了一种 AWS 原生的方式来控制集群内部和外部的网络流量,而无需依赖策略守护进程,但也有其他选择。例如,Cilium 策略引擎允许您在网络策略中引用 DNS 名称。Calico Enterprise 包括一个将网络策略映射到 AWS 安全组的选项。如果您已经实施了 Istio 等服务网格,您可以使用出口网关来限制网络出口流量到特定的完全限定域名或 IP 地址。有关此选项的更多信息,请阅读[Istio 中的出口流量控制](https://istio.io/blog/2019/egress-traffic-control-in-istio-part-1/)的三篇系列文章。

## 何时使用网络策略与何时使用 pod 的安全组?

### 何时使用 Kubernetes 网络策略

- **控制 pod 到 pod 的流量**
  - 适用于控制集群内部 pod 之间的网络流量(东西向流量)
- **在 IP 地址或端口级别(OSI 第 3 层或第 4 层)控制流量**

### 何时使用 AWS 安全组用于 pod (SGP)

- **利用现有的 AWS 配置**
  - 如果您已经有一组复杂的 EC2 安全组来管理对 AWS 服务的访问,并且正在将应用程序从 EC2 实例迁移到 EKS,SGP 可能是一个非常好的选择,允许您重复使用安全组资源并将其应用于您的 pod。
- **控制对 AWS 服务的访问**
  - 您在 EKS 集群中运行的应用程序需要与其他 AWS 服务(RDS 数据库)进行通信,使用 SGP 作为一种有效的机制来控制 pod 到 AWS 服务的流量。
- **隔离 Pod 和节点流量**
  - 如果您想完全分离 pod 流量和其他节点流量,请在 `POD_SECURITY_GROUP_ENFORCING_MODE=strict` 模式下使用 SGP。

### 使用"pod 安全组"和"网络策略"的最佳实践

- **分层安全**
  - 结合使用 SGP 和 Kubernetes 网络策略来实现分层安全方法
  - 使用 SGP 来限制对集群外 AWS 服务的网络级访问,而 Kubernetes 网络策略可以限制集群内 pod 之间的网络流量
- **最小权限原则**
  - 只允许 pod 或命名空间之间必要的流量
- **细分您的应用程序**
  - 尽可能通过网络策略细分应用程序,以减少应用程序被入侵时的影响范围
- **保持策略简单明了**
  - Kubernetes 网络策略可能非常细化和复杂,最好保持它们尽可能简单,以减少配置错误的风险并降低管理开销
- **减少攻击面**
  - 通过限制应用程序的暴露来最小化攻击面

!!! attention
    pod 安全组提供两种强制模式:"strict"和"standard"。当在 EKS 集群中同时使用网络策略和 pod 安全组功能时,您必须使用"standard"模式。

在网络安全方面,分层方法通常是最有效的解决方案。结合使用 Kubernetes 网络策略和 SGP 可以为您在 EKS 中运行的应用程序提供一个强大的纵深防御策略。

## 服务网格策略执行或 Kubernetes 网络策略

`服务网格`是一个专门的基础设施层,您可以将其添加到应用程序中。它允许您透明地添加诸如可观察性、流量管理和安全性等功能,而无需将它们添加到您自己的代码中。

服务网格在 OSI 模型的第 7 层(应用程序)执行策略,而 Kubernetes 网络策略在第 3 层(网络)和第 4 层(传输)运行。这个领域有很多产品,如 AWS AppMesh、Istio、Linkerd 等。

### 何时使用服务网格进行策略执行

- 已经投资了服务网格
- 需要更高级的功能,如流量管理、可观察性和安全性
  - 流量控制、负载均衡、熔断、限流、超时等

- 详细了解您的服务性能(延迟、错误率、每秒请求数、请求量等)
- 您希望实施和利用服务网格的安全功能,如双向 TLS

### 选择 Kubernetes 网络策略以简化使用情况

- 限制哪些 pod 可以相互通信
- 网络策略所需的资源比服务网格少,因此适合于较简单的使用情况或较小的集群,在这种情况下运行和管理服务网格可能无法得到合理的证明

!!! tip
    网络策略和服务网格也可以一起使用。使用网络策略提供基本的安全性和 pod 之间的隔离,然后使用服务网格添加其他功能,如流量管理、可观察性和安全性。

## 第三方网络策略引擎

当您有高级策略需求(如全局网络策略、支持基于 DNS 主机名的规则、第 7 层规则、基于服务帐户的规则以及显式拒绝/日志操作等)时,请考虑使用第三方网络策略引擎。[Calico](https://docs.projectcalico.org/introduction/) 是来自 [Tigera](https://tigera.io) 的开源策略引擎,可与 EKS 很好地配合使用。除了实现完整的 Kubernetes 网络策略功能外,Calico 还支持扩展的网络策略,具有更丰富的功能,包括与 Istio 集成时对第 7 层规则(如 HTTP)的支持。Calico 策略可以针对命名空间、pod 或服务帐户进行范围界定,也可以全局应用。当策略的范围限定在服务帐户时,它会将一组入口/出口规则与该服务帐户相关联。通过适当的 RBAC 规则,您可以防止团队覆盖这些规则,让 IT 安全专业人员能够安全地委派命名空间的管理。Isovalent(Cilium 的维护者)也扩展了网络策略,部分支持第 7 层规则(如 HTTP)。Cilium 还支持 DNS 主机名,这对于限制 Kubernetes 服务/pod 与 VPC 内外资源之间的流量很有用。相比之下,Calico Enterprise 包括一项功能,可将 Kubernetes 网络策略映射到 AWS 安全组,以及 DNS 主机名。

您可以在 [https://github.com/ahmetb/kubernetes-network-policy-recipes](https://github.com/ahmetb/kubernetes-network-policy-recipes) 找到常见的 Kubernetes 网络策略列表。Calico 的类似规则可在 [https://docs.projectcalico.org/security/calico-network-policy](https://docs.projectcalico.org/security/calico-network-policy) 获得。

### 迁移到 Amazon VPC CNI 网络策略引擎

为了保持一致性并避免意外的 pod 通信行为,建议在您的集群中只部署一个网络策略引擎。如果您想从 3P 迁移到 VPC CNI 网络策略引擎,我们建议在启用 VPC CNI 网络策略支持之前,将现有的 3P NetworkPolicy CRD 转换为 Kubernetes NetworkPolicy 资源。并且,在将它们应用于生产环境之前,请在单独的测试集群中测试迁移后的策略。这样可以帮助您识别和解决 pod 通信行为中的任何潜在问题或不一致性。

#### 迁移工具

为了协助您的迁移过程,我们开发了一个名为 [K8s Network Policy Migrator](https://github.com/awslabs/k8s-network-policy-migrator) 的工具,它可以将您现有的 Calico/Cilium 网络策略 CRD 转换为 Kubernetes 原生网络策略。转换后,您可以直接在运行 VPC CNI 网络策略控制器的新集群上测试转换后的网络策略。该工具旨在帮助您简化迁移过程,确保顺利过渡。

!!! 重要
    迁移工具只会转换与原生 Kubernetes 网络策略 API 兼容的 3P 策略。如果您使用 3P 插件提供的高级网络策略功能,迁移工具将跳过并报告它们。

请注意,迁移工具目前不受 AWS VPC CNI 网络策略工程团队支持,它是根据最大努力原则提供给客户使用的。我们鼓励您利用这个工具来促进您的迁移过程。如果您遇到任何问题或错误,请在 [GitHub 问题](https://github.com/awslabs/k8s-network-policy-migrator/issues)中创建一个问题。您的反馈对我们很宝贵,将有助于我们不断改进我们的服务。

### 其他资源

- [Kubernetes & Tigera: 网络策略、安全性和审计](https://youtu.be/lEY2WnRHYpg)
- [Calico Enterprise](https://www.tigera.io/tigera-products/calico-enterprise/)
- [Cilium](https://cilium.readthedocs.io/en/stable/intro/)
- [NetworkPolicy Editor](https://cilium.io/blog/2021/02/10/network-policy-editor) 来自 Cilium 的交互式策略编辑器
- [Inspektor Gadget 建议 network-policy 工具](https://www.inspektor-gadget.io/docs/latest/gadgets/advise/network-policy/) 根据网络流量分析建议网络策略

## 传输中的加密

需要符合 PCI、HIPAA 或其他法规的应用程序可能需要在传输过程中加密数据。如今,TLS 已成为加密网络流量的事实标准。TLS 就像它的前身 SSL 一样,提供了在网络上进行安全通信的加密协议。TLS 使用对称加密,其中用于加密数据的密钥是基于会话开始时协商的共享秘密生成的。以下是在 Kubernetes 环境中加密数据的几种方式。
### Nitro 实例

默认情况下,在以下 Nitro 实例类型之间交换的流量(例如 C5n、G4、I3en、M5dn、M5n、P3dn、R5dn 和 R5n)会自动加密。当存在中间跳转,如传输网关或负载均衡器时,流量不会加密。有关传输中加密的更多详细信息以及支持默认网络加密的完整实例类型列表,请参见[传输中加密](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/data-protection.html#encryption-transit)。

### 容器网络接口 (CNI)

[WeaveNet](https://www.weave.works/oss/net/) 可以配置为使用 NaCl 加密对所有流量进行自动加密,并使用 IPsec ESP 对快速数据路径流量进行加密。

### 服务网格

传输中加密也可以通过服务网格(如 App Mesh、Linkerd v2 和 Istio)来实现。App Mesh 支持使用 X.509 证书或 Envoy 的 Secret Discovery Service (SDS) 进行双向 TLS (mTLS)。Linkerd 和 Istio 也都支持 mTLS。

[aws-app-mesh-examples](https://github.com/aws/aws-app-mesh-examples) GitHub 存储库提供了使用 X.509 证书和 SPIRE 作为 SDS 提供程序配置 mTLS 的演练:

- [使用 X.509 证书配置 mTLS](https://github.com/aws/aws-app-mesh-examples/tree/main/walkthroughs/howto-k8s-mtls-file-based)
- [使用 SPIRE (SDS) 配置 TLS](https://github.com/aws/aws-app-mesh-examples/tree/main/walkthroughs/howto-k8s-mtls-sds-based)

App Mesh 还支持使用由 [AWS Certificate Manager](https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html) (ACM) 颁发的私有证书或存储在虚拟节点本地文件系统上的证书进行 [TLS 加密](https://docs.aws.amazon.com/app-mesh/latest/userguide/virtual-node-tls.html)。

[aws-app-mesh-examples](https://github.com/aws/aws-app-mesh-examples) GitHub 存储库提供了使用 ACM 颁发的证书和打包在 Envoy 容器中的证书配置 TLS 的演练:

- [使用文件提供的 TLS 证书配置 TLS](https://github.com/aws/aws-app-mesh-examples/tree/master/walkthroughs/howto-tls-file-provided)
- [使用 AWS Certificate Manager 配置 TLS](https://github.com/aws/aws-app-mesh-examples/tree/master/walkthroughs/tls-with-acm)

### 入口控制器和负载均衡器
[Ingress控制器是一种让您能够智能地将来自集群外部的HTTP/S流量路由到集群内部运行的服务的方式。通常情况下,这些Ingress会由一个第4层负载均衡器(如经典负载均衡器或网络负载均衡器(NLB))提供前端支持。加密流量可以在网络中的不同位置终止,例如在负载均衡器、Ingress资源或Pod处。您如何以及在何处终止SSL连接最终将由您组织的网络安全政策决定。例如,如果您有一项要求端到端加密的政策,您将不得不在Pod处解密流量。这将给您的Pod带来额外的负担,因为它将不得不花费周期来建立初始握手。总的来说,SSL/TLS处理是非常CPU密集的。因此,如果您有灵活性,请尝试在Ingress或负载均衡器处执行SSL卸载。]

#### 在AWS弹性负载均衡器上使用加密

[AWS应用程序负载均衡器](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html)(ALB)和[网络负载均衡器](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html)(NLB)都支持传输加密(SSL和TLS)。ALB的`alb.ingress.kubernetes.io/certificate-arn`注解让您指定要添加到ALB的证书。如果您省略该注解,控制器将尝试使用可用的[AWS证书管理器(ACM)](https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html)证书来匹配需要它的侦听器。从EKS v1.15开始,您可以使用如下例所示的`service.beta.kubernetes.io/aws-load-balancer-ssl-cert`注解与NLB一起使用。
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
- [如何使用ACM在Amazon EKS工作负载上终止HTTPS流量?](https://aws.amazon.com/premiumsupport/knowledge-center/terminate-https-traffic-eks-acm/)

!!! 注意
    一些入口,如AWS LB控制器,使用注释而不是作为入口规范的一部分来实现SSL/TLS。

### 使用cert-manager的ACM私有CA

您可以使用ACM私有证书颁发机构(CA)和[cert-manager](https://cert-manager.io/)在入口、pod和pod之间启用TLS和mTLS,以保护您的EKS应用程序工作负载。ACM私有CA是一个高可用、安全的托管CA,无需承担管理自己的CA的前期和维护成本。如果您正在使用默认的Kubernetes证书颁发机构,那么使用ACM私有CA可以提高您的安全性并满足合规性要求。与默认CA在内存中存储密钥(安全性较低)相比,ACM私有CA在FIPS 140-2 Level 3硬件安全模块中保护私钥(非常安全)。集中式CA还可以为内部和外部Kubernetes环境的私有证书提供更多控制和改善审核功能。

#### 用于工作负载之间的双向TLS的短期CA模式

在EKS中使用ACM私有CA进行mTLS时,建议您使用短期证书与_短期CA模式_。尽管在通用CA模式下也可以颁发短期证书,但使用短期CA模式对于需要频繁颁发新证书的用例来说更加经济高效(比通用模式便宜约75%)。此外,您应该尝试将私有证书的有效期与EKS集群中pod的生命周期保持一致。[在此了解有关ACM私有CA及其优势的更多信息](https://aws.amazon.com/certificate-manager/private-certificate-authority/)。

#### ACM设置说明

首先,按照[ACM私有CA技术文档](https://docs.aws.amazon.com/acm-pca/latest/userguide/create-CA.html)中提供的步骤创建一个私有CA。创建私有CA后,按照[常规安装说明](https://cert-manager.io/docs/installation/)安装cert-manager。安装cert-manager后,按照[GitHub上的设置说明](https://github.com/cert-manager/aws-privateca-issuer#setup)安装私有CA Kubernetes cert-manager插件。该插件允许cert-manager从ACM私有CA请求私有证书。

现在您已经拥有了一个私有 CA 和一个安装了 cert-manager 和插件的 EKS 集群,是时候设置权限并创建颁发者了。更新 EKS 节点角色的 IAM 权限,以允许访问 ACM 私有 CA。将 `<CA_ARN>` 替换为您的私有 CA 的值:
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

[服务角色用于 IAM 帐户，或 IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)也可以使用。请参阅下面的"其他资源"部分以获取完整的示例。

在 Amazon EKS 中创建一个颁发者,方法是创建一个名为 cluster-issuer.yaml 的自定义资源定义文件,其中包含以下文本,并用您的私有 CA 的 `<CA_ARN>` 和 `<Region>` 信息替换。
```yaml
apiVersion: awspca.cert-manager.io/v1beta1
kind: AWSPCAClusterIssuer
metadata:
          name: demo-test-root-ca
spec:
          arn: <CA_ARN>
          region: <Region>
```

部署您创建的发行人。
```bash
kubectl apply -f cluster-issuer.yaml
```


您的 EKS 集群已配置为从私有 CA 请求证书。您现在可以使用 cert-manager 的 `Certificate` 资源通过将 `issuerRef` 字段的值更改为您之前创建的私有 CA Issuer 来颁发证书。有关如何指定和请求证书资源的更多详细信息,请查看 cert-manager 的 [Certificate Resources 指南](https://cert-manager.io/docs/usage/certificate/)。[在此处查看示例](https://github.com/cert-manager/aws-privateca-issuer/tree/main/config/samples/)。

### 使用 Istio 和 cert-manager 的 ACM 私有 CA

如果您在 EKS 集群中运行 Istio,您可以禁用 Istio 控制平面(特别是 `istiod`)作为根证书颁发机构(CA),并将 ACM 私有 CA 配置为工作负载之间 mTLS 的根 CA。如果您采用这种方法,请考虑在 ACM 私有 CA 中使用 _短期 CA 模式_。请参考[前一节](#short-lived-ca-mode-for-mutual-tls-between-workloads)和[此博客文章](https://aws.amazon.com/blogs/security/how-to-use-aws-private-certificate-authority-short-lived-certificate-mode)了解更多详细信息。

#### Istio 中证书签名的工作原理(默认)

Kubernetes 中的工作负载使用服务帐户进行标识。如果您没有指定服务帐户,Kubernetes 将自动为您的工作负载分配一个服务帐户。此外,服务帐户会自动挂载相关的令牌。工作负载使用此令牌向 Kubernetes API 进行身份验证。服务帐户可能足以作为 Kubernetes 中的身份,但 Istio 有自己的身份管理系统和 CA。当一个工作负载连同其 envoy 边车代理启动时,它需要从 Istio 获得一个身份,以便被视为可信并被允许与网格中的其他服务进行通信。

为了从 Istio 获得此身份,`istio-agent` 会发送一个称为证书签名请求(或 CSR)的请求到 Istio 控制平面。此 CSR 包含服务帐户令牌,以便在处理之前验证工作负载的身份。此验证过程由 `istiod` 处理,它同时充当注册机构(或 RA)和 CA。RA 充当看门人,确保只有经过验证的 CSR 才能进入 CA。一旦 CSR 得到验证,它将被转发到 CA,CA 将颁发一个包含 [SPIFFE](https://spiffe.io/) 身份的证书,该身份与服务帐户相关。此证书称为 SPIFFE 可验证身份文档(或 SVID)。SVID 被分配给请求服务,用于标识目的和加密通信中的传输流量。

![Istio 证书签名请求的默认流程](./images/default-istio-csr-flow.png)

#### 使用 ACM 私有 CA 的 Istio 证书签名工作原理

您可以使用名为 Istio 证书签名请求代理 ([istio-csr](https://cert-manager.io/docs/projects/istio-csr/)) 的 cert-manager 插件来将 Istio 与 ACM Private CA 集成。此代理允许 Istio 工作负载和控制平面组件使用 cert manager 发行者（在本例中为 ACM Private CA）进行安全保护。_istio-csr_ 代理公开了与默认配置中 _istiod_ 提供的相同的服务，用于验证传入的 CSR。但是，在验证之后，它将请求转换为 cert manager 支持的资源（即与外部 CA 发行者的集成）。

每当有工作负载发出 CSR 时，它都会被转发到 _istio-csr_，后者将从 ACM Private CA 请求证书。_istio-csr_ 与 ACM Private CA 之间的通信由 [AWS Private CA 发行者插件](https://github.com/cert-manager/aws-privateca-issuer)启用。Cert manager 使用此插件从 ACM Private CA 请求 TLS 证书。发行者插件将与 ACM Private CA 服务进行通信，以请求为工作负载签署的证书。一旦证书被签署，它将被返回给 _istio-csr_，后者将读取签名的请求并将其返回给发起 CSR 的工作负载。

![Istio 证书签名请求与 istio-csr 的流程](./images/istio-csr-with-acm-private-ca.png)

#### Istio 与 Private CA 设置说明

1. 首先按照[本节中的设置说明](#acm-private-ca-with-cert-manager)完成以下操作:
2. 创建一个 Private CA
3. 安装 cert-manager
4. 安装发行者插件
5. 设置权限并创建一个发行者。该发行者代表 CA，用于签署 `istiod` 和网格工作负载证书。它将与 ACM Private CA 进行通信。
6. 创建一个 `istio-system` 命名空间。这是部署 `istiod 证书`和其他 Istio 资源的位置。
7. 安装配置有 AWS Private CA 发行者插件的 Istio CSR。您可以保留工作负载的证书签名请求以验证它们是否获得批准和签名 (`preserveCertificateRequests=true`)。

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

8. 使用自定义配置安装 Istio，将 `istiod` 替换为 `cert-manager istio-csr` 作为网格的证书提供程序。这个过程可以使用 [Istio Operator](https://tetrate.io/blog/what-is-istio-operator/) 来完成。

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
        # 将证书提供程序更改为 cert-manager istio agent
        caAddress: cert-manager-istio-csr.cert-manager.svc:443
      components:
        pilot:
          k8s:
            env:
              # 禁用 istiod CA 服务器功能
            - name: ENABLE_CA_SERVER
              value: "false"
            overlays:
            - apiVersion: apps/v1
              kind: Deployment
              name: istiod
              patches:

                # 从 Secret 挂载 istiod 服务和 webhook 证书
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

10. 现在您可以在 EKS 集群中部署一个工作负载到网格中并[强制执行 mTLS](https://istio.io/latest/docs/reference/config/security/peer_authentication/)。

![Istio 证书签名请求](./images/istio-csr-requests.png)

## 工具和资源

- [Amazon EKS 安全沉浸式研讨会 - 网络安全](https://catalog.workshops.aws/eks-security-immersionday/en-US/6-network-security)
- [如何实施 cert-manager 和 ACM 私有 CA 插件以在 EKS 中启用 TLS](https://aws.amazon.com/blogs/security/tls-enabled-kubernetes-clusters-with-acm-private-ca-and-amazon-eks-2/)。
- [使用新的 AWS Load Balancer Controller 和 ACM 私有 CA 在 Amazon EKS 上设置端到端 TLS 加密](https://aws.amazon.com/blogs/containers/setting-up-end-to-end-tls-encryption-on-amazon-eks-with-the-new-aws-load-balancer-controller/)。
- [GitHub 上的 Private CA Kubernetes cert-manager 插件](https://github.com/cert-manager/aws-privateca-issuer)。
- [Private CA Kubernetes cert-manager 插件用户指南](https://docs.aws.amazon.com/acm-pca/latest/userguide/PcaKubernetes.html)。
- [如何使用 AWS 私有证书颁发机构短期证书模式](https://aws.amazon.com/blogs/security/how-to-use-aws-private-certificate-authority-short-lived-certificate-mode)
- [使用 ksniff 和 Wireshark 验证 Kubernetes 中的服务网格 TLS](https://itnext.io/verifying-service-mesh-tls-in-kubernetes-using-ksniff-and-wireshark-2e993b26bf95)
- [ksniff](https://github.com/eldadru/ksniff)
- [egress-operator](https://github.com/monzo/egress-operator) 一个操作符和 DNS 插件,用于控制集群外部流量,无需进行协议检查
- [SUSE 的 NeuVector](https://www.suse.com/neuvector/) 开源的零信任容器安全平台,提供策略网络规则、数据丢失预防 (DLP)、Web 应用防火墙 (WAF) 和网络威胁签名。
