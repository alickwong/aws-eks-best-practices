
# EKS 控制平面

Amazon Elastic Kubernetes Service (EKS) 是一项托管的 Kubernetes 服务,可以轻松让您在 AWS 上运行 Kubernetes,无需安装、操作和维护自己的 Kubernetes 控制平面或工作节点。它运行上游 Kubernetes,并经过 Kubernetes 一致性认证。这种一致性确保 EKS 支持 Kubernetes API,就像您可以在 EC2 或本地安装的开源社区版本一样。在上游 Kubernetes 上运行的现有应用程序与 Amazon EKS 兼容。

EKS 自动管理 Kubernetes 控制平面节点的可用性和可伸缩性,并自动替换不健康的控制平面节点。

## EKS 架构

EKS 架构旨在消除可能危及 Kubernetes 控制平面可用性和持久性的任何单点故障。

由 EKS 管理的 Kubernetes 控制平面在 EKS 托管的 VPC 内运行。EKS 控制平面包括 Kubernetes API 服务器节点和 etcd 集群。运行 API 服务器、调度程序和 `kube-controller-manager` 等组件的 Kubernetes API 服务器节点在自动伸缩组中运行。EKS 在不同的可用区(AZ)内至少运行两个 API 服务器节点。同样,为了提高持久性,etcd 服务器节点也在跨三个 AZ 的自动伸缩组中运行。EKS 在每个 AZ 中运行一个 NAT 网关,API 服务器和 etcd 服务器在私有子网中运行。这种架构确保单个 AZ 中的事件不会影响 EKS 集群的可用性。

当您创建新集群时,Amazon EKS 会创建一个高度可用的托管 Kubernetes API 服务器端点,您可以使用它与集群进行通信(使用 `kubectl` 等工具)。托管端点使用 NLB 来负载平衡 Kubernetes API 服务器。EKS 还在不同的 AZ 中配置了两个 [ENI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html),以促进与您的工作节点的通信。

![EKS 数据平面网络连接](./images/eks-data-plane-connectivity.jpeg)

您可以[配置您的 Kubernetes 集群 API 服务器](https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html)是否可从公共互联网(使用公共端点)或通过您的 VPC(使用 EKS 托管的 ENI)访问,或者两者都可以。

无论用户和工作节点是通过公共端点还是 EKS 托管的 ENI 连接到 API 服务器,都有冗余的连接路径。

## 建议

## 监控控制平面指标

监控 Kubernetes API 指标可以让您洞察控制平面的性能并识别问题。不健康的控制平面可能会影响集群内运行的工作负载的可用性。例如,编写不当的控制器可能会使 API 服务器过载,从而影响您的应用程序的可用性。

Kubernetes 在 `/metrics` 端点公开控制平面指标。
您可以使用 `kubectl` 查看公开的指标:
```shell
kubectl get --raw /metrics
```


这些指标以 [Prometheus 文本格式](https://github.com/prometheus/docs/blob/master/content/docs/instrumenting/exposition_formats.md)表示。

您可以使用 Prometheus 收集和存储这些指标。2020 年 5 月,CloudWatch 增加了对在 CloudWatch Container Insights 中监控 Prometheus 指标的支持。因此,您也可以使用 Amazon CloudWatch 监控 EKS 控制平面。您可以使用[添加新的 Prometheus 抓取目标教程:Prometheus KPI 服务器指标](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights-Prometheus-Setup-configure.html#ContainerInsights-Prometheus-Setup-new-exporters)来收集指标并创建 CloudWatch 仪表板来监控您集群的控制平面。

您可以在[此处](https://github.com/kubernetes/apiserver/blob/master/pkg/endpoints/metrics/metrics.go)找到 Kubernetes API 服务器指标。例如,`apiserver_request_duration_seconds`可以指示 API 请求运行的时间长度。

考虑监控这些控制平面指标:

### API 服务器

| 指标 | 描述 |
|:--|:--|
| `apiserver_request_total` | 按动词、干运行值、组、版本、资源、范围、组件和 HTTP 响应代码细分的 apiserver 请求计数器。 |
| `apiserver_request_duration_seconds*` | 按动词、干运行值、组、版本、资源、子资源、范围和组件细分的响应延迟分布(以秒为单位)。 |
| `apiserver_admission_controller_admission_duration_seconds` | 按名称、操作和 API 资源和类型(验证或准入)细分的准入控制器延迟直方图(以秒为单位)。 |
| `apiserver_admission_webhook_rejection_count` | 准入 webhook 拒绝计数。按名称、操作、拒绝代码、类型(验证或准入)和错误类型(调用 webhook 错误、apiserver 内部错误、无错误)标识。 |
| `rest_client_request_duration_seconds` | 按动词和 URL 细分的请求延迟(以秒为单位)。 |
| `rest_client_requests_total` | 按状态代码、方法和主机细分的 HTTP 请求数量。 |

### etcd

| 指标 | 描述 |
|:--|:--|
| `etcd_request_duration_seconds` | 按操作和对象类型细分的 etcd 请求延迟(以秒为单位)。 |
| `etcd_db_total_size_in_bytes` 或 <br />`apiserver_storage_db_total_size_in_bytes`(从 EKS v1.26 开始) 或 <br />`apiserver_storage_size_bytes`(从 EKS v1.28 开始) | etcd 数据库大小。 |
考虑使用 [Kubernetes 监控概览仪表板](https://grafana.com/grafana/dashboards/14623)来可视化和监控 Kubernetes API 服务器请求和延迟以及 etcd 延迟指标。

以下 Prometheus 查询可用于监控 etcd 的当前大小。该查询假设有一个名为 `kube-apiserver` 的作业来从 API 指标端点抓取指标,并且 EKS 版本低于 v1.26。
```text
max(etcd_db_total_size_in_bytes{job="kube-apiserver"} / (8 * 1024 * 1024 * 1024))
```

当数据库大小限制被超过时，etcd 会发出没有空间的警报并停止接受进一步的写入请求。换句话说，集群变为只读模式，所有修改对象(如创建新 pod、扩缩部署等)的请求都会被集群的 API 服务器拒绝。

## 集群身份验证

EKS 目前支持两种身份验证方式：[持有者/服务帐户令牌](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#service-account-tokens)和使用 [Webhook 令牌身份验证](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication)的 IAM 身份验证。当用户调用 Kubernetes API 时，Webhook 会将请求中包含的身份验证令牌传递给 IAM。该令牌是一个 base64 编码的签名 URL,由 AWS 命令行界面([AWS CLI](https://aws.amazon.com/cli/))生成。

创建 EKS 集群的 IAM 用户或角色会自动获得对集群的完全访问权限。您可以通过编辑 [`aws-auth` configmap](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html) 来管理对 EKS 集群的访问。

如果您错误配置了 `aws-auth` configmap 并失去了对集群的访问权限,您仍然可以使用创建集群的用户或角色来访问您的 EKS 集群。

在极少数情况下,如果您无法在 AWS 区域中使用 IAM 服务,您也可以使用 Kubernetes 服务帐户的持有者令牌来管理集群。

创建一个"超级管理员"帐户,该帐户被允许在集群中执行所有操作:
```
kubectl -n kube-system create serviceaccount super-admin
```

创建一个角色绑定,赋予超级管理员集群管理员角色:
```
kubectl create clusterrolebinding super-admin-rb --clusterrole=cluster-admin --serviceaccount=kube-system:super-admin
```

获取服务帐户的密钥:
```
SECRET_NAME=`kubectl -n kube-system get serviceaccount/super-admin -o jsonpath='{.secrets[0].name}'`
```

获取与密钥关联的令牌:
```
TOKEN=`kubectl -n kube-system get secret $SECRET_NAME -o jsonpath='{.data.token}'| base64 --decode`
```

将服务帐户和令牌添加到 `kubeconfig`:
```
kubectl config set-credentials super-admin --token=$TOKEN
```

在 `kubeconfig` 中设置当前上下文以使用超级管理员帐户:
```
kubectl config set-context --current --user=super-admin
```

最终的 `kubeconfig` 应该如下所示:
```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data:<REDACTED>
    server: https://<CLUSTER>.gr7.us-west-2.eks.amazonaws.com
  name: arn:aws:eks:us-west-2:<account number>:cluster/<cluster name>
contexts:
- context:
    cluster: arn:aws:eks:us-west-2:<account number>:cluster/<cluster name>
    user: super-admin
  name: arn:aws:eks:us-west-2:<account number>:cluster/<cluster name>
current-context: arn:aws:eks:us-west-2:<account number>:cluster/<cluster name>
kind: Config
preferences: {}
users:
#- name: arn:aws:eks:us-west-2:<account number>:cluster/<cluster name>
#  user:
#    exec:
#      apiVersion: client.authentication.k8s.io/v1alpha1
#      args:
#      - --region
#      - us-west-2
#      - eks
#      - get-token
#      - --cluster-name
#      - <<cluster name>>
#      command: aws
#      env: null
- name: super-admin
  user:
    token: <<super-admin sa’s secret>>
```

## 准入 Webhooks

Kubernetes 有两种类型的准入 Webhooks：[验证准入 Webhooks 和变更准入 Webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers)。这些允许用户扩展 Kubernetes API，并在对象被 API 接受之前对其进行验证或变更。这些 Webhooks 的配置不当可能会destabilize EKS 控制平面，阻塞集群关键操作。

为了避免影响集群关键操作，请避免设置类似以下的"catch-all"Webhooks：
```
- name: "pod-policy.example.com"
  rules:
  - apiGroups:   ["*"]
    apiVersions: ["*"]
    operations:  ["*"]
    resources:   ["*"]
    scope: "*"
```


在集群关键工作负载受到影响的情况下,请确保 webhook 具有故障开放策略,并且超时时间短于 30 秒。

### 阻止使用不安全 `sysctls` 的 Pod

`Sysctl` 是一个 Linux 实用程序,允许用户在运行时修改内核参数。这些内核参数控制操作系统行为的各个方面,如网络、文件系统、虚拟内存和进程管理。

Kubernetes 允许为 Pod 分配 `sysctl` 配置文件。Kubernetes 将 `sysctls` 分为安全和不安全。安全的 `sysctls` 在容器或 Pod 中是命名空间化的,设置它们不会影响节点上的其他 Pod 或节点本身。相比之下,不安全的 sysctls 默认被禁用,因为它们可能会扰乱其他 Pod 或使节点不稳定。

由于默认禁用不安全的 `sysctls`,kubelet 将不会创建具有不安全 `sysctl` 配置文件的 Pod。如果您创建了这样的 Pod,调度器将反复将这些 Pod 分配到节点,而节点无法启动它们。这种无限循环最终会使集群控制平面承受压力,导致集群不稳定。

考虑使用 [OPA Gatekeeper](https://github.com/open-policy-agent/gatekeeper-library/blob/377cb915dba2db10702c25ef1ee374b4aa8d347a/src/pod-security-policy/forbidden-sysctls/constraint.tmpl) 或 [Kyverno](https://kyverno.io/policies/pod-security/baseline/restrict-sysctls/restrict-sysctls/) 来拒绝具有不安全 `sysctls` 的 Pod。

## 处理集群升级

自 2021 年 4 月以来,Kubernetes 的发布周期已从每年四次(每季度一次)改为每年三次。新的次版本(如 1.**21** 或 1.**22**)大约每 15 周发布一次。从 Kubernetes 1.19 开始,每个次版本在首次发布后大约支持 12 个月。随着 Kubernetes v1.28 的出现,控制平面和工作节点之间的兼容性偏差已从 n-2 扩展到 n-3 个次版本。要了解更多信息,请参见[集群升级最佳实践](../../upgrades/index.md)。

## 运行大型集群

EKS 会主动监控控制平面实例的负载,并自动扩展它们以确保高性能。但是,在运行大型集群时,您应该考虑 Kubernetes 内部可能出现的性能问题和限制,以及 AWS 服务中的配额。

- 拥有超过 1000 个服务的集群可能会在使用 `iptables` 模式的 `kube-proxy` 时遇到网络延迟,这是根据 ProjectCalico 团队的[测试结果](https://www.projectcalico.org/comparing-kube-proxy-modes-iptables-or-ipvs/)得出的。解决方案是[切换到在 `ipvs` 模式下运行 `kube-proxy`](https://medium.com/@jeremy.i.cowan/the-problem-with-kube-proxy-enabling-ipvs-on-eks-169ac22e237e)。

- 您也可能遇到[EC2 API 请求节流](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/throttling.html)的情况,如果 CNI 需要为 Pod 请求 IP 地址,或者您需要频繁创建新的 EC2 实例。您可以通过配置 CNI 缓存 IP 地址来减少对 EC2 API 的调用。您可以使用更大的 EC2 实例类型来减少 EC2 扩展事件。

## 其他资源:

- [揭开 Amazon EKS 工作节点集群网络的神秘面纱](https://aws.amazon.com/blogs/containers/de-mystifying-cluster-networking-for-amazon-eks-worker-nodes/)
- [Amazon EKS 集群端点访问控制](https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html)
- [AWS re:Invent 2019: Amazon EKS 内部原理 (CON421-R1)](https://www.youtube.com/watch?v=7vxDWDD2YnM)
