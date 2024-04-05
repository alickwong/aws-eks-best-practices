
# 已知限制和服务配额
Amazon EKS 可用于各种工作负载,并可与广泛的 AWS 服务进行交互,我们已经看到客户工作负载遇到了类似范围的 AWS 服务配额和其他问题,这些问题阻碍了可扩展性。

您的 AWS 帐户有默认配额(您的团队可以请求的每种 AWS 资源的上限)。每项 AWS 服务都定义了自己的配额,配额通常是特定于区域的。您可以请求增加某些配额(软限制),而其他配额无法增加(硬限制)。在设计应用程序时,您应该考虑这些值。请定期查看这些服务限制,并在应用程序设计过程中将其纳入考虑。

您可以在 [AWS 服务配额控制台](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-resource-limits.html#request-increase)或使用 [AWS CLI](https://repost.aws/knowledge-center/request-service-quota-increase-cli) 查看您帐户中的使用情况并提出配额增加请求。有关服务配额的更多详细信息以及任何进一步的限制或通知,请参阅相应 AWS 服务的 AWS 文档。

!!! note
    [Amazon EKS 服务配额](https://docs.aws.amazon.com/eks/latest/userguide/service-quotas.html)列出了服务配额,并提供了在可用的情况下请求增加的链接。

## 其他 AWS 服务配额
我们已经看到 EKS 客户受到下面列出的其他 AWS 服务配额的影响。这些可能只适用于特定的用例或配置,但是您可能会考虑您的解决方案在扩展时是否会遇到这些问题。这些配额按服务组织,每个配额都有一个格式为 L-XXXXXXXX 的 ID,您可以在 [AWS 服务配额控制台](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-resource-limits.html#request-increase)中查找。

| 服务 | 配额 (L-xxxxx) | **影响** | **ID (L-xxxxx)** | 默认值 |
| ---- | -------------- | --------- | ---------------- | ------ |
| IAM  | 每个帐户的角色  | 可能限制帐户中的集群或 IRSA 角色数量。 | L-FE177D64 | 1,000 |
| IAM  | 每个帐户的 OpenId 连接提供商 | 可能限制每个帐户的集群数量,IRSA 使用 OpenID Connect | L-858F3967 | 100 |

| IAM            | 角色信任策略长度                                                                   | 可以限制IRSA关联的集群数量                                        | L-C07B4B0D       | 2,048   |
| VPC            | 每个网络接口的安全组                                                      | 可以限制集群的网络控制或连接性                                           | L-2AFB9258       | 5       |
| VPC            | 每个VPC的IPv4 CIDR块                                                                   | 可以限制EKS Worker节点的数量                                                                           | L-83CA0A9D       | 5       |
| VPC            | 每个路由表的路由                                                                     | 可以限制集群的网络控制或连接性                                           | L-93826ACB       | 50      |
| VPC            | 每个VPC的活跃VPC对等连接                                                     | 可以限制集群的网络控制或连接性                                           | L-7E9ECCDB       | 50      |
| VPC            | 每个安全组的入站或出站规则                                              | 可以限制集群的网络控制或连接性,EKS中的某些控制器会创建新规则 | L-0EA8095F       | 50      |
| VPC            | 每个区域的VPC                                                                            | 可以限制每个账户的集群数量或集群的网络控制或连接性     | L-F678F1CE       | 5       |
| VPC            | 每个区域的互联网网关                                                               | 可以限制每个账户的集群数量或集群的网络控制或连接性     | L-A4707A72       | 5       |
| VPC            | 每个区域的网络接口                                                              | 可以限制EKS Worker节点的数量,或影响EKS控制平面的扩展/更新活动.                   | L-DF5E4CA3       | 5,000   |
| VPC            | 网络地址使用                                                                      | 可以限制每个账户的集群数量或集群的网络控制或连接性     | L-BB24F6E5       | 64,000  |
| VPC            | 对等网络地址使用                                                               | 可以限制每个账户的集群数量或集群的网络控制或连接性     | L-CD17FD4B       | 128,000 |
| ELB            | 每个网络负载均衡器的侦听器                                                                 | 可以限制对集群的入口流量的控制。                                                                         | L-57A373D6       | 50      |
| ELB            | 每个区域的目标组                                                                        | 可以限制对集群的入口流量的控制。                                                                         | L-B22855CB       | 3,000   |
| ELB            | 每个应用程序负载均衡器的目标                                                                | 可以限制对集群的入口流量的控制。                                                                         | L-7E6692B2       | 1,000   |
| ELB            | 每个网络负载均衡器的目标                                                                   | 可以限制对集群的入口流量的控制。                                                                         | L-EEF1AD04       | 3,000   |
| ELB            | 每个网络负载均衡器每个可用区的目标                                                             | 可以限制对集群的入口流量的控制。                                                                         | L-B211E961       | 500     |
| ELB            | 每个区域每个目标组的目标                                                                   | 可以限制对集群的入口流量的控制。                                                                         | L-A0D0B863       | 1,000   |
| ELB            | 每个区域的应用程序负载均衡器                                                                 | 可以限制对集群的入口流量的控制。                                                                         | L-53DA6B97       | 50      |
| ELB            | 每个区域的经典负载均衡器                                                                   | 可以限制对集群的入口流量的控制。                                                                         | L-E9E9831D       | 20      |
| ELB            | 每个区域的网络负载均衡器                                                                   | 可以限制对集群的入口流量的控制。                                                                         | L-69A177A2       | 50      |
| EC2            | 运行标准(A、C、D、H、I、M、R、T、Z)按需实例(最大vCPU数)                                        | 可以限制EKS工作节点的数量                                                                             | L-1216C47A       | 5       |
| EC2            | 所有标准(A、C、D、H、I、M、R、T、Z)Spot实例请求(最大vCPU数)                                    | 可以限制EKS工作节点的数量                                                                             | L-34B43A08       | 5       |

| EC2            | EC2-VPC 弹性 IP                                                                           | 可以限制 NAT 网关的数量(因此也限制了 VPC 的数量),这可能会限制某个区域内集群的数量                | L-0263D0A3       | 5       |
| EBS            | 每个区域的快照                                                                            | 可以限制有状态工作负载的备份策略                                                               | L-309BACF6       | 100,000 |
| EBS            | 通用型 SSD (gp3) 卷的存储空间,以 TiB 为单位                                               | 可以限制 EKS 工作节点的数量,或 PersistentVolume 存储空间                                       | L-7A658B76       | 50      |
| EBS            | 通用型 SSD (gp2) 卷的存储空间,以 TiB 为单位                                               | 可以限制 EKS 工作节点的数量,或 PersistentVolume 存储空间                                        | L-D18FCD1D       | 50      |
| ECR            | 注册的仓库                                                                               | 可以限制集群中工作负载的数量                                                                 | L-CFEB8E8D       | 10,000  |
| ECR            | 每个仓库的镜像                                                                           | 可以限制集群中工作负载的数量                                                                 | L-03A36CE1       | 10,000  |
| SecretsManager | 每个区域的密钥                                                                           | 可以限制集群中工作负载的数量                                                                 | L-2F66C23C       | 500,000 |


## AWS 请求限制

AWS 服务还实施了请求限制,以确保它们保持高性能和可用性,以供所有客户使用。与服务配额类似,每个 AWS 服务都维护自己的请求限制阈值。如果您的工作负载需要快速发出大量 API 调用,或者您在应用程序中遇到请求限制错误,请考虑查看相应的 AWS 服务文档。

在大型集群或集群急剧扩展时,与配置 EC2 网络接口或 IP 地址相关的 EC2 API 请求可能会遇到请求限制。下表显示了我们看到客户遇到请求限制的一些 API 操作。
您可以在 [EC2 文档中关于速率限制的部分](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/throttling.html)查看 EC2 速率限制默认值和请求速率限制增加的步骤。

| 可变操作                        | 只读操作                        |
| ------------------------------- | ------------------------------- |
| AssignPrivateIpAddresses        | DescribeDhcpOptions             |

| 附加网络接口 | 描述实例 |
| 创建网络接口 | 描述网络接口 |
| 删除网络接口 | 描述安全组 |
| 删除标签 | 描述标签 |
| 分离网络接口 | 描述VPC |
| 修改网络接口属性 | 描述卷 |
| 取消分配私有IP地址 | |

## 其他已知限制

* Route 53 DNS解析器的限制为[每秒1024个数据包](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-dns.html#vpc-dns-limits)。当大型集群的DNS流量通过少量的CoreDNS Pod副本时,可能会遇到此限制。[扩展CoreDNS并优化DNS行为](../cluster-services/#scale-coredns)可以避免DNS查找超时。
    * [Route 53 API的请求速率限制也相当低,为每秒5次请求](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/DNSLimitations.html#limits-api-requests)。如果您有大量域名需要使用外部DNS等项目进行更新,可能会遇到速率限制和域名更新延迟。

* 某些[Nitro实例类型的卷附加限制为28](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/volume_limits.html#instance-type-volume-limits),这个限制包括Amazon EBS卷、网络接口和NVMe实例存储卷。如果您的工作负载挂载了大量的EBS卷,可能会遇到pod密度受限的情况。

* 每个EC2实例可跟踪的最大连接数有限。[如果您的工作负载处理大量连接,可能会遇到通信失败或错误,因为已达到此最大值。](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/security-group-connection-tracking.html#connection-tracking-throttling)您可以使用`conntrack_allowance_available`和`conntrack_allowance_exceeded`[网络性能指标来监控EKS工作节点上的跟踪连接数](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-network-performance-ena.html)。

* 在EKS环境中,etcd存储限制为**8 GiB**,这是根据[上游指南](https://etcd.io/docs/v3.5/dev-guide/limit/#storage-size-limit)确定的。请监控指标`etcd_db_total_size_in_bytes`来跟踪etcd数据库大小。您可以参考[警报规则](https://github.com/etcd-io/etcd/blob/main/contrib/mixin/mixin.libsonnet#L213-L240)`etcdBackendQuotaLowSpace`和`etcdExcessiveDatabaseGrowth`来设置此监控。
