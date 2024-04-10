# 已知限制和服务配额

Amazon EKS 可用于各种工作负载,并可与广泛的 AWS 服务进行交互,我们已看到客户工作负载遇到类似范围的 AWS 服务配额和其他问题,这些问题阻碍了可扩展性。

您的 AWS 帐户有默认配额(您的团队可以请求的每个 AWS 资源的上限)。每个 AWS 服务都定义了自己的配额,配额通常是特定于区域的。您可以请求增加某些配额(软限制),而其他配额无法增加(硬限制)。在设计应用程序时,您应该考虑这些值。请定期查看这些服务限制,并在应用程序设计过程中将其纳入考虑。

您可以在 [AWS 服务配额控制台](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-resource-limits.html#request-increase)或使用 [AWS CLI](https://repost.aws/knowledge-center/request-service-quota-increase-cli) 查看您帐户中的使用情况并提出配额增加请求。有关服务配额的更多详细信息以及任何进一步的限制或通知,请参阅相应 AWS 服务的 AWS 文档。

!!! note
    [Amazon EKS 服务配额](https://docs.aws.amazon.com/eks/latest/userguide/service-quotas.html)列出了服务配额,并提供了在可用的情况下请求增加的链接。

## 其他 AWS 服务配额

我们已经看到 EKS 客户受到下面列出的其他 AWS 服务配额的影响。这些可能只适用于特定的用例或配置,但是您可能会考虑您的解决方案在扩展时是否会遇到这些问题。这些配额按服务组织,每个配额都有一个格式为 L-XXXXXXXX 的 ID,您可以在 [AWS 服务配额控制台](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-resource-limits.html#request-increase)中查找。

| 服务        | 配额 (L-xxxxx)                                                                            | **影响**                                                                                                         | **ID (L-xxxxx)** | 默认值 |
| -------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------ | ---------------- | ------- |
| IAM            | 每个帐户的角色                                                                          | 可能限制帐户中的集群或 IRSA 角色数量。                                                                      | L-FE177D64       | 1,000   |
| IAM            | 每个帐户的 OpenId 连接提供程序                                                       | 可能限制每个帐户的集群数量,IRSA 使用 OpenID Connect                                                       | L-858F3967       | 100     |
| IAM            | 角色信任策略长度                                                                   | 可能限制 IAM 角色与 IRSA 关联的集群数量                                                                        | L-C07B4B0D       | 2,048   |
| VPC            | 每个网络接口的安全组                                                                      | 可能限制集群的网络控制或连接                                                                           | L-2AFB9258       | 5       |
| VPC            | 每个 VPC 的 IPv4 CIDR 块                                                                   | 可能限制 EKS 工作节点的数量                                                                                           | L-83CA0A9D       | 5       |
| VPC            | 每个路由表的路由                                                                     | 可能限制集群的网络控制或连接                                                                           | L-93826ACB       | 50      |
| VPC            | 每个 VPC 的活跃 VPC 对等连接                                                     | 可能限制集群的网络控制或连接                                                                           | L-7E9ECCDB       | 50      |
| VPC            | 每个安全组的入站或出站规则。                                              | 可能限制集群的网络控制或连接,EKS 中的某些控制器会创建新规则 | L-0EA8095F       | 50      |
| VPC            | 每个区域的 VPC                                                                            | 可能限制每个帐户的集群数量或集群的网络控制或连接                                                     | L-F678F1CE       | 5       |
| VPC            | 每个区域的互联网网关                                                               | 可能限制每个帐户的集群数量或集群的网络控制或连接                                                     | L-A4707A72       | 5       |
| VPC            | 每个区域的网络接口                                                              | 可能限制 EKS 工作节点的数量,或影响 EKS 控制平面的扩展/更新活动。                                   | L-DF5E4CA3       | 5,000   |
| VPC            | 网络地址使用量                                                                      | 可能限制每个帐户的集群数量或集群的网络控制或连接                                                     | L-BB24F6E5       | 64,000  |
| VPC            | 对等网络地址使用量                                                               | 可能限制每个帐户的集群数量或集群的网络控制或连接                                                     | L-CD17FD4B       | 128,000 |
| ELB            | 每个网络负载均衡器的侦听器                                                        | 可能限制对集群的流量入口的控制。                                                                           | L-57A373D6       | 50      |
| ELB            | 每个区域的目标组                                                                   | 可能限制对集群的流量入口的控制。                                                                           | L-B22855CB       | 3,000   |
| ELB            | 每个应用程序负载均衡器的目标                                                      | 可能限制对集群的流量入口的控制。                                                                           | L-7E6692B2       | 1,000   |
| ELB            | 每个网络负载均衡器的目标                                                          | 可能限制对集群的流量入口的控制。                                                                           | L-EEF1AD04       | 3,000   |
| ELB            | 每个网络负载均衡器每个可用区的目标                                                    | 可能限制对集群的流量入口的控制。                                                                           | L-B211E961       | 500     |
| ELB            | 每个区域每个目标组的目标                                                        | 可能限制对集群的流量入口的控制。                                                                           | L-A0D0B863       | 1,000   |
| ELB            | 每个区域的应用程序负载均衡器                                                      | 可能限制对集群的流量入口的控制。                                                                           | L-53DA6B97       | 50      |
| ELB            | 每个区域的经典负载均衡器                                                          | 可能限制对集群的流量入口的控制。                                                                           | L-E9E9831D       | 20      |
| ELB            | 每个区域的网络负载均衡器                                                          | 可能限制对集群的流量入口的控制。                                                                           | L-69A177A2       | 50      |
| EC2            | 正在运行的按需标准(A、C、D、H、I、M、R、T、Z)实例(最大 vCPU 数)| 可能限制 EKS 工作节点的数量                                                                           | L-1216C47A       | 5       |
| EC2            | 所有标准(A、C、D、H、I、M、R、T、Z)Spot 实例请求(最大 vCPU 数) | 可能限制 EKS 工作节点的数量                                                                           | L-34B43A08       | 5       |
| EC2            | EC2-VPC 弹性 IP                                                                        | 可能限制 NAT 网关(和 VPC)的数量,从而限制某个区域内的集群数量                                | L-0263D0A3       | 5       |
| EBS            | 每个区域的快照                                                                       | 可能限制有状态工作负载的备份策略                                                                               | L-309BACF6       | 100,000 |
| EBS            | 通用型 SSD(gp3)卷的存储,以 TiB 为单位                                      | 可能限制 EKS 工作节点的数量或 PersistentVolume 存储                                              | L-7A658B76       | 50      |
| EBS            | 通用型 SSD(gp2)卷的存储,以 TiB 为单位                                      | 可能限制 EKS 工作节点的数量或 PersistentVolume 存储                                             | L-D18FCD1D       | 50      |
| ECR            | 注册的存储库                                                                    | 可能限制集群中工作负载的数量                                                                                 | L-CFEB8E8D       | 10,000  |
| ECR            | 每个存储库的镜像                                                                      | 可能限制集群中工作负载的数量                                                                                 | L-03A36CE1       | 10,000  |
| SecretsManager | 每个区域的秘密                                                                         | 可能限制集群中工作负载的数量                                                                                 | L-2F66C23C       | 500,000 |

## AWS 请求节流

AWS 服务还实施请求节流,以确保它们保持高性能和可用性,以供所有客户使用。与服务配额类似,每个 AWS 服务都维护自己的请求节流阈值。如果您的工作负载需要快速发出大量 API 调用,或者如果您在应用程序中遇到请求节流错误,请查看相应的 AWS 服务文档。

围绕配置 EC2 网络接口或 IP 地址的 EC2 API 请求可能会在大型集群或集群急剧扩展时遇到请求节流。下表显示了我们看到客户遇到请求节流的一些 API 操作。
您可以在 [EC2 文档中关于速率限制的部分](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/throttling.html)查看 EC2 速率限制默认值和请求速率限制增加的步骤。

| 可变操作                | 只读操作(v2)   |
| ------------------------------- |---------------------------|
| AssignPrivateIpAddresses        | DescribeDhcpOptions       |
| AttachNetworkInterface          | DescribeInstances         |
| CreateNetworkInterface          | DescribeNetworkInterfaces |
| DeleteNetworkInterface          | DescribeSecurityGroups    |
| DeleteTags                      | DescribeTags              |
| DetachNetworkInterface          | DescribeVpcs              |
| ModifyNetworkInterfaceAttribute | DescribeVolumes           |
| UnassignPrivateIpAddresses      |                           |

## 其他已知限制和服务配额

* Route 53 DNS 解析器的限制为[每秒 1024 个数据包](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-dns.html#vpc-dns-limits)。当大型集群的 DNS 流量通过少量的 CoreDNS Pod 副本进行传输时,可能会遇到此限制。[扩展 CoreDNS 并优化 DNS 行为](../cluster-services/#scale-coredns)可以避免 DNS 查找超时。
    * [Route 53 API 的速率限制也相当低,为每秒 5 个请求](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/DNSLimitations.html#limits-api-requests)。如果您有大量域名需要使用外部 DNS 等项目进行更新,可能会遇到速率节流和域名更新延迟。

* 某些 [Nitro 实例类型有 28 个卷附加限制](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/volume_limits.html#instance-type-volume-limits),这个限制包括 Amazon EBS 卷、网络接口和 NVMe 实例存储卷。如果您的工作负载挂载了大量 EBS 卷,您可能会遇到使用这些实例类型实现 pod 密度的限制。

* 每个 Ec2 实例可跟踪的最大连接数有限。[如果您的工作负载处理大量连接,可能会遇到通信失败或错误,因为已达到此最大值。](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/security-group-connection-tracking.html#connection-tracking-throttling)您可以使用 `conntrack_allowance_available` 和 `conntrack_allowance_exceeded` [网络性能指标来监控 EKS 工作节点上跟踪的连接数](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-network-performance-ena.html)。

* 在 EKS 环境中,etcd 存储限制为 **8 GiB**,这是根据[上游指南](https://etcd.io/docs/v3.5/dev-guide/limit/#storage-size-limit)确定的。请监控指标 `etcd_db_total_size_in_bytes` 来跟踪 etcd db 大小。您可以参考[警报规则](https://github.com/etcd-io/etcd/blob/main/contrib/mixin/mixin.libsonnet#L213-L240)`etcdBackendQuotaLowSpace` 和 `etcdExcessiveDatabaseGrowth` 来设置此监控。