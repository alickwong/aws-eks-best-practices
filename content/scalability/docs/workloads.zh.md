# 工作负载

工作负载对集群规模的扩展有影响。大量使用 Kubernetes API 的工作负载会限制单个集群中可以运行的总工作负载数量,但您可以更改一些默认设置来减轻负载。

Kubernetes 集群中的工作负载可以访问与 Kubernetes API 集成的功能(如 Secrets 和 ServiceAccounts),但这些功能并非总是必需的,如果未使用,应将其禁用。限制工作负载对 Kubernetes 控制平面的访问和依赖性,将增加集群中可以运行的工作负载数量,并通过删除对工作负载的不必要访问和实施最小权限实践来提高集群的安全性。请阅读[安全最佳实践](https://aws.github.io/aws-eks-best-practices/security/docs/)以了解更多信息。

## 使用 IPv6 进行 pod 网络

您无法将 VPC 从 IPv4 过渡到 IPv6,因此在配置集群之前启用 IPv6 很重要。如果您在 VPC 中启用了 IPv6,这并不意味着您必须使用它,如果您的 pod 和服务使用 IPv6,您仍然可以将流量路由到和从 IPv4 地址。请参阅[EKS 网络最佳实践](https://aws.github.io/aws-eks-best-practices/networking/index/)以了解更多信息。

在集群中使用 [IPv6](https://docs.aws.amazon.com/eks/latest/userguide/cni-ipv6.html) 可以避免一些最常见的集群和工作负载扩展限制。IPv6 避免了 IP 地址耗尽,即 pod 和节点无法创建,因为没有可用的 IP 地址。它还具有每个节点的性能改进,因为 pod 通过减少每个节点的 ENI 附件数量而更快地获得 IP 地址。您可以通过在 VPC CNI 中使用 [IPv4 前缀模式](https://aws.github.io/aws-eks-best-practices/networking/prefix-mode/)来实现类似的节点性能,但您仍然需要确保 VPC 中有足够的 IP 地址可用。

## 限制每个命名空间的服务数量

[每个命名空间最多 5,000 个服务,集群中最多 10,000 个服务](https://github.com/kubernetes/community/blob/master/sig-scalability/configs-and-limits/thresholds.md)。为了帮助组织工作负载和服务,提高性能,并避免命名空间范围内资源的级联影响,我们建议将每个命名空间的服务数量限制在 500 以内。

随着集群中服务总数的增加,kube-proxy 每个节点创建的 IP 表规则数量也在增加。生成成千上万的 IP 表规则并通过这些规则路由数据包会对节点产生性能影响,并增加网络延迟。

创建包含单个应用程序环境的 Kubernetes 命名空间,只要每个命名空间的服务数量不超过 500。这将使服务发现保持足够小,以避免服务发现限制,还可以帮助您避免服务命名冲突。应用程序环境(如 dev、test、prod)应使用单独的 EKS 集群,而不是命名空间。

## 了解弹性负载均衡器配额

在创建服务时,请考虑您将使用哪种类型的负载均衡(例如网络负载均衡器 (NLB) 或应用程序负载均衡器 (ALB))。每种负载均衡器类型提供不同的功能,并有[不同的配额](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-limits.html)。一些默认配额可以调整,但有一些配额上限是无法更改的。要查看您的帐户配额和使用情况,请在 AWS 控制台中查看[服务配额仪表板](http://console.aws.amazon.com/servicequotas)。

例如,默认 ALB 目标是 1000。如果您有一个服务有超过 1000 个端点,您将需要增加配额或将服务拆分到多个 ALB 或使用 Kubernetes Ingress。默认 NLB 目标是 3000,但每个 AZ 限制为 500 个目标。如果您的集群为 NLB 服务运行超过 500 个 pod,您将需要使用多个 AZ 或请求提高配额限制。

使用与服务耦合的负载均衡器的替代方法是使用[入口控制器](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)。AWS Load Balancer 控制器可以为入口资源创建 ALB,但您可能考虑在集群中运行专用的控制器。集群内入口控制器允许您通过在集群内运行反向代理来公开多个 Kubernetes 服务。控制器具有不同的功能,例如对 [Gateway API](https://gateway-api.sigs.k8s.io/) 的支持,这可能会根据您的工作负载数量和大小而有所不同。

## 使用 Route 53、Global Accelerator 或 CloudFront

要使用多个负载均衡器公开服务,您需要使用 [Amazon CloudFront](https://aws.amazon.com/cloudfront/)、[AWS Global Accelerator](https://aws.amazon.com/global-accelerator/) 或 [Amazon Route 53](https://aws.amazon.com/route53/) 将所有负载均衡器公开为单个面向客户的端点。每个选项都有不同的优势,可以单独或组合使用,具体取决于您的需求。

Route 53 可以在一个通用名称下公开多个负载均衡器,并根据分配的权重将流量发送到每个负载均衡器。您可以在[文档](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-values-weighted.html#rrsets-values-weighted-weight)中了解更多关于 DNS 权重的信息,并在 [AWS Load Balancer Controller 文档](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/integrations/external_dns/#usage)中了解如何使用 [Kubernetes external DNS 控制器](https://github.com/kubernetes-sigs/external-dns)实现它们。

Global Accelerator 可以根据请求 IP 地址将工作负载路由到最近的区域。这对于部署到多个区域的工作负载可能很有用,但它不会改善对单个区域集群的路由。将 Route 53 与 Global Accelerator 结合使用还有其他好处,如健康检查和 AZ 不可用时的自动故障转移。您可以在[这篇博文](https://aws.amazon.com/blogs/containers/operating-a-multi-regional-stateless-application-using-amazon-eks/)中看到使用 Global Accelerator 和 Route 53 的示例。

CloudFront 可以与 Route 53 和 Global Accelerator 一起使用,也可以单独使用,将流量路由到多个目的地。CloudFront 会缓存来自源的资产,这可能会根据您要提供的内容减少带宽需求。

## 使用 EndpointSlices 而不是 Endpoints

在发现与服务标签匹配的 pod 时,您应该使用 [EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/) 而不是 Endpoints。Endpoints 是一种在小规模下公开服务的简单方法,但自动扩展或更新的大型服务会在 Kubernetes 控制平面上产生大量流量。EndpointSlices 具有自动分组功能,可以启用拓扑感知提示等功能。

并非所有控制器都默认使用 EndpointSlices。您应该验证您的控制器设置并在需要时启用它。对于 [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/deploy/configurations/#controller-command-line-flags),您应该启用可选标志 `--enable-endpoint-slices` 以使用 EndpointSlices。

## 尽可能使用不可变和外部 Secrets

kubelet 会缓存 pod 在该节点上使用的 Secrets 的当前密钥和值。kubelet 会监视 Secrets 以检测更改。随着集群的扩展,不断增加的监视数量会对 API 服务器性能产生负面影响。

有两种策略可以减少对 Secrets 的监视:

* 对于不需要访问 Kubernetes 资源的应用程序,您可以通过设置 `automountServiceAccountToken: false` 来禁用自动挂载服务帐户 Secrets
* 如果您的应用程序 Secrets 是静态的并且将来不会修改,请将[密钥标记为不可变](https://kubernetes.io/docs/concepts/configuration/secret/#secret-immutable)。kubelet 不会为不可变的 Secrets 维护 API 监视。

要禁用自动将服务帐户挂载到 pod,您可以在工作负载中使用以下设置。如果特定工作负载需要服务帐户,您可以覆盖这些设置。

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app
automountServiceAccountToken: true
```

监控集群中 Secrets 的数量,确保不超过 10,000 个的限制。您可以使用以下命令查看集群中 Secrets 的总数。您应该通过集群监控工具监控此限制。

```
kubectl get secrets -A | wc -l
```

您应该设置监控,在达到此限制之前提醒集群管理员。考虑使用外部 Secrets 管理选项,如 [AWS Key Management Service (AWS KMS)](https://aws.amazon.com/kms/) 或 [Hashicorp Vault](https://www.vaultproject.io/) 与 [Secrets Store CSI 驱动程序](https://secrets-store-csi-driver.sigs.k8s.io/)。

## 限制部署历史记录

创建、更新或删除 pod 可能会很慢,因为旧对象仍在集群中被跟踪。您可以减少[部署](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#clean-up-policy)的 `revisionHistoryLimit`,以清理旧的 ReplicaSets,这将降低 Kubernetes 控制器管理器跟踪的总对象数量。部署的默认历史记录限制为 10。

如果您的集群通过 CronJobs 或其他机制创建大量 job 对象,您应该使用 [`ttlSecondsAfterFinished` 设置](https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/)来自动清理集群中的旧 pod。这将在指定的时间后从 job 历史记录中删除成功执行的 job。

## 默认禁用 enableServiceLinks

当 Pod 在节点上运行时,kubelet 会为每个活动服务添加一组环境变量。Linux 进程有一个最大的环境大小,如果您的命名空间中有太多服务,这个大小可能会被达到。每个命名空间的服务数量不应超过 5,000。超过这个数量后,服务环境变量的数量会超过 shell 限制,导致 pod 在启动时崩溃。

还有其他原因 pod 不应该使用服务环境变量进行服务发现。环境变量名称冲突、泄露服务名称和总环境大小都是一些原因。您应该使用 CoreDNS 来发现服务端点。

## 限制每个资源的动态准入 Webhooks

[动态准入 Webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)包括准入 Webhooks 和变更 Webhooks。它们是不属于 Kubernetes 控制平面的 API 端点,在资源发送到 Kubernetes API 时按顺序调用。每个 Webhook 的默认超时时间为 10 秒,如果您有多个 Webhook 或任何一个 Webhook 超时,都可能增加 API 请求的时间。

确保您的 Webhooks 高度可用,特别是在 AZ 故障期间,并正确设置 [failurePolicy](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#failure-policy) 以拒绝资源或忽略失败。不要在不需要时调用 Webhooks,允许 --dry-run kubectl 命令绕过 Webhook。

```
apiVersion: admission.k8s.io/v1
kind: AdmissionReview
request:
  dryRun: False
```

变更 Webhooks 可以频繁地修改资源。如果您有 5 个变更 Webhooks 并部署 50 个资源,etcd 将存储每个资源的所有版本,直到压缩运行(每 5 分钟一次)以删除修改资源的旧版本。在这种情况下,当 etcd 删除过时的资源时,将有 200 个资源版本被删除,并且根据资源的大小可能会在 etcd 主机上占用相当大的空间,直到 15 分钟后进行碎片整理。

这种碎片整理可能会导致 etcd 暂停,从而可能会对 Kubernetes API 和控制器产生其他影响。您应该避免频繁修改大型资源或快速连续修改数百个资源。