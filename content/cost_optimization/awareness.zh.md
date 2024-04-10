# 支出意识

支出意识是了解在您的 EKS 集群中谁、在哪里以及什么导致了支出。获得这些数据的准确图景将有助于提高您的支出意识并突出需要补救的领域。


## 建议
### 使用 Cost Explorer

[AWS Cost Explorer](https://aws.amazon.com/aws-cost-management/aws-cost-explorer/) 拥有易于使用的界面,可让您可视化、了解和管理您的 AWS 成本和使用情况随时间的变化。您可以使用 Cost Explorer 中可用的过滤器分析成本和使用数据,并查看不同级别的信息。

#### EKS 控制平面和 EKS Fargate 成本

使用过滤器,我们可以查询 EKS 控制平面和 Fargate Pod 产生的成本,如下图所示:

![Cost Explorer - EKS Control Plane](../images/eks-controlplane-costexplorer.png)

使用过滤器,我们可以查询 EKS 中 Fargate Pod 在各个区域产生的总成本,包括 vCPU-Hours 和 GB Hrs,如下图所示:

![Cost Explorer - EKS Fargate](../images/eks-fargate-costexplorer.png)

#### 资源标记

Amazon EKS 支持[添加 AWS 标签](https://docs.aws.amazon.com/eks/latest/userguide/eks-using-tags.html)到您的 Amazon EKS 集群。这使得轻松控制对 EKS API 的访问以管理您的集群。添加到 EKS 集群的标签特定于 AWS EKS 集群资源,它们不会传播到集群使用的其他 AWS 资源,如 EC2 实例或负载均衡器。目前,通过 AWS API、控制台和 SDK,支持为所有新的和现有的 EKS 集群进行集群标记。

AWS Fargate 是一项提供按需、大小合适的容器计算能力的技术。在您的集群中调度 pod 到 Fargate 之前,您必须定义至少一个 Fargate 配置文件,该配置文件指定哪些 pod 在启动时应使用 Fargate。

添加和列出 EKS 集群的标签:
```
$ aws eks tag-resource --resource-arn arn:aws:eks:us-west-2:xxx:cluster/ekscluster1 --tags team=devops,env=staging,bu=cio,costcenter=1234
$ aws eks list-tags-for-resource --resource-arn arn:aws:eks:us-west-2:xxx:cluster/ekscluster1
{
    "tags": {
        "bu": "cio",
        "env": "staging",
        "costcenter": "1234",
        "team": "devops"
    }
}
```
在[AWS Cost Explorer](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-alloc-tags.html)中激活成本分配标签后,AWS 将使用成本分配标签在您的成本分配报告中组织您的资源成本,以帮助您更轻松地对您的 AWS 成本进行分类和跟踪。

标签对 Amazon EKS 没有任何语义含义,只是被解释为一串字符。例如,您可以为 Amazon EKS 集群定义一组标签,以帮助您跟踪每个集群的所有者和堆栈级别。

### 使用 AWS Trusted Advisor

AWS Trusted Advisor 提供了一套丰富的最佳实践检查和建议,涵盖成本优化、安全性、容错能力、性能和服务限制等五个类别。

对于成本优化,Trusted Advisor 可帮助消除未使用和空闲的资源,并建议做出预留容量的承诺。将有助于 Amazon EKS 的关键行动项目包括低利用率的 EC2 实例、未关联的弹性 IP 地址、空闲的负载均衡器、未充分利用的 EBS 卷等。完整的检查列表可在 https://aws.amazon.com/premiumsupport/technology/trusted-advisor/best-practice-checklist/ 找到。

Trusted Advisor 还为 EC2 实例和 Fargate 提供了节省计划和预留实例的建议,允许您承诺一致的使用量以换取折扣费率。

!!! 注意
    Trusted Advisor 的建议是通用建议,而不是特定于 EKS 的。

### 使用 Kubernetes 仪表板

***Kubernetes 仪表板***

Kubernetes 仪表板是一个通用的基于 Web 的 Kubernetes 集群 UI,提供有关 Kubernetes 集群的信息,包括集群、节点和 pod 级别的资源使用情况。在 Amazon EKS 集群上部署 Kubernetes 仪表板的说明在 [Amazon EKS 文档](https://docs.aws.amazon.com/eks/latest/userguide/dashboard-tutorial.html)中有描述。

仪表板提供了每个节点和 pod 的资源使用情况细分,以及 pod、服务、部署和其他 Kubernetes 对象的详细元数据。这些综合信息提供了对 Kubernetes 环境的可见性。

![Kubernetes 仪表板](../images/kubernetes-dashboard.png)

***kubectl top 和 describe 命令***

使用 kubectl top 和 kubectl describe 命令查看资源使用指标。kubectl top 将显示您集群中 pod 或节点的当前 CPU 和内存使用情况,或特定 pod 或节点的使用情况。kubectl describe 命令将提供有关特定节点或 pod 的更详细信息。
```
$ kubectl top pods
$ kubectl top nodes
$ kubectl top pod pod-name --namespace mynamespace --containers
```

使用 top 命令,输出将显示节点使用的 CPU (以核心为单位) 和内存 (以 MiB 为单位) 的总量,以及这些数字占节点可分配容量的百分比。然后,您可以通过添加 *--containers* 标志深入到 pod 内的容器级别。

```
$ kubectl describe node <node>
$ kubectl describe pod <pod>
```

*kubectl describe* 返回每个资源请求或限制所占总可用容量的百分比。

kubectl top 和 describe 跟踪 Kubernetes pod、节点和容器的关键资源(如 CPU、内存和存储)的利用率和可用性。这种意识将有助于了解资源使用情况并有助于控制成本。


### 使用 CloudWatch 容器洞察

使用 [CloudWatch 容器洞察](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/deploy-container-insights-EKS.html)收集、聚合和汇总您的容器化应用程序和微服务的指标和日志。容器洞察适用于 Amazon Elastic Kubernetes Service on EC2 和 Amazon EC2 上的 Kubernetes 平台。指标包括 CPU、内存、磁盘和网络的利用率。

洞察的安装在[文档](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/deploy-container-insights-EKS.html)中给出。

CloudWatch 在集群、节点、pod、任务和服务级别创建聚合指标作为 CloudWatch 指标。

**以下查询显示了按平均节点 CPU 利用率排序的节点列表**
```
STATS avg(node_cpu_utilization) as avg_node_cpu_utilization by NodeName
| SORT avg_node_cpu_utilization DESC 
```

**按容器名称的 CPU 使用情况**
```
stats pct(container_cpu_usage_total, 50) as CPUPercMedian by kubernetes.container_name 
| filter Type="Container"
```
**按容器名称的磁盘使用情况**
```
stats floor(avg(container_filesystem_usage/1024)) as container_filesystem_usage_avg_kb by InstanceId, kubernetes.container_name, device 
| filter Type="ContainerFS" 
| sort container_filesystem_usage_avg_kb desc
```

更多示例查询可在[容器洞察文档](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-view-metrics.html)中找到

这种意识将有助于了解资源使用情况并有助于控制成本。



### 使用 KubeCost 进行支出意识和指导

第三方工具如 [kubecost](https://kubecost.com/) 也可以部署在 Amazon EKS 上,以获得运行 Kubernetes 集群的可见性。请参考此 [AWS 博客](https://aws.amazon.com/blogs/containers/how-to-track-costs-in-multi-tenant-amazon-eks-clusters-using-kubecost/)了解使用 Kubecost 跟踪成本的信息。

使用 Helm 3 部署 kubecost:
```
$ curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
$ helm version --short
v3.2.1+gfe51cd1
$ helm repo add stable https://kubernetes-charts.storage.googleapis.com/
$ helm repo add stable https://kubernetes-charts.storage.googleapis.com/c^C
$ kubectl create namespace kubecost 
namespace/kubecost created
$ helm repo add kubecost https://kubecost.github.io/cost-analyzer/ 
"kubecost" has been added to your repositories

$ helm install kubecost kubecost/cost-analyzer --namespace kubecost --set kubecostToken="aGRoZEBqc2pzLmNvbQ==xm343yadf98"
NAME: kubecost
LAST DEPLOYED: Mon May 18 08:49:05 2020
NAMESPACE: kubecost
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
--------------------------------------------------Kubecost has been successfully installed. When pods are Ready, you can enable port-forwarding with the following command:
    
    kubectl port-forward --namespace kubecost deployment/kubecost-cost-analyzer 9090
    
Next, navigate to http://localhost:9090 in a web browser.
$ kubectl port-forward --namespace kubecost deployment/kubecost-cost-analyzer 9090

Note: If you are using Cloud 9 or have a need to forward it to a different port like 8080, issue the following command
$ kubectl port-forward --namespace kubecost deployment/kubecost-cost-analyzer 8080:9090

```
Kube Cost 仪表板 -
![Kubernetes 集群自动缩放器日志](../images/kube-cost.png)

### 使用 Kubernetes 成本分配和容量规划分析工具

[Kubernetes Opex Analytics](https://github.com/rchakode/kube-opex-analytics) 是一个工具,可帮助组织跟踪其 Kubernetes 集群消耗的资源,以防止过度支付。为此,它生成短期(7 天)、中期(14 天)和长期(12 个月)的使用报告,显示每个项目随时间推移所消耗的资源金额的相关见解。

![Kubernetes Opex Analytics](../images/kube-opex-analytics.png)


### Magalix Kubeadvisor

[KubeAdvisor](https://www.magalix.com/kubeadvisor) 持续扫描您的 Kubernetes 集群,并报告您如何修复问题、应用最佳实践和优化您的集群(包括围绕成本效率的 CPU/内存等资源的建议)。

### Spot.io,以前称为 Spotinst

Spotinst Ocean 是一项应用程序缩放服务。与 Amazon Elastic Compute Cloud (Amazon EC2) Auto Scaling 组类似,Spotinst Ocean 旨在通过利用 Spot 实例与按需和预留实例的组合来优化性能和成本。通过自动化的 Spot 实例管理和各种实例大小的组合,Ocean 集群自动缩放器根据 pod 资源需求进行缩放。Spotinst Ocean 还包括一种预测算法,可在 Spot 实例中断前 15 分钟预测中断并在不同的 Spot 容量池中启动新节点。

这可作为 [AWS Quickstart](https://aws.amazon.com/quickstart/architecture/spotinst-ocean-eks/) 由 Spotinst, Inc. 与 AWS 合作开发。

EKS 研讨会也有一个关于[优化 Amazon EKS 管理的工作节点](https://eksworkshop.com/beginner/190_ocean/)的模块,其中包括成本分配、合理调整和缩放策略等部分。

### Yotascale

Yotascale 有助于准确分配 Kubernetes 成本。Yotascale Kubernetes 成本分配功能利用实际成本数据(包括预留实例折扣和 spot 实例定价),而不是一般市场价格估算,来了解 Kubernetes 成本足迹的总体情况。

更多详细信息可在[他们的网站](https://www.yotascale.com/)上找到。


### Alcide Advisor

Alcide 是 AWS 合作伙伴网络 (APN) 高级技术合作伙伴。Alcide Advisor 有助于确保您的 Amazon EKS 集群、节点和 pod 配置按照安全最佳实践和内部指南进行调整。Alcide Advisor 是一种无代理的 Kubernetes 审核和合规服务,旨在确保无摩擦和安全的 DevSecOps 流程,通过在进入生产环境之前加强开发阶段来实现这一目标。

更多详细信息可在此[博客文章](https://aws.amazon.com/blogs/apn/driving-continuous-security-and-configuration-checks-for-amazon-eks-with-alcide-advisor/)中找到。


## 其他工具


### Kubernetes 垃圾收集

[Kubernetes 垃圾收集器](https://kubernetes.io/docs/concepts/workloads/controllers/garbage-collection/)的作用是删除某些曾经有所有者但现在没有所有者的对象。

### Fargate 计数

[Fargatecount](https://github.com/mreferre/fargatecount) 是一个有用的工具,它允许 AWS 客户在特定区域的特定帐户中跟踪部署在 Fargate 上的 EKS pod 的总数,并使用自定义 CloudWatch 指标。这有助于跟踪 EKS 集群中运行的所有 Fargate pod。

### Kubernetes Ops View

[Kube Ops View](https://github.com/hjacobs/kube-ops-view) 是一个有用的工具,它为多个 Kubernetes 集群提供了一个通用的操作视图。

```
git clone https://github.com/hjacobs/kube-ops-view
cd kube-ops-view
kubectl apply -k deploy/
```

![主页](../images/kube-ops-report.png)


### Popeye - A Kubernetes 集群卫生检查器

[Popeye - A Kubernetes 集群卫生检查器](https://github.com/derailed/popeye)是一个扫描实时 Kubernetes 集群并报告部署资源和配置的潜在问题的实用程序。它基于部署的内容对集群进行卫生检查,而不是基于磁盘上的内容。通过扫描您的集群,它可以检测到配置错误,并帮助您确保最佳实践得到遵守。

### 资源
请参考以下资源了解有关成本优化最佳实践的更多信息。

文档和博客
+	[Amazon EKS 支持标记](https://docs.aws.amazon.com/eks/latest/userguide/eks-using-tags.html)

工具
+	[什么是 AWS 账单和成本管理?](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-alloc-tags.html)
+	[Amazon CloudWatch 容器洞察](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights.html)
+   [如何使用 Kubecost 跟踪 Amazon EKS 集群中的成本](https://aws.amazon.com/blogs/containers/how-to-track-costs-in-multi-tenant-amazon-eks-clusters-using-kubecost/) 
+   [Kube Cost](https://kubecost.com/)
+   [Kube Opsview](https://github.com/hjacobs/kube-ops-view)
+  [Kube Janitor](https://github.com/hjacobs/kube-janitor)
+  [Kubernetes Opex Analytics](https://github.com/rchakode/kube-opex-analytics)