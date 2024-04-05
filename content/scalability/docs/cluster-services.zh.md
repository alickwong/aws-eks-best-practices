!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

# 集群服务

集群服务在 EKS 集群内运行,但它们不是用户工作负载。如果您有一台 Linux 服务器,通常需要运行诸如 NTP、syslog 和容器运行时等服务来支持您的工作负载。集群服务类似,支持帮助您自动化和操作集群的服务。在 Kubernetes 中,它们通常在 kube-system 命名空间中运行,有些则作为 [DaemonSets](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) 运行。

集群服务预计具有较高的正常运行时间,在停机和故障排除期间通常很关键。如果核心集群服务不可用,您可能会丢失有助于恢复或预防停机的数据(例如高磁盘利用率)。它们应该在专用计算实例上运行,如单独的节点组或 AWS Fargate。这将确保集群服务不会受到共享实例上可能扩展或使用更多资源的工作负载的影响。

## 扩展 CoreDNS

扩展 CoreDNS 有两种主要机制。减少对 CoreDNS 服务的调用次数和增加副本数量。

### 通过降低 ndots 来减少外部查询

ndots 设置指定域名中被视为足够避免 DNS 查询的句点(即"点")数。如果您的应用程序有一个 ndots 设置为 5(默认值),并且您请求来自外部域(如 api.example.com,有 2 个点)的资源,那么 CoreDNS 将被查询以获取在 /etc/resolv.conf 中定义的每个搜索域的更具体的域。默认情况下,在进行外部请求之前,将搜索以下域。
```
api.example.<namespace>.svc.cluster.local
api.example.svc.cluster.local
api.example.cluster.local
api.example.<region>.compute.internal
```

`命名空间`和`区域`值将被替换为您的工作负载命名空间和计算区域。根据您的集群设置,您可能有其他搜索域。

您可以通过[降低工作负载的ndots选项](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-dns-config)或完全限定域请求(例如`api.example.com.`)来减少对CoreDNS的请求数量。如果您的工作负载通过DNS连接到外部服务,我们建议将ndots设置为2,以便工作负载不会在集群内部进行不必要的集群DNS查询。如果工作负载不需要访问集群内部的服务,您可以设置不同的DNS服务器和搜索域。
```
spec:
  dnsPolicy: "None"
  dnsConfig:
    options:
      - name: ndots
        value: "2"
      - name: edns0
```


如果您将 ndots 降低到太低的值或您正在连接的域没有足够的特定性(包括尾部 .)，那么 DNS 查找可能会失败。请确保您测试了此设置将如何影响您的工作负载。

### 水平扩展 CoreDNS

可以通过添加更多副本来扩展 CoreDNS 实例。建议您使用 [NodeLocal DNS](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/) 或 [集群比例自动缩放器](https://github.com/kubernetes-sigs/cluster-proportional-autoscaler) 来扩展 CoreDNS。

NodeLocal DNS 将需要在每个节点上运行一个实例 - 作为 DaemonSet - 这需要在集群中投入更多的计算资源,但它将避免 DNS 请求失败并降低集群中 DNS 查询的响应时间。集群比例自动缩放器将根据集群中的节点数或内核数来扩展 CoreDNS。这与请求查询并不直接相关,但根据您的工作负载和集群大小可能会很有用。默认的比例缩放是在集群中每增加 256 个内核或 16 个节点时添加一个额外的副本,以先发生者为准。

## 垂直扩展 Kubernetes Metrics Server

Kubernetes Metrics Server 支持水平和垂直扩展。通过水平扩展 Metrics Server,它将具有高可用性,但它不会水平扩展以处理更多集群指标。您需要根据[他们的建议](https://kubernetes-sigs.github.io/metrics-server/#scaling)垂直扩展 Metrics Server,因为节点和收集的指标会添加到集群中。

Metrics Server 会保留它收集、聚合和提供的数据。随着集群的增长,Metrics Server 存储的数据量也会增加。在大型集群中,Metrics Server 将需要比默认安装中指定的内存和 CPU 预留更多的计算资源。您可以使用 [Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) (VPA) 或 [Addon Resizer](https://github.com/kubernetes/autoscaler/tree/master/addon-resizer) 来扩展 Metrics Server。Addon Resizer 会根据工作节点进行垂直扩展,VPA 会根据 CPU 和内存使用情况进行扩展。

## CoreDNS 延迟退出持续时间

Pods 使用 `kube-dns` 服务进行名称解析。Kubernetes 使用目标 NAT (DNAT) 将 `kube-dns` 流量从节点重定向到 CoreDNS 后端 pods。当您扩展 CoreDNS 部署时, `kube-proxy` 会更新节点上的 iptables 规则和链,以将 DNS 流量重定向到 CoreDNS pods。在您扩展时传播新的端点,以及在缩减 CoreDNS 时删除规则,这可能需要 1 到 10 秒,具体取决于集群的大小。
这种传播延迟可能会导致 CoreDNS pod 被终止时 DNS 查找失败,但节点的 iptables 规则尚未更新。在这种情况下,节点可能会继续向已终止的 CoreDNS Pod 发送 DNS 查询。

您可以通过在 CoreDNS pod 中设置 [lameduck](https://coredns.io/plugins/health/) 持续时间来减少 DNS 查找失败。在 lameduck 模式下,CoreDNS 将继续响应正在进行的请求。设置 lameduck 持续时间将延迟 CoreDNS 关闭过程,为节点更新其 iptables 规则和链条提供所需的时间。

我们建议将 CoreDNS lameduck 持续时间设置为 30 秒。

## CoreDNS 就绪探测

我们建议对 CoreDNS 的就绪探测使用 `/ready` 而不是 `/health`。

为了与前面建议的将 lameduck 持续时间设置为 30 秒的建议保持一致,从而为节点的 iptables 规则更新提供充足的时间,在 pod 终止之前,使用 `/ready` 而不是 `/health` 进行 CoreDNS 就绪探测可确保 CoreDNS pod 在启动时已完全准备就绪,可以及时响应 DNS 请求。
```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8181
    scheme: HTTP
```


关于 CoreDNS Ready 插件的更多信息,请参考 [https://coredns.io/plugins/ready/](https://coredns.io/plugins/ready/)

## 日志和监控代理

日志和监控代理可能会给您的集群控制平面带来显著的负载,因为这些代理需要查询 API 服务器来丰富日志和指标数据,添加工作负载元数据。节点上的代理只能访问本地节点资源,查看容器和进程名称等信息。查询 API 服务器可以添加更多详细信息,如 Kubernetes 部署名称和标签。这对于故障排除非常有帮助,但对于扩展来说可能是有害的。

由于日志和监控有许多不同的选择,我们无法为每个提供商展示示例。对于 [fluentbit](https://docs.fluentbit.io/manual/pipeline/filters/kubernetes)，我们建议启用 Use_Kubelet 从本地 kubelet 获取元数据,而不是从 Kubernetes API 服务器获取,并设置 `Kube_Meta_Cache_TTL` 为一个可以减少重复调用的数字(例如 60)。

监控和日志的扩展有两种一般选择:

* 禁用集成
* 采样和过滤

禁用集成通常不是一个选择,因为您会丢失日志元数据。这消除了 API 扩展问题,但在需要时不会有所需的元数据,会引入其他问题。

采样和过滤减少了收集的指标和日志数量。这将降低对 Kubernetes API 的请求数量,并减少收集的指标和日志的存储量。减少存储成本将降低整个系统的成本。

配置采样的能力取决于代理软件,可以在不同的摄取点实现。重要的是尽可能在代理附近添加采样,因为那里可能发生 API 服务器调用。请联系您的提供商了解更多关于采样支持的信息。

如果您正在使用 CloudWatch 和 CloudWatch Logs,您可以使用 [文档中描述的模式](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/FilterAndPatternSyntax.html) 添加代理过滤。

为了避免丢失日志和指标,您应该将数据发送到一个可以在接收端出现故障时缓冲数据的系统。使用 fluentbit,您可以使用 [Amazon Kinesis Data Firehose](https://docs.fluentbit.io/manual/pipeline/outputs/firehose) 临时保存数据,这可以减少过载最终数据存储位置的机会。
