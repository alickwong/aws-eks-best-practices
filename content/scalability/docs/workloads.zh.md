
# 工作负载

工作负载对集群规模的扩展有影响。大量使用Kubernetes API的工作负载会限制单个集群中可以运行的总工作负载数量,但是您可以更改一些默认设置来帮助减轻负载。

Kubernetes集群中的工作负载可以访问与Kubernetes API集成的功能(例如Secrets和ServiceAccounts),但这些功能并非总是必需的,如果未使用,应将其禁用。限制工作负载对Kubernetes控制平面的访问和依赖性将增加集群中可以运行的工作负载数量,并通过删除对工作负载的不必要访问和实施最小权限实践来提高集群的安全性。请阅读[安全最佳实践](https://aws.github.io/aws-eks-best-practices/security/docs/)以了解更多信息。

## 使用IPv6进行Pod网络

您无法将VPC从IPv4过渡到IPv6,因此在配置集群之前启用IPv6很重要。如果您在VPC中启用了IPv6,这并不意味着您必须使用它,如果您的Pod和服务使用IPv6,您仍然可以将流量路由到和从IPv4地址。请参阅[EKS网络最佳实践](https://aws.github.io/aws-eks-best-practices/networking/index/)以了解更多信息。

在[集群中使用IPv6](https://docs.aws.amazon.com/eks/latest/userguide/cni-ipv6.html)可以避免一些最常见的集群和工作负载扩展限制。IPv6可以避免IP地址耗尽,从而无法创建Pod和节点。它还具有每个节点的性能改进,因为Pod可以更快地获得IP地址,从而减少了每个节点的ENI附加数量。您可以通过在VPC CNI中使用[IPv4前缀模式](https://aws.github.io/aws-eks-best-practices/networking/prefix-mode/)来实现类似的节点性能,但您仍然需要确保VPC中有足够的IP地址可用。

## 限制每个命名空间的服务数量

[每个命名空间最多5,000个服务,每个集群最多10,000个服务](https://github.com/kubernetes/community/blob/master/sig-scalability/configs-and-limits/thresholds.md)。为了帮助组织工作负载和服务,提高性能,并避免命名空间范围内资源的级联影响,我们建议将每个命名空间的服务数量限制在500个以内。

随着集群中服务总数的增加,kube-proxy创建的iptables规则数量也会增加。生成成千上万的iptables规则并通过这些规则路由数据包会对节点性能产生影响,并增加网络延迟。

创建 Kubernetes 命名空间,以涵盖单个应用程序环境,只要每个命名空间中的服务数量不超过 500 个。这将使服务发现保持足够小,以避免服务发现限制,并可帮助您避免服务命名冲突。应用程序环境(例如 dev、test、prod)应使用单独的 EKS 集群,而不是命名空间。

## 了解弹性负载均衡器配额

在创建您的服务时,请考虑您将使用哪种类型的负载均衡(例如网络负载均衡器 (NLB) 或应用程序负载均衡器 (ALB))。每种负载均衡器类型提供不同的功能,并有[不同的配额](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-limits.html)。可以调整某些默认配额,但有一些配额上限是无法更改的。要查看您的帐户配额和使用情况,请在 AWS 控制台中查看[服务配额仪表板](http://console.aws.amazon.com/servicequotas)。

例如,默认 ALB 目标是 1000。如果您有一个服务有超过 1000 个端点,您将需要增加配额或将服务拆分到多个 ALB 或使用 Kubernetes Ingress。默认 NLB 目标是 3000,但每个可用区限制为 500 个目标。如果您的集群为 NLB 服务运行超过 500 个 pod,您将需要使用多个可用区或请求提高配额限制。

使用与服务耦合的负载均衡器的替代方法是使用[入口控制器](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)。AWS Load Balancer 控制器可以为入口资源创建 ALB,但您可能考虑在集群中运行专用控制器。集群内入口控制器允许您通过在集群内运行反向代理来公开多个 Kubernetes 服务。控制器具有不同的功能,例如对 [Gateway API](https://gateway-api.sigs.k8s.io/) 的支持,这可能会根据您的工作负载数量和大小而有所不同。

## 使用 Route 53、Global Accelerator 或 CloudFront

要使用多个负载均衡器提供的服务可用作单个端点,您需要使用 [Amazon CloudFront](https://aws.amazon.com/cloudfront/)、[AWS Global Accelerator](https://aws.amazon.com/global-accelerator/) 或 [Amazon Route 53](https://aws.amazon.com/route53/) 将所有负载均衡器公开为单个面向客户的端点。每个选项都有不同的优势,可以单独或结合使用,具体取决于您的需求。
[简体中文]

Route 53可以在一个通用名称下公开多个负载均衡器,并根据分配的权重将流量发送到每个负载均衡器。您可以在[文档](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-values-weighted.html#rrsets-values-weighted-weight)中了解更多关于DNS权重的信息,并可以在[AWS Load Balancer Controller文档](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/integrations/external_dns/#usage)中了解如何使用[Kubernetes external DNS controller](https://github.com/kubernetes-sigs/external-dns)实现它们。

Global Accelerator可以根据请求IP地址将工作负载路由到最近的区域。这对于部署到多个区域的工作负载可能很有用,但它不会改善单个区域内单个集群的路由。将Route 53与Global Accelerator结合使用还有其他好处,如健康检查和当AZ不可用时的自动故障转移。您可以在[这篇博客文章](https://aws.amazon.com/blogs/containers/operating-a-multi-regional-stateless-application-using-amazon-eks/)中看到使用Global Accelerator和Route 53的示例。

CloudFront可以与Route 53和Global Accelerator一起使用,也可以单独使用,以将流量路由到多个目的地。CloudFront会缓存来自源的资产,这可能会根据您提供的内容减少带宽需求。

## 使用EndpointSlices而不是Endpoints

在发现与服务标签匹配的pod时,您应该使用[EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)而不是Endpoints。Endpoints是一种在小规模下公开服务的简单方法,但自动扩展或更新的大型服务会在Kubernetes控制平面上产生大量流量。EndpointSlices具有自动分组功能,可以启用拓扑感知提示等功能。

并非所有控制器都默认使用EndpointSlices。您应该验证控制器设置并在需要时启用它。对于[AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/deploy/configurations/#controller-command-line-flags),您应该启用`--enable-endpoint-slices`可选标志以使用EndpointSlices。

## 尽可能使用不可变和外部密钥

kubelet会缓存节点上pod使用的密钥的当前密钥和值。kubelet会监视密钥的变化。随着集群规模的扩大,不断增加的监视数量会对API服务器性能产生负面影响。

有两种策略可以减少对密钥的监视数量:

* 对于不需要访问Kubernetes资源的应用程序,您可以通过设置`automountServiceAccountToken: false`来禁用自动挂载服务帐户密钥。
* 如果您的应用程序的密钥是静态的且将来不会被修改,请将[密钥标记为不可变](https://kubernetes.io/docs/concepts/configuration/secret/#secret-immutable)。kubelet不会为不可变的密钥维护API监视。

要禁用自动将服务帐户挂载到pod,您可以在工作负载中使用以下设置。如果特定工作负载需要服务帐户,您可以覆盖这些设置。
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app
automountServiceAccountToken: true
```

在集群中的密钥数量超过10,000的限制之前进行监控。您可以使用以下命令查看集群中密钥的总数。您应该通过集群监控工具来监控此限制。
```
kubectl get secrets -A | wc -l
```


您应该设置监控来提醒集群管理员在达到此限制之前。考虑使用诸如[AWS Key Management Service (AWS KMS)](https://aws.amazon.com/kms/)或[Hashicorp Vault](https://www.vaultproject.io/)等外部机密管理选项,并配合[Secrets Store CSI 驱动程序](https://secrets-store-csi-driver.sigs.k8s.io/)使用。

## 限制部署历史记录

创建、更新或删除 Pod 可能会很慢,因为旧对象仍在集群中被跟踪。您可以减少[部署](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#clean-up-policy)的`revisionHistoryLimit`,以清理较旧的 ReplicaSet,这将降低 Kubernetes 控制器管理器跟踪的总对象数量。部署的默认历史记录限制为 10。

如果您的集群通过 CronJob 或其他机制创建大量作业对象,您应该使用 [`ttlSecondsAfterFinished` 设置](https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/)来自动清理集群中的旧 Pod。这将在指定的时间后从作业历史记录中删除成功执行的作业。

## 默认禁用 enableServiceLinks

当 Pod 在节点上运行时,kubelet 会为每个活动服务添加一组环境变量。Linux 进程有一个最大的环境大小,如果您的命名空间中有太多服务,就可能达到这个限制。每个命名空间的服务数量不应超过 5,000。超过这个数量,服务环境变量的数量就会超过 shell 的限制,导致 Pod 在启动时崩溃。

还有其他原因不应该使用服务环境变量进行服务发现。环境变量名称冲突、泄露服务名称和总环境大小都是问题。您应该使用 CoreDNS 来发现服务端点。

## 限制每个资源的动态准入 Webhook

[动态准入 Webhook](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)包括准入 Webhook 和变更 Webhook。它们是 Kubernetes 控制平面之外的 API 端点,在资源发送到 Kubernetes API 时按顺序调用。每个 Webhook 的默认超时时间为 10 秒,如果您有多个 Webhook 或任何一个 Webhook 超时,都可能增加 API 请求的时间。

确保您的 Webhook 高度可用,特别是在 AZ 故障期间,并正确设置 [failurePolicy](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#failure-policy)以拒绝资源或忽略故障。不要在不需要时调用 Webhook,允许 --dry-run kubectl 命令绕过 Webhook。
```
apiVersion: admission.k8s.io/v1
kind: AdmissionReview
request:
  dryRun: False
```

可变 Webhooks 可以频繁地修改资源。如果您有 5 个可变 Webhooks 并部署 50 个资源，etcd 将存储每个资源的所有版本，直到压缩运行（每 5 分钟一次）以删除已修改资源的旧版本。在这种情况下，当 etcd 删除被取代的资源时，将从 etcd 中删除 200 个资源版本，并且根据资源的大小，可能会在 etcd 主机上占用大量空间，直到每 15 分钟运行一次碎片整理。

这种碎片整理可能会导致 etcd 暂停，这可能会对 Kubernetes API 和控制器产生其他影响。您应该避免频繁修改大型资源或快速连续修改数百个资源。
