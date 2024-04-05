!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

# 每个 Pod 的安全组

AWS 安全组充当 EC2 实例的虚拟防火墙,用于控制入站和出站流量。默认情况下,Amazon VPC CNI 将使用与主 ENI 关联的安全组。更具体地说,与实例关联的每个 ENI 都将具有相同的 EC2 安全组。因此,在节点上运行的每个 Pod 都共享该节点的安全组。

如下图所示,在工作节点上运行的所有应用程序 Pod 都将能够访问 RDS 数据库服务(考虑到 RDS 入站允许节点安全组)。安全组粒度太粗,因为它们适用于节点上运行的所有 Pod。Pod 的安全组提供了工作负载的网络隔离,这是良好深度防御策略的重要组成部分。

![illustration of node with security group connecting to RDS](./image.png)
通过为 Pod 使用安全组,您可以通过在共享计算资源上运行具有不同网络安全要求的应用程序来提高计算效率。可以在单个位置使用 EC2 安全组定义多种类型的安全规则,如 Pod 到 Pod 和 Pod 到外部 AWS 服务,并使用 Kubernetes 原生 API 应用于工作负载。下图显示了在 Pod 级别应用的安全组,以及它如何简化您的应用程序部署和节点架构。Pod 现在可以访问 Amazon RDS 数据库。

![illustration of pod and node with different security groups connecting to RDS](./image-2.png)

您可以通过为 VPC CNI 设置 `ENABLE_POD_ENI=true` 来启用 Pod 的安全组。启用后,由 EKS 管理的"VPC 资源控制器"在控制平面上运行,并创建并附加一个称为"aws-k8s-trunk-eni"的主干接口。主干接口充当附加到实例的标准网络接口。要管理主干接口,您必须将 `AmazonEKSVPCResourceController` 托管策略添加到与您的 Amazon EKS 集群相关的集群角色。

控制器还创建名为"aws-k8s-branch-eni"的分支接口,并将它们与主干接口关联。使用 [SecurityGroupPolicy](https://github.com/aws/amazon-vpc-resource-controller-k8s/blob/master/config/crd/bases/vpcresources.k8s.aws_securitygrouppolicies.yaml) 自定义资源为 Pod 分配安全组,并将它们与分支接口关联。由于安全组是使用网络接口指定的,因此我们现在能够在这些附加网络接口上调度需要特定安全组的 Pod。请查看 [EKS 用户指南中关于 Pod 安全组的部分](https://docs.aws.amazon.com/eks/latest/userguide/security-groups-for-pods.html),包括部署先决条件。

![illustration of worker subnet with security groups associated with ENIs](./image-3.png)
[分支接口容量是对现有实例类型限制的*可加性*。使用安全组的 Pod 不包括在 max-pods 公式中,当您使用 Pod 的安全组时,您需要考虑提高 max-pods 值或接受运行的 Pod 数量少于节点实际支持的情况。

m5.large 最多可有 9 个分支网络接口和 27 个辅助 IP 地址分配给其标准网络接口。如下例所示,m5.large 的默认 max-pods 为 29,EKS 将使用安全组的 Pod 计入最大 Pod 数。请参阅 [EKS 用户指南](https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html)了解如何更改节点的 max-pods。

当 Pod 使用安全组与[自定义网络](https://docs.aws.amazon.com/eks/latest/userguide/cni-custom-network.html)结合使用时,将使用 Pod 安全组中定义的安全组,而不是 ENIConfig 中指定的安全组。因此,在启用自定义网络时,请仔细评估使用 Pod 安全组时的安全组顺序。

## 建议

### 为存活探测禁用 TCP 提前分解

如果您使用存活或就绪探测,您还需要禁用 TCP 提前分解,以便 kubelet 可以通过 TCP 连接到分支网络接口上的 Pod。这仅在严格模式下需要。要执行此操作,请运行以下命令:]
```
kubectl edit daemonset aws-node -n kube-system
```


在 `initContainer` 部分下，将 `DISABLE_TCP_EARLY_DEMUX` 的值更改为 `true`。

### 使用安全组来利用现有的 AWS 配置投资

安全组可以更轻松地限制对 VPC 资源(如 RDS 数据库或 EC2 实例)的网络访问。每个 Pod 使用安全组的一个明显优势是可以重复使用现有的 AWS 安全组资源。
如果您使用安全组作为网络防火墙来限制对 AWS 服务的访问,我们建议将安全组应用于使用分支 ENI 的 Pod。如果您正在将应用程序从 EC2 实例迁移到 EKS,并且使用安全组限制对其他 AWS 服务的访问,请考虑为 Pod 使用安全组。

### 配置 Pod 安全组强制模式

Amazon VPC CNI 插件版本 1.11 添加了一个名为 `POD_SECURITY_GROUP_ENFORCING_MODE`("强制模式")的新设置。强制模式同时控制应用于 Pod 的安全组以及是否启用源 NAT。您可以将强制模式指定为严格模式或标准模式。严格模式是默认模式,反映了之前 VPC CNI 在 `ENABLE_POD_ENI` 设置为 `true` 时的行为。

在严格模式下,只有分支 ENI 安全组才会生效。源 NAT 也被禁用。

在标准模式下,主 ENI 和分支 ENI(与 Pod 关联)的安全组都会被应用。网络流量必须符合这两个安全组的要求。

!!! 警告
    任何模式更改只会影响新启动的 Pod。现有的 Pod 将使用创建 Pod 时配置的模式。如果客户想要更改流量行为,需要回收使用安全组的现有 Pod。

### 强制模式:使用严格模式隔离 Pod 和节点流量:

默认情况下,Pod 的安全组设置为"严格模式"。如果您必须完全分离 Pod 流量和节点其他部分的流量,请使用此设置。在严格模式下,源 NAT 被关闭,因此可以使用分支 ENI 出站安全组。

!!! 警告
    启用严格模式时,Pod 的所有出站流量都将离开节点并进入 VPC 网络。同一节点上的 Pod 之间的流量将通过 VPC 进行。这会增加 VPC 流量,并限制节点级功能。NodeLocal DNSCache 不支持严格模式。

### 强制模式:在以下情况下使用标准模式

**客户端源 IP 对 Pod 中的容器可见**
[如果您需要让容器在 Pod 中保留客户端源 IP 地址可见,请考虑将 `POD_SECURITY_GROUP_ENFORCING_MODE` 设置为 `standard`。Kubernetes 服务支持 externalTrafficPolicy=local 来保留客户端源 IP 地址(默认类型为 cluster)。您现在可以使用实例目标在标准模式下运行 NodePort 和 LoadBalancer 类型的 Kubernetes 服务,并将 externalTrafficPolicy 设置为 Local。`Local` 保留了客户端源 IP 地址,并避免了 LoadBalancer 和 NodePort 类型服务的第二次跳转。

**部署 NodeLocal DNSCache**

在为 Pod 使用安全组时,请配置标准模式以支持使用 [NodeLocal DNSCache](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/) 的 Pod。NodeLocal DNSCache 通过在集群节点上以 DaemonSet 的形式运行 DNS 缓存代理来提高集群 DNS 性能。这将帮助具有最高 DNS QPS 需求的 Pod 查询本地 kube-dns/CoreDNS,并拥有本地缓存,从而提高延迟。

在严格模式下不支持 NodeLocal DNSCache,因为所有网络流量,即使是到节点的流量,也会进入 VPC。

**支持 Kubernetes 网络策略**

我们建议在使用网络策略与具有关联安全组的 Pod 时使用标准强制模式。

我们强烈建议使用 Pod 的安全组来限制对 AWS 服务的网络级访问,这些 AWS 服务不是集群的一部分。考虑使用网络策略来限制 Pod 内部的网络流量,通常称为东西向流量。

### 识别与每个 Pod 的安全组的不兼容性

基于 Windows 的实例和非 Nitro 实例不支持 Pod 的安全组。要使用 Pod 的安全组,实例必须标记为 isTrunkingEnabled。如果您的 Pod 不依赖于 VPC 内部或外部的任何 AWS 服务,请使用网络策略来管理 Pod 之间的访问,而不是使用安全组。

### 使用每个 Pod 的安全组有效地控制对 AWS 服务的流量

如果在 EKS 集群中运行的应用程序需要与 VPC 内的其他资源(例如 RDS 数据库)进行通信,那么请考虑使用 Pod 的 SG。虽然有一些策略引擎允许您指定 CIDR 或 DNS 名称,但在与 VPC 内部的 AWS 服务进行通信时,它们并不是最佳选择。]
[网络策略](https://kubernetes.io/docs/concepts/services-networking/network-policies/)提供了一种控制集群内部和外部流量的机制。如果您的应用程序对其他AWS服务的依赖性有限,则应考虑使用Kubernetes网络策略。您可以配置基于CIDR范围的出口规则来限制对AWS服务的访问,而不是使用AWS原生语义(如安全组)。您可以使用Kubernetes网络策略来控制Pod之间(通常称为东西向流量)以及Pod与外部服务之间的网络流量。Kubernetes网络策略在OSI第3层和第4层实现。

Amazon EKS允许您使用[Calico](https://projectcalico.docs.tigera.io/getting-started/kubernetes/managed-public-cloud/eks)和[Cilium](https://docs.cilium.io/en/stable/intro/)等网络策略引擎。默认情况下,网络策略引擎未安装。请查看相应的安装指南,了解如何进行设置。有关如何使用网络策略的更多信息,请参见[EKS安全最佳实践](https://aws.github.io/aws-eks-best-practices/security/docs/network/#network-policy)。DNS主机名功能在网络策略引擎的企业版中可用,这可能对于控制Kubernetes服务/Pod与AWS外部运行的资源之间的流量很有用。您还可以考虑为默认不支持安全组的AWS服务提供DNS主机名支持。

### 标记单个安全组以使用AWS负载均衡器控制器

当为Pod分配多个安全组时,Amazon EKS建议使用[`kubernetes.io/cluster/$name`](http://kubernetes.io/cluster/$name)共享或拥有标记单个安全组。该标签允许AWS负载均衡器控制器更新安全组规则以将流量路由到Pod。如果只为Pod分配一个安全组,则标记是可选的。安全组中设置的权限是累加的,因此标记单个安全组对于负载均衡器控制器定位和协调规则就足够了。这也有助于遵守安全组[默认配额](https://docs.aws.amazon.com/vpc/latest/userguide/amazon-vpc-limits.html#vpc-limits-security-groups)。

### 配置出站流量的NAT

为分配了安全组的Pod禁用源NAT。对于使用需要访问互联网的安全组的Pod,请在私有子网上启动工作节点,并配置NAT网关或实例,并在CNI中启用[外部SNAT](https://docs.aws.amazon.com/eks/latest/userguide/external-snat.html)。
```
kubectl set env daemonset -n kube-system aws-node AWS_VPC_K8S_CNI_EXTERNALSNAT=true
```

部署带有安全组的 Pod 到私有子网

必须在部署到私有子网的节点上运行分配有安全组的 Pod。请注意，部署到公共子网的带有分配安全组的 Pod 将无法访问互联网。

验证 Pod 规范文件中的 *terminationGracePeriodSeconds*

确保您的 Pod 规范文件中的 `terminationGracePeriodSeconds` 不为零(默认为 30 秒)。这对于 Amazon VPC CNI 从工作节点中删除 Pod 网络至关重要。如果设置为零,CNI 插件不会从主机上删除 Pod 网络,分支 ENI 也无法得到有效清理。

在 Fargate 上使用安全组的 Pod

Fargate 上运行的 Pod 的安全组的工作方式与在 EC2 工作节点上运行的 Pod 非常相似。例如,您必须在引用 SecurityGroupPolicy 中的安全组之前先创建该安全组。默认情况下,当您没有明确为 Fargate Pod 分配 SecurityGroupPolicy 时,[集群安全组](https://docs.aws.amazon.com/eks/latest/userguide/sec-group-reqs.html)会被分配给所有 Fargate Pod。为简单起见,您可能希望将集群安全组添加到 Fagate Pod 的 SecurityGroupPolicy,否则您将不得不为您的安全组添加最小的安全组规则。您可以使用 describe-cluster API 找到集群安全组。
```bash
 aws eks describe-cluster --name CLUSTER_NAME --query 'cluster.resourcesVpcConfig.clusterSecurityGroupId'
```

```bash
cat >my-fargate-sg-policy.yaml <<EOF
apiVersion: vpcresources.k8s.aws/v1beta1
kind: SecurityGroupPolicy
metadata:
  name: my-fargate-sg-policy
  namespace: my-fargate-namespace
spec:
  podSelector: 
    matchLabels:
      role: my-fargate-role
  securityGroups:
    groupIds:
      - cluster_security_group_id
      - my_fargate_pod_security_group_id
EOF
```

最小安全组规则列在[此处](https://docs.aws.amazon.com/eks/latest/userguide/sec-group-reqs.html)。这些规则允许Fargate Pod与集群内的kube-apiserver、kubelet和CoreDNS等服务进行通信。您还需要添加规则,以允许从Fargate Pod进出的入站和出站连接。这将使您的Pod能够与VPC中的其他Pod或资源进行通信。此外,您还必须包括Fargate从Amazon ECR或DockerHub等其他容器注册表拉取容器�像的规则。有关更多信息,请参见[AWS General Reference](https://docs.aws.amazon.com/general/latest/gr/aws-ip-ranges.html)中的AWS IP地址范围。

您可以使用以下命令来查找应用于Fargate Pod的安全组。
```bash
kubectl get pod FARGATE_POD -o jsonpath='{.metadata.annotations.vpc\.amazonaws\.com/pod-eni}{"\n"}'
```

注意记下上述命令中的eniId。
```bash
aws ec2 describe-network-interfaces --network-interface-ids ENI_ID --query 'NetworkInterfaces[*].Groups[*]'
```

现有的 Fargate 容器组必须被删除并重新创建,以便应用新的安全组。例如,以下命令启动了 example-app 的部署。要更新特定的容器组,您可以在以下命令中更改命名空间和部署名称。
```bash
kubectl rollout restart -n example-ns deployment example-pod
```

