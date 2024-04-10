# Amazon VPC CNI

<iframe width="560" height="315" src="https://www.youtube.com/embed/RBE3yk2UlYA" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>

Amazon EKS通过[Amazon VPC Container Network Interface](https://github.com/aws/amazon-vpc-cni-k8s)[(VPC CNI)](https://github.com/aws/amazon-vpc-cni-k8s)插件实现集群网络。CNI插件允许Kubernetes Pods拥有与VPC网络上相同的IP地址。更具体地说,Pod内的所有容器共享一个网络命名空间,它们可以使用本地端口相互通信。

Amazon VPC CNI有两个组件:

* CNI二进制文件,用于设置Pod网络以实现Pod到Pod的通信。CNI二进制文件在节点根文件系统上运行,当新的Pod添加到节点或从节点中删除现有Pod时,kubelet会调用它。
* ipamd,一个长期运行的节点本地IP地址管理(IPAM)守护进程,负责:
  * 管理节点上的ENI,以及
  * 维护可用IP地址或前缀的预热池

当创建实例时,EC2会创建并附加一个与主子网关联的主ENI。主子网可以是公有或私有的。在hostNetwork模式下运行的Pods使用分配给节点主ENI的主IP地址,并与主机共享相同的网络命名空间。

CNI插件管理节点上的[弹性网络接口(ENI)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html)。当节点配置时,CNI插件会自动从节点的子网中为主ENI分配一个IP地址池。这个池被称为*预热池*,其大小由节点的实例类型决定。根据CNI设置,一个槽可以是一个IP地址或一个前缀。当一个ENI上的槽被分配后,CNI可能会附加额外的ENI,并为这些ENI分配预热池。这些额外的ENI称为辅助ENI。每个ENI只能支持一定数量的槽,这取决于实例类型。CNI会根据需要的槽数量(通常对应于Pods的数量)附加更多ENI到实例上。这个过程一直持续,直到节点无法再支持额外的ENI。CNI还会预先分配"预热"ENI和槽,以加快Pod启动。请注意,每种实例类型都有最大的ENI数量。这是Pod密度(每个节点的Pod数量)的一个限制因素,除了计算资源之外。

![流程图说明了需要新的ENI委派前缀时的过程](./image.png)

每种EC2实例类型支持的网络接口数量和可使用的槽数量是不同的。由于每个Pod会消耗一个槽上的IP地址,因此您可以在特定的EC2实例上运行的Pods数量取决于可以附加到该实例的ENI数量以及每个ENI支持的槽数。我们建议将EKS用户指南中的最大Pods设置为上限,以避免耗尽实例的CPU和内存资源。使用`hostNetwork`的Pods不包括在此计算中。您可以考虑使用名为[max-pod-calculator.sh](https://github.com/awslabs/amazon-eks-ami/blob/master/files/max-pods-calculator.sh)的脚本来计算EKS推荐的特定实例类型的最大Pods数量。

## 概述

辅助IP模式是VPC CNI的默认模式。本指南提供了在启用辅助IP模式时VPC CNI行为的一般概述。ipamd的功能(IP地址分配)可能会根据VPC CNI的配置设置而有所不同,例如[前缀模式](../prefix-mode/index_linux.md)、[每个Pod的安全组](../sgpp/index.md)和[自定义网络](../custom-networking/index.md)。

Amazon VPC CNI被部署为名为aws-node的Kubernetes Daemonset,运行在工作节点上。当工作节点配置时,它会附加一个默认的ENI,称为主ENI。CNI从附加到节点主ENI的子网中分配一个预热池的ENI和辅助IP地址。默认情况下,ipamd会尝试为节点分配一个额外的ENI。当一个Pod被调度并从主ENI分配一个辅助IP地址时,IPAMD会分配另一个ENI。这个"预热"ENI可以实现更快的Pod网络。当辅助IP地址池耗尽时,CNI会添加另一个ENI来分配更多地址。

ENI和IP地址池的数量通过名为[WARM_ENI_TARGET、WARM_IP_TARGET、MINIMUM_IP_TARGET](https://github.com/aws/amazon-vpc-cni-k8s/blob/master/docs/eni-and-ip-target.md)的环境变量进行配置。`aws-node` Daemonset将定期检查是否有足够的ENI附加。当满足所有`WARM_ENI_TARGET`或`WARM_IP_TARGET`和`MINIMUM_IP_TARGET`条件时,就认为有足够的ENI附加。如果附加的ENI不足,CNI将调用EC2 API来附加更多ENI,直到达到`MAX_ENI`限制。

* `WARM_ENI_TARGET` - 整数,值>0表示要求启用
  * 要维护的预热ENI数量。当一个ENI作为辅助ENI附加到节点上,但没有被任何Pod使用时,它就处于"预热"状态。更具体地说,ENI的IP地址都没有被分配给Pod。
  * 例如:考虑一个实例有2个ENI,每个ENI支持5个IP地址。WARM_ENI_TARGET设置为1。如果正好有5个IP地址与该实例关联,CNI会维护2个附加到该实例的ENI。第一个ENI正在使用,所有5个可能的IP地址都在使用。第二个ENI是"预热"的,所有5个IP地址都在池中。如果在该实例上启动另一个Pod,需要第6个IP地址。CNI将从第二个ENI的池中分配这第6个IP地址。第二个ENI现在正在使用,不再处于"预热"状态。CNI将分配第3个ENI,以维持至少1个预热ENI。

!!! 注意
    预热ENI仍然会消耗来自VPC CIDR的IP地址。IP地址在被工作负载(如Pod)关联之前是"未使用"或"预热"的。

* `WARM_IP_TARGET`,整数,值>0表示要求启用
  * 要维护的预热IP地址数量。预热IP可用于已积极附加的ENI,但尚未分配给Pod。换句话说,可用的预热IP数量是可以分配给Pod而无需附加额外ENI的IP数量。
  * 例如:考虑一个实例有1个ENI,每个ENI支持20个IP地址。WARM_IP_TARGET设置为5。WARM_ENI_TARGET设置为0。只会附加1个ENI,直到需要第16个IP地址。然后,CNI将附加第二个ENI,从子网CIDR中消耗20个可能的地址。
* `MINIMUM_IP_TARGET`,整数,值>0表示要求启用
  * 任何时候都要分配的最小IP地址数量。这通常用于在实例启动时预先分配多个ENI。
  * 例如:考虑一个新启动的实例。它有1个ENI,每个ENI支持10个IP地址。MINIMUM_IP_TARGET设置为100。ENI立即附加9个更多ENI,总共有100个地址。这种情况发生,不管WARM_IP_TARGET或WARM_ENI_TARGET的值是什么。

该项目包括一个[子网计算器Excel文档](../subnet-calc/subnet-calc.xlsx)。该计算器文档模拟了在不同的ENI配置选项(如`WARM_IP_TARGET`和`WARM_ENI_TARGET`)下指定工作负载的IP地址消耗情况。

![说明分配IP地址到Pod的组件](./image-2.png)

当Kubelet收到添加Pod的请求时,CNI二进制文件会查询ipamd获取可用的IP地址,ipamd然后将其提供给Pod。CNI二进制文件连接主机和Pod网络。

默认情况下,部署在节点上的Pods被分配与主ENI相同的安全组。或者,Pods也可以配置不同的安全组。

![第二个说明分配IP地址到Pod的组件](./image-3.png)

当IP地址池耗尽时,插件会自动附加另一个弹性网络接口到实例,并为该接口分配另一组辅助IP地址。这个过程一直持续,直到节点无法再支持额外的弹性网络接口。

![第三个说明分配IP地址到Pod的组件](./image-4.png)

当一个Pod被删除时,VPC CNI会将Pod的IP地址放入一个30秒的冷却缓存。冷却缓存中的IP地址不会分配给新的Pods。冷却期结束后,VPC CNI会将Pod IP移回预热池。冷却期可以防止Pod IP地址过早被回收,并允许集群中所有节点上的kube-proxy更新iptables规则。当IP或ENI的数量超过预热池设置的数量时,ipamd插件会将IP和ENI返回给VPC。

如上所述的辅助IP模式中,每个Pod从附加到实例的ENI之一获得一个辅助私有IP地址。由于每个Pod使用一个IP地址,因此您可以在特定的EC2实例上运行的Pods数量取决于可以附加到该实例的ENI数量以及它支持的IP地址数量。VPC CNI会检查[限制](https://github.com/aws/amazon-vpc-resource-controller-k8s/blob/master/pkg/aws/vpc/limits.go)文件,以了解每种实例类型允许的ENI和IP地址数量。

您可以使用以下公式来确定可以部署在节点上的最大Pods数量。

`(实例类型的网络接口数量 × (每个网络接口的IP地址数量 - 1)) + 2`

+2表示使用主机网络的Pods,如kube-proxy和VPC CNI。Amazon EKS要求kube-proxy和VPC CNI在每个节点上运行,这些要求被纳入max-pods值。如果您想运行更多使用主机网络的Pods,请考虑更新max-pods值。

+2表示使用主机网络的Kubernetes Pods,如kube-proxy和VPC CNI。Amazon EKS要求kube-proxy和VPC CNI在每个节点上运行,并将其计算在max-pods中。如果您计划运行更多使用主机网络的Pods,请考虑更新max-pods。您可以在启动模板的用户数据中指定`--kubelet-extra-args "—max-pods=110"`。

例如,在一个有3个c5.large节点(3个ENI,每个ENI最多10个IP)的集群中,当集群启动并有2个CoreDNS Pods时,CNI将消耗49个IP地址并将它们保持在预热池中。预热池可以在部署应用程序时实现更快的Pod启动。

节点1(有CoreDNS Pod):2个ENI,分配20个IP

节点2(有CoreDNS Pod):2个ENI,分配20个IP

节点3(无Pod):1个ENI,分配10个IP。

请记住,作为Daemonset运行的基础设施Pods也会计入max-pod计数。这些可能包括:

* CoreDNS
* Amazon Elastic LoadBalancer
* 指标服务器的操作Pods

我们建议您通过结合这些Pods的容量来规划您的基础设施。有关每种实例类型支持的最大Pods数量,请参见GitHub上的[eni-max-Pods.txt](https://github.com/awslabs/amazon-eks-ami/blob/master/files/eni-max-pods.txt)。

![说明多个ENI附加到节点的插图](./image-5.png)

## 建议

### 部署VPC CNI托管加载项

在配置集群时,Amazon EKS会自动安装VPC CNI。但是,Amazon EKS支持托管加载项,使集群能够与底层AWS资源(如计算、存储和网络)进行交互。我们强烈建议您使用托管加载项(包括VPC CNI)部署集群。

Amazon EKS托管加载项提供VPC CNI的安装和管理功能。Amazon EKS加载项包含最新的安全补丁、错误修复,并经AWS验证可与Amazon EKS配合使用。VPC CNI加载项使您能够持续确保Amazon EKS集群的安全性和稳定性,并减少安装、配置和更新加载项所需的工作量。此外,托管加载项可通过Amazon EKS API、AWS Management Console、AWS CLI和eksctl添加、更新或删除。

您可以使用`--show-managed-fields`标志与`kubectl get`命令查找VPC CNI的托管字段。

```
kubectl get daemonset aws-node --show-managed-fields -n kube-system -o yaml
```

托管加载项通过每15分钟自动覆盖配置来防止配置偏差。这意味着,通过Kubernetes API在加载项创建后对托管加载项进行的任何更改,都将被自动防止偏差的过程覆盖,并在加载项更新过程中设置为默认值。

由EKS管理的字段列在managedFields下,manager为EKS。EKS管理的字段包括服务帐户、镜像、镜像URL、存活探测、就绪探测、标签、卷和卷挂载。

!!! 信息
最常用的字段,如WARM_ENI_TARGET、WARM_IP_TARGET和MINIMUM_IP_TARGET,不受管理,更新时不会被重置。对这些字段的更改将在加载项更新时保留。

我们建议在更新生产集群之前,先在非生产集群中测试加载项行为以了解特定配置。此外,请遵循EKS用户指南中关于[加载项](https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html)配置的步骤。

#### 迁移到托管加载项

您将管理自管理VPC CNI的版本兼容性和更新安全补丁。要更新自管理加载项,您必须使用Kubernetes API和[EKS用户指南](https://docs.aws.amazon.com/eks/latest/userguide/managing-vpc-cni.html#updating-vpc-cni-add-on)中概述的说明。我们建议在迁移到托管加载项之前,先备份当前的CNI设置。要配置托管加载项,您可以使用Amazon EKS API、AWS Management Console或AWS命令行界面。

```
kubectl apply view-last-applied daemonset aws-node -n kube-system > aws-k8s-cni-old.yaml
```

如果字段被列为托管,Amazon EKS将用默认设置替换CNI配置设置。我们警告不要修改托管字段。加载项不会协调诸如*预热*环境变量和CNI模式之类的配置字段。在迁移到托管CNI时,Pods和应用程序将继续运行。

#### 更新前备份CNI设置

VPC CNI在客户数据平面(节点)上运行,因此当发布新版本或[更新集群](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html)到新的Kubernetes次版本时,Amazon EKS不会自动更新加载项(托管和自管理)。要更新现有集群的加载项,您必须通过update-addon API或在EKS控制台上单击加载项的"更新"链接来触发更新。如果您部署了自管理加载项,请遵循[更新自管理VPC CNI加载项](https://docs.aws.amazon.com/eks/latest/userguide/managing-vpc-cni.html#updating-vpc-cni-add-on)中提到的步骤。

我们强烈建议您一次只更新一个次版本。例如,如果您当前的次版本是`1.9`,并且想要更新到`1.11`,您应该先更新到`1.10`的最新补丁版本,然后再更新到`1.11`的最新补丁版本。

在更新Amazon VPC CNI之前,请检查aws-node Daemonset。备份现有设置。如果使用托管加载项,请确认您没有更新任何Amazon EKS可能覆盖的设置。我们建议在自动化工作流程中添加更新后的挂钩,或在加载项更新后手动应用步骤。

```
kubectl apply view-last-applied daemonset aws-node -n kube-system > aws-k8s-cni-old.yaml
```

对于自管理加载项,请将备份与GitHub上的`releases`进行比较,以查看可用版本并熟悉您想要更新到的版本中的更改。我们建议使用Helm来管理自管理加载项,并利用值文件来应用设置。任何涉及Daemonset删除的更新操作都会导致应用程序停机,必须避免。

### 了解安全上下文

我们强烈建议您了解用于有效管理VPC CNI的配置的安全上下文。Amazon VPC CNI有两个组件:CNI二进制文件和ipamd(aws-node)Daemonset。CNI作为二进制文件在节点上运行,并访问节点根文件系统,同时还具有特权访问权限,因为它处理节点级别的iptables。当Pods被添加或删除时,kubelet会调用CNI二进制文件。

aws-node Daemonset是一个长期运行的进程,负责在节点级别管理IP地址。aws-node在`hostNetwork`模式下运行,允许访问回环设备和同一节点上其他Pods的网络活动。aws-node init容器以特权模式运行,并挂载CRI套接字,允许Daemonset监视在节点上运行的Pods的IP使用情况。Amazon EKS正在努力消除aws-node init容器的特权要求。此外,aws-node需要更新NAT条目,并加载iptables模块,因此以NET_ADMIN特权运行。

Amazon EKS建议部署如aws-node清单所定义的安全策略,用于Pods的IP管理和网络设置。请考虑更新到最新版本的VPC CNI。此外,如果您有特定的安全要求,请考虑在[GitHub问题](https://github.com/aws/amazon-vpc-cni-k8s/issues)中提出。

### 为CNI使用单独的IAM角色

AWS VPC CNI需要AWS Identity and Access Management (IAM)权限。在使用IAM角色之前,需要设置CNI策略。您可以使用[`AmazonEKS_CNI_Policy`](https://console.aws.amazon.com/iam/home#/policies/arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy%24jsonEditor),这是一个用于IPv4集群的AWS托管策略。AmazonEKS CNI托管策略只有IPv4集群的权限。您必须为IPv6集群创建一个单独的IAM策略,权限列表[在此](https://docs.aws.amazon.com/eks/latest/userguide/cni-iam-role.html#cni-iam-role-create-ipv6-policy)。

默认情况下,VPC CNI继承[Amazon EKS节点IAM角色](https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html)(托管和自管理节点组)。

**强烈**建议为Amazon VPC CNI配置一个单独的IAM角色。否则,Amazon VPC CNI的Pods将获得分配给节点IAM角色的权限,并访问分配给节点的实例配置文件。

VPC CNI插件创建并配置了一个名为aws-node的服务帐户。默认情况下,该服务帐户绑定到附有Amazon EKS CNI策略的Amazon EKS节点IAM角色。要使用单独的IAM角色,我们建议[创建一个新的服务帐户](https://docs.aws.amazon.com/eks/latest/userguide/cni-iam-role.html#cni-iam-role-create-role),并附加Amazon EKS CNI策略。要使用新的服务帐户,您必须[重新部署CNI Pods](https://docs.aws.amazon.com/eks/latest/userguide/cni-iam-role.html#cni-iam-role-redeploy-pods)。在创建新集群时,请考虑为VPC CNI托管加载项指定`--service-account-role-arn`。确保从Amazon EKS节点角色中删除Amazon EKS CNI策略,包括IPv4和IPv6。

建议[阻止对实例元数据的访问](https://aws.github.io/aws-eks-best-practices/security/docs/iam/#restrict-access-to-the-instance-profile-assigned-to-the-worker-node),以最小化安全漏洞的影响范围。

### 处理存活/就绪探测失败

我们建议增加存活和就绪探测超时值(默认`timeoutSeconds: 10`)以防止EKS 1.20及更高版本集群上的探测失败导致应用程序的Pod陷入containerCreating状态。这个问题在数据密集型和批处理集群中有所发现。高CPU使用会导致aws-node探测健康失败,从而无法满足Pod的CPU请求。除了修改探测超时,还要确保正确配置了aws-node的CPU资源请求(默认`CPU: 25m`)。除非您的节点出现问题,否则我们不建议更新这些设置。

我们强烈鼓励您在与Amazon EKS支持人员交流时,在节点上运行sudo `bash /opt/cni/bin/aws-cni-support.sh`。该脚本将帮助评估节点上的kubelet日志和内存利用率。请考虑在Amazon EKS工作节点上安装SSM Agent以运行该脚本。

### 在非EKS优化AMI实例上配置IPTables转发策略

如果您使用自定义AMI,请确保在[kubelet.service](https://github.com/awslabs/amazon-eks-ami/blob/master/files/kubelet.service#L8)下将iptables转发策略设置为ACCEPT。许多系统将iptables转发策略设置为DROP。您可以使用[HashiCorp Packer](https://packer.io/intro/why.html)和来自[AWS GitHub上的Amazon EKS AMI存储库](https://github.com/awslabs/amazon-eks-ami)的构建规范和配置脚本来构建自定义AMI。您可以更新[kubelet.service](https://github.com/awslabs/amazon-eks-ami/blob/master/files/kubelet.service#L8),并按照[此处](https://aws.amazon.com/premiumsupport/knowledge-center/eks-custom-linux-ami/)指定的说明创建自定义AMI。

### 定期升级CNI版本

VPC CNI是向后兼容的。最新版本适用于所有Amazon EKS支持的Kubernetes版本。此外,VPC CNI作为EKS加载项提供(请参见上面的"部署VPC CNI托管加载项")。虽然EKS加载项编排了加载项的升级,但它不会自动升级像CNI这样的加载项,因为它们运行在数据平面上。您有责任在工作节点升级后升级VPC CNI加载项。