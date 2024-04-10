# 随时间优化(合理调整)

根据 AWS 完善架构框架,合理调整是"...使用成本最低的资源,同时仍能满足特定工作负载的技术规格"。

当您为 Pod 中的容器指定资源 `requests` 时,调度程序会使用此信息来决定将 Pod 放置在哪个节点上。当您为容器指定资源 `limits` 时,kubelet 会强制执行这些限制,以确保正在运行的容器不会使用超过您设置的限制的资源。有关 Kubernetes 如何管理容器资源的详细信息,请参见[文档](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)。

在 Kubernetes 中,这意味着设置正确的计算资源([CPU 和内存统称为计算资源](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)) - 设置尽可能接近实际利用率的资源 `requests`。有关获取 Pod 实际资源使用情况的工具,请参见下面的建议部分。

**AWS Fargate 上的 Amazon EKS**: 当 pod 在 Fargate 上调度时,pod 规范中的 vCPU 和内存预留决定了为 pod 配置多少 CPU 和内存。如果您没有指定 vCPU 和内存组合,则使用最小可用组合 (.25 vCPU 和 0.5 GB 内存)。有关在 Fargate 上运行的 pod 可用的 vCPU 和内存组合列表,请参见 [Amazon EKS 用户指南](https://docs.aws.amazon.com/eks/latest/userguide/fargate-pod-configuration.html)。

**EC2 上的 Amazon EKS**: 创建 Pod 时,您可以指定容器需要的每种资源(如 CPU 和内存)的数量。我们不应过度配置(这将导致浪费)或配置不足(这将导致节流)分配给容器的资源非常重要。

## 建议
### 使用工具根据观察数据分配资源
有工具如 [kube resource report](https://github.com/hjacobs/kube-resource-report) 可以帮助调整部署在 Amazon EKS 上 EC2 节点上的 pod 的大小。

kube resource report 的部署步骤:
```
$ git clone https://github.com/hjacobs/kube-resource-report
$ cd kube-resource-report
$ helm install kube-resource-report ./unsupported/chart/kube-resource-report
$ helm status kube-resource-report
$ export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=kube-resource-report,app.kubernetes.io/instance=kube-resource-report" -o jsonpath="{.items[0].metadata.name}")
$ echo "Visit http://127.0.0.1:8080 to use your application"
$ kubectl port-forward $POD_NAME 8080:8080
```
此工具的示例报告截图:

![Home Page](../images/kube-resource-report1.png)

![Cluster level data](../images/kube-resource-report2.png)

![Pod level data](../images/kube-resource-report3.png)

**FairwindsOps Goldilocks**: [FairwindsOps Goldilocks](https://github.com/FairwindsOps/goldilocks) 是一个工具,它为每个部署创建一个垂直 Pod 自动缩放器 (VPA),然后查询它们以获取信息。一旦 VPA 就位,我们就会在 Goldilocks 仪表板上看到建议。

根据[文档](https://docs.aws.amazon.com/eks/latest/userguide/vertical-pod-autoscaler.html)部署垂直 Pod 自动缩放器。

启用命名空间 - 选择一个应用程序命名空间并将其标记,如下所示,以便看到一些数据,在下面的示例中,我们指定了 default 命名空间:

```
$ kubectl label ns default goldilocks.fairwinds.com/enabled=true
```

查看仪表板 - 默认安装会为仪表板创建一个 ClusterIP 服务。您可以通过端口转发访问:

```
$ kubectl -n goldilocks port-forward svc/goldilocks-dashboard 8080:80
```

然后在浏览器中打开 http://localhost:8080

![Goldilocks recommendation Page](../images/Goldilocks.png)

### 使用 CloudWatch Container Insights 和 Amazon CloudWatch 中的 Prometheus 指标等应用程序分析工具

使用 [CloudWatch Container Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/deploy-container-insights-EKS.html) 了解如何使用原生 CloudWatch 功能来监控您的 EKS 集群性能。您可以使用 CloudWatch Container Insights 收集、聚合和汇总在 Amazon Elastic Kubernetes Service 上运行的容器化应用程序和微服务的指标和日志。这些指标包括 CPU、内存、磁盘和网络等资源的利用率 - 这可以帮助合理调整 Pod 并节省成本。

[Container Insights Prometheus Metrics Monitoring](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights-Prometheus-metrics.html) 目前,对 Prometheus 指标的支持仍处于测试阶段。CloudWatch Container Insights 对 Prometheus 的监控自动发现容器化系统和工作负载的 Prometheus 指标。Prometheus 是一个开源的系统监控和警报工具包。所有 Prometheus 指标都收集在 ContainerInsights/Prometheus 命名空间中。

cAdvisor 和 kube-state-metrics 提供的指标可用于使用 Prometheus 和 Grafana 监控 AWS Fargate 上的 Amazon EKS 上的 pod,这可用于在容器中实现 **requests**。有关更多详细信息,请参阅[此博客文章](https://aws.amazon.com/blogs/containers/monitoring-amazon-eks-on-aws-fargate-using-prometheus-and-grafana/)。

**Right Size Guide**: [right size guide (rsg)](https://mhausenblas.info/right-size-guide/) 是一个简单的 CLI 工具,可为您的应用程序提供内存和 CPU 建议。该工具适用于多个容器编排器,包括 Kubernetes,并且易于部署。

通过使用 CloudWatch Container Insights、Kube Resource Report、Goldilocks 和其他工具,在 Kubernetes 集群中运行的应用程序可以得到合理调整,从而降低成本。

## 资源
请参考以下资源了解有关成本优化最佳实践的更多信息。

### 文档和博客
+	[Amazon EKS Workshop - 设置 EKS CloudWatch Container Insights](https://www.eksworkshop.com/intermediate/250_cloudwatch_container_insights/)
+	[在 Amazon CloudWatch 中使用 Prometheus 指标](https://aws.amazon.com/blogs/containers/using-prometheus-metrics-in-amazon-cloudwatch/)
+ [使用 Prometheus 和 Grafana 监控 AWS Fargate 上的 Amazon EKS](https://aws.amazon.com/blogs/containers/monitoring-amazon-eks-on-aws-fargate-using-prometheus-and-grafana/)

### 工具
+  [Kube resource report](https://github.com/hjacobs/kube-resource-report)
+  [Right size guide](https://github.com/mhausenblas/right-size-guide)
+ [Fargate count](https://github.com/mreferre/fargatecount)
+ [FairwindsOps Goldilocks](https://github.com/FairwindsOps/goldilocks)
+ [选择正确的节点大小](https://learnk8s.io/research#choosing-node-size)