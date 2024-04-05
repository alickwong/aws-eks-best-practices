!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

# Amazon VPC CNI

<iframe width="560" height="315" src="https://www.youtube.com/embed/RBE3yk2UlYA" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

Amazon EKS通过[Amazon VPC Container Network Interface](https://github.com/aws/amazon-vpc-cni-k8s)[(VPC CNI)](https://github.com/aws/amazon-vpc-cni-k8s)插件实现集群网络。该CNI插件允许Kubernetes Pod具有与VPC网络上相同的IP地址。更具体地说,Pod内的所有容器共享一个网络命名空间,它们可以使用本地端口相互通信。

Amazon VPC CNI有两个组件:

* CNI二进制文件,用于设置Pod网络以实现Pod到Pod的通信。CNI二进制文件在节点根文件系统上运行,当有新的Pod添加到节点或从节点中删除现有Pod时,kubelet会调用它。
* ipamd,一个长期运行的节点本地IP地址管理(IPAM)守护进程,负责:
  * 管理节点上的ENI,以及
  * 维护可用IP地址或前缀的预热池

创建实例时,EC2会创建并附加一个与主子网关联的主ENI。主子网可以是公有或私有的。在hostNetwork模式下运行的Pod使用分配给节点主ENI的主IP地址,并与主机共享相同的网络命名空间。

CNI插件管理节点上的[弹性网络接口(ENI)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html)。当节点配置时,CNI插件会自动从节点子网中分配一个插槽(IP或前缀)池到主ENI。这个池称为*预热池*,其大小由节点的实例类型决定。根据CNI设置,插槽可以是IP地址或前缀。当ENI上的插槽被分配后,CNI可能会附加额外的ENI,并为其分配预热池插槽。这些额外的ENI称为辅助ENI。每个ENI只能支持一定数量的插槽,这取决于实例类型。CNI会根据所需的插槽数量(通常对应于Pod数量)附加更多ENI到实例。这个过程会一直持续,直到节点无法再支持额外的ENI。CNI还会预先分配"预热"ENI和插槽,以加快Pod启动速度。请注意,每种实例类型都有最大可附加的ENI数量。这是Pod密度(每个节点的Pod数量)的一个限制因素,除了计算资源之外。

![流程图说明了需要新的ENI委托前缀时的过程](./image.png)
[简体中文翻译]

每个 EC2 实例支持的网络接口数量和插槽数量因实例类型而有所不同。由于每个 Pod 都会在一个插槽上占用一个 IP 地址,因此您可以在特定 EC2 实例上运行的 Pod 数量取决于可以附加到该实例的 ENI 数量以及每个 ENI 支持的插槽数量。我们建议将 EKS 用户指南中的最大 Pod 数设置为避免耗尽实例的 CPU 和内存资源。使用 `hostNetwork` 的 Pod 不包括在此计算中。您可以考虑使用名为 [max-pod-calculator.sh](https://github.com/awslabs/amazon-eks-ami/blob/master/files/max-pods-calculator.sh) 的脚本来计算给定实例类型的 EKS 建议最大 Pod 数。

## 概述

辅助 IP 模式是 VPC CNI 的默认模式。本指南提供了在启用辅助 IP 模式时 VPC CNI 行为的一般概述。ipamd 的功能(IP 地址分配)可能会根据 VPC CNI 的配置设置而有所不同,例如[前缀模式](../prefix-mode/index_linux.md)、[每个 Pod 的安全组](../sgpp/index.md)和[自定义网络](../custom-networking/index.md)。

Amazon VPC CNI 作为名为 aws-node 的 Kubernetes Daemonset 部署在工作节点上。当工作节点配置时,它会附加一个默认的 ENI,称为主 ENI。CNI 会分配一个 ENI 和附加到节点主 ENI 的子网的辅助 IP 地址的温池。默认情况下,ipamd 会尝试为节点分配另一个 ENI。当调度一个 Pod 并从主 ENI 分配一个辅助 IP 地址时,IPAMD 会分配另一个 ENI。这个"温"ENI 可以更快地实现 Pod 网络。当辅助 IP 地址池耗尽时,CNI 会添加另一个 ENI 来分配更多地址。

ENI 和 IP 地址池的数量通过名为 [WARM_ENI_TARGET、WARM_IP_TARGET、MINIMUM_IP_TARGET](https://github.com/aws/amazon-vpc-cni-k8s/blob/master/docs/eni-and-ip-target.md) 的环境变量进行配置。`aws-node` Daemonset 将定期检查是否附加了足够数量的 ENI。当满足所有 `WARM_ENI_TARGET` 或 `WARM_IP_TARGET` 和 `MINIMUM_IP_TARGET` 条件时,就认为附加了足够数量的 ENI。如果附加的 ENI 不足,CNI 将调用 EC2 API 附加更多 ENI,直到达到 `MAX_ENI` 限制。

* `WARM_ENI_TARGET` - 整数,值 >0 表示要求已启用
  * 要维护的温 ENI 数量。当 ENI 作为辅助 ENI 附加到节点但未被任何 Pod 使用时,它就是"温"的。更具体地说,ENI 的 IP 地址都未与 Pod 关联。

* 示例：考虑一个具有 2 个 ENI 的实例,每个 ENI 支持 5 个 IP 地址。WARM_ENI_TARGET 设置为 1。如果实例关联了恰好 5 个 IP 地址,CNI 将保持 2 个 ENI 附加到实例。第一个 ENI 正在使用,其所有 5 个可能的 IP 地址都在使用。第二个 ENI 处于"温暖"状态,其所有 5 个 IP 地址都在池中。如果在该实例上启动另一个 Pod,将需要第 6 个 IP 地址。CNI 将从第二个 ENI 的池中分配此第 6 个 Pod 的 IP 地址。第二个 ENI 现在正在使用,不再处于"温暖"状态。CNI 将分配第 3 个 ENI 以保持至少 1 个温暖 ENI。

!!! 注意
    温暖 ENI 仍然会消耗来自您的 VPC CIDR 的 IP 地址。IP 地址在与工作负载(如 Pod)关联之前处于"未使用"或"温暖"状态。

* `WARM_IP_TARGET`，整数，值 >0 表示要求已启用
  * 要维护的温暖 IP 地址数量。温暖 IP 可在活动附加的 ENI 上使用,但尚未分配给 Pod。换句话说,可用的温暖 IP 数量是可以分配给 Pod 而无需附加其他 ENI 的 IP 数量。
  * 示例：考虑一个具有 1 个 ENI 的实例,每个 ENI 支持 20 个 IP 地址。WARM_IP_TARGET 设置为 5。WARM_ENI_TARGET 设置为 0。只有在需要第 16 个 IP 地址时,才会附加第二个 ENI,从子网 CIDR 中消耗 20 个可能的地址。
* `MINIMUM_IP_TARGET`，整数，值 >0 表示要求已启用
  * 任何时候都要分配的最小 IP 地址数。这通常用于在实例启动时预先分配多个 ENI。
  * 示例：考虑一个新启动的实例。它有 1 个 ENI,每个 ENI 支持 10 个 IP 地址。MINIMUM_IP_TARGET 设置为 100。ENI 立即附加 9 个其他 ENI,总共分配 100 个地址。这种情况发生,不管 WARM_IP_TARGET 或 WARM_ENI_TARGET 的值如何。

本项目包括一个[子网计算器 Excel 文档](../subnet-calc/subnet-calc.xlsx)。该计算器文档模拟了在不同 ENI 配置选项(如 `WARM_IP_TARGET` 和 `WARM_ENI_TARGET`)下指定工作负载的 IP 地址消耗情况。

![分配 IP 地址到 pod 的组件示意图](./image-2.png)

当 Kubelet 收到添加 Pod 的请求时,CNI 二进制文件会查询 ipamd 以获取可用的 IP 地址,然后 ipamd 将其提供给 Pod。CNI 二进制文件连接主机和 Pod 网络。

默认情况下,部署在节点上的 Pod 被分配到与主 ENI 相同的安全组。或者,Pod 也可以配置为使用不同的安全组。

![分配 IP 地址到 pod 的第二个组件示意图](./image-3.png)

作为 IP 地址池被耗尽,插件会自动将另一个弹性网络接口附加到实例并为该接口分配另一组辅助 IP 地址。这个过程一直持续到节点无法再支持额外的弹性网络接口。

![third illustration of components involved in assigning an IP address to a pod](./image-4.png)

当 Pod 被删除时,VPC CNI 将 Pod 的 IP 地址放入 30 秒的冷却缓存。冷却缓存中的 IP 地址不会分配给新的 Pod。冷却期结束后,VPC CNI 将 Pod IP 移回温池。冷却期可防止 Pod IP 地址过早回收,并允许集群中所有节点上的 kube-proxy 完成更新 iptables 规则。当 IP 或 ENI 的数量超过温池设置的数量时,ipamd 插件会将 IP 和 ENI 返回给 VPC。

如上所述的辅助 IP 模式中,每个 Pod 从附加到实例的 ENI 之一获得一个辅助私有 IP 地址。由于每个 Pod 使用一个 IP 地址,因此您可以在特定 EC2 实例上运行的 Pod 数量取决于可以附加到该实例的 ENI 数量以及它支持的 IP 地址数量。VPC CNI 检查 [limits](https://github.com/aws/amazon-vpc-resource-controller-k8s/blob/master/pkg/aws/vpc/limits.go) 文件,以了解每种实例类型允许的 ENI 和 IP 地址数量。

您可以使用以下公式来确定可以部署在节点上的最大 Pod 数量。

`(实例类型的网络接口数量 × (每个网络接口的 IP 地址数量 - 1)) + 2`

+2 表示需要主机网络的 Pod,例如 kube-proxy 和 VPC CNI。Amazon EKS 要求 kube-proxy 和 VPC CNI 在每个节点上运行,这些要求已纳入 max-pods 值。如果您想运行其他主机网络 Pod,请考虑更新 max-pods 值。

+2 表示使用主机网络的 Kubernetes Pod,例如 kube-proxy 和 VPC CNI。Amazon EKS 要求 kube-proxy 和 VPC CNI 在每个节点上运行,并且这些要求已计入 max-pods。如果您计划运行更多主机网络 Pod,请考虑更新 max-pods。您可以在启动模板的用户数据中指定 `--kubelet-extra-args "—max-pods=110"`。

例如,在一个由 3 个 c5.large 节点(3 个 ENI 和每个 ENI 最多 10 个 IP)组成的集群中,当集群启动并有 2 个 CoreDNS pod 时,CNI 将消耗 49 个 IP 地址并将它们保留在温池中。温池可以加快应用程序部署时的 Pod 启动。

节点 1(带有 CoreDNS pod): 2 个 ENI,分配 20 个 IP

节点 2(带有 CoreDNS pod): 2 个 ENI,分配 20 个 IP

节点 3(无 Pod): 1 个 ENI,分配 10 个 IP

请记住,基础设施 pod(通常作为守护进程集运行)每个都会对 max-pod 计数产生贡献。这些可能包括:

* CoreDNS
* Amazon Elastic LoadBalancer
* 用于指标服务器的操作 pod
我们建议您通过结合这些 Pod 的容量来规划您的基础设施。有关每种实例类型支持的最大 Pod 数量的列表,请参见 GitHub 上的 [eni-max-Pods.txt](https://github.com/awslabs/amazon-eks-ami/blob/master/files/eni-max-pods.txt)。

![illustration of multiple ENIs attached to a node](./image-5.png)

## 建议

### 部署 VPC CNI 托管插件

在配置集群时,Amazon EKS 会自动安装 VPC CNI。但是,Amazon EKS 支持托管插件,使集群能够与底层 AWS 资源(如计算、存储和网络)进行交互。我们强烈建议您使用包括 VPC CNI 在内的托管插件部署集群。

Amazon EKS 托管插件提供 VPC CNI 的安装和管理功能,适用于 Amazon EKS 集群。Amazon EKS 插件包含最新的安全补丁和错误修复,并经 AWS 验证可与 Amazon EKS 配合使用。VPC CNI 插件使您能够持续确保 Amazon EKS 集群的安全性和稳定性,并减少安装、配置和更新插件所需的工作量。此外,托管插件可通过 Amazon EKS API、AWS Management Console、AWS CLI 和 eksctl 进行添加、更新或删除。

您可以使用 `kubectl get` 命令的 `--show-managed-fields` 标志查找 VPC CNI 的托管字段。
```
kubectl get daemonset aws-node --show-managed-fields -n kube-system -o yaml
```

托管加载项通过每 15 分钟自动覆盖配置来防止配置漂移。这意味着通过 Kubernetes API 在加载项创建后对托管加载项进行的任何更改都将被自动漂移预防过程覆盖,并且在加载项更新过程中也将设置为默认值。

EKS 管理的字段列在 managedFields 下,管理者为 EKS。EKS 管理的字段包括服务帐户、镜像、镜像 URL、活性探针、就绪探针、标签、卷和卷挂载。

!!! 信息
最常用的字段,如 WARM_ENI_TARGET、WARM_IP_TARGET 和 MINIMUM_IP_TARGET,不受管理,不会被协调。对这些字段的更改将在更新加载项时得到保留。

我们建议在更新生产集群之前,在非生产集群中测试特定配置的加载项行为。此外,请遵循 EKS 用户指南中的步骤进行[加载项](https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html)配置。

#### 迁移到托管加载项

您将管理自管理 VPC CNI 的版本兼容性和安全补丁更新。要更新自管理加载项,您必须使用 Kubernetes API 和 [EKS 用户指南](https://docs.aws.amazon.com/eks/latest/userguide/managing-vpc-cni.html#updating-vpc-cni-add-on)中概述的说明。我们建议迁移到托管加载项以用于现有的 EKS 集群,并强烈建议在迁移之前备份当前的 CNI 设置。要配置托管加载项,您可以利用 Amazon EKS API、AWS Management Console 或 AWS Command Line Interface。
```
kubectl apply view-last-applied daemonset aws-node -n kube-system > aws-k8s-cni-old.yaml
```


亚马逊 EKS 将替换 CNI 配置设置,如果该字段被列为默认设置的托管字段。我们警告不要修改托管字段。该附加组件不会协调诸如 *warm* 环境变量和 CNI 模式等配置字段。在您迁移到托管 CNI 时,Pods 和应用程序将继续运行。

#### 更新前备份 CNI 设置

VPC CNI 在客户数据平面(节点)上运行,因此当发布新版本或您[更新集群](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html)到新的 Kubernetes 次版本后,亚马逊 EKS 不会自动更新该附加组件(托管和自管理)。要更新现有集群的附加组件,您必须通过 update-addon API 触发更新或在 EKS 控制台的附加组件中单击"立即更新"链接。如果您已部署自管理的附加组件,请遵循[更新自管理 VPC CNI 附加组件](https://docs.aws.amazon.com/eks/latest/userguide/managing-vpc-cni.html#updating-vpc-cni-add-on)中提到的步骤。

我们强烈建议您一次更新一个次版本。例如,如果您当前的次版本是 `1.9`,并且您想要更新到 `1.11`,您应该先更新到 `1.10` 的最新补丁版本,然后再更新到 `1.11` 的最新补丁版本。

在更新亚马逊 VPC CNI 之前,请检查 aws-node Daemonset。备份现有设置。如果使用托管附加组件,请确认您没有更新任何亚马逊 EKS 可能覆盖的设置。我们建议在自动化工作流程中添加更新后的挂钩或在附加组件更新后手动应用步骤。
```
kubectl apply view-last-applied daemonset aws-node -n kube-system > aws-k8s-cni-old.yaml
```


对于自管理的附加组件,请将备份与 GitHub 上的 `releases` 进行比较,以查看可用版本并熟悉您想要更新到的版本中的更改。我们建议使用 Helm 来管理自管理的附加组件,并利用值文件来应用设置。任何涉及 Daemonset 删除的更新操作都将导致应用程序停机,必须避免。

### 了解安全上下文

我们强烈建议您了解为有效管理 VPC CNI 而配置的安全上下文。Amazon VPC CNI 有两个组件 CNI 二进制文件和 ipamd (aws-node) Daemonset。CNI 作为二进制文件在节点上运行,并可访问节点根文件系统,还具有特权访问权限,因为它在节点级别处理 iptables。当 Pod 添加或删除时,kubelet 会调用 CNI 二进制文件。

aws-node Daemonset 是一个长期运行的进程,负责在节点级别管理 IP 地址。aws-node 在 `hostNetwork` 模式下运行,并允许访问回环设备和同一节点上其他 pod 的网络活动。aws-node init-container 在特权模式下运行,并挂载 CRI 套接字,允许 Daemonset 监控节点上运行的 Pod 的 IP 使用情况。Amazon EKS 正在努力消除 aws-node init 容器的特权要求。此外,aws-node 需要更新 NAT 条目并加载 iptables 模块,因此以 NET_ADMIN 特权运行。

Amazon EKS 建议部署 aws-node 清单定义的安全策略,用于 Pod 的 IP 管理和网络设置。请考虑更新到最新版本的 VPC CNI。此外,如果您有特定的安全要求,请考虑在 [GitHub 问题](https://github.com/aws/amazon-vpc-cni-k8s/issues)中提出。

### 为 CNI 使用单独的 IAM 角色

AWS VPC CNI 需要 AWS 身份和访问管理 (IAM) 权限。在使用 IAM 角色之前,需要设置 CNI 策略。您可以使用 [`AmazonEKS_CNI_Policy`](https://console.aws.amazon.com/iam/home#/policies/arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy%24jsonEditor),这是一个用于 IPv4 集群的 AWS 托管策略。AmazonEKS CNI 托管策略仅包含 IPv4 集群的权限。您必须为 IPv6 集群创建一个单独的 IAM 策略,其中包含[此处](https://docs.aws.amazon.com/eks/latest/userguide/cni-iam-role.html#cni-iam-role-create-ipv6-policy)列出的权限。

默认情况下,VPC CNI 继承 [Amazon EKS 节点 IAM 角色](https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html)(托管和自管理节点组)。

强烈建议配置一个单独的 IAM 角色,并为 Amazon VPC CNI 应用相关策略。否则,Amazon VPC CNI 的 pod 将获得分配给节点 IAM 角色的权限,并可访问分配给节点的实例配置文件。
[简体中文]

VPC CNI插件创建并配置了一个名为aws-node的服务帐户。默认情况下,该服务帐户绑定到附有Amazon EKS CNI策略的Amazon EKS节点IAM角色。为了使用单独的IAM角色,我们建议您[创建一个新的服务帐户](https://docs.aws.amazon.com/eks/latest/userguide/cni-iam-role.html#cni-iam-role-create-role)并附加Amazon EKS CNI策略。要使用新的服务帐户,您必须[重新部署CNI pod](https://docs.aws.amazon.com/eks/latest/userguide/cni-iam-role.html#cni-iam-role-redeploy-pods)。在创建新集群时,请考虑为VPC CNI托管加载项指定`--service-account-role-arn`。确保从Amazon EKS节点角色中删除IPv4和IPv6的Amazon EKS CNI策略。

建议您[阻止对实例元数据的访问](https://aws.github.io/aws-eks-best-practices/security/docs/iam/#restrict-access-to-the-instance-profile-assigned-to-the-worker-node),以最小化安全漏洞的影响范围。

### 处理存活/就绪探测失败

我们建议增加存活和就绪探测超时值(默认为`timeoutSeconds: 10`)以防止EKS 1.20及更高版本集群中的探测失败导致应用程序的Pod陷入containerCreating状态。这个问题在数据密集型和批处理集群中有所发现。高CPU使用会导致aws-node探测健康失败,从而无法满足Pod的CPU请求。除了修改探测超时,还要确保正确配置aws-node的CPU资源请求(默认为`CPU: 25m`)。除非您的节点出现问题,否则我们不建议更改这些设置。

我们强烈建议您在与Amazon EKS支持人员交流时,在节点上运行`sudo bash /opt/cni/bin/aws-cni-support.sh`。该脚本将帮助评估节点上的kubelet日志和内存利用率。请考虑在Amazon EKS工作节点上安装SSM代理以运行该脚本。

### 在非EKS优化AMI实例上配置IPTables转发策略

如果您使用自定义AMI,请确保在[kubelet.service](https://github.com/awslabs/amazon-eks-ami/blob/master/files/kubelet.service#L8)中将iptables转发策略设置为ACCEPT。许多系统将iptables转发策略设置为DROP。您可以使用[HashiCorp Packer](https://packer.io/intro/why.html)和来自[Amazon EKS AMI存储库on AWS GitHub](https://github.com/awslabs/amazon-eks-ami)的构建规范和配置脚本来构建自定义AMI。您可以更新[kubelet.service](https://github.com/awslabs/amazon-eks-ami/blob/master/files/kubelet.service#L8),并按照[此处](https://aws.amazon.com/premiumsupport/knowledge-center/eks-custom-linux-ami/)指定的说明创建自定义AMI。

### 定期升级CNI版本
VPC CNI 具有向后兼容性。最新版本适用于所有 Amazon EKS 支持的 Kubernetes 版本。此外,VPC CNI 作为 EKS 附加组件提供(请参见上文"部署 VPC CNI 托管附加组件")。虽然 EKS 附加组件编排附加组件的升级,但它不会自动升级像 CNI 这样的附加组件,因为它们在数据平面上运行。您负责在托管和自管理的工作节点升级之后升级 VPC CNI 附加组件。
