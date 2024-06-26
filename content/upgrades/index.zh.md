!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

# 集群升级最佳实践

本指南向集群管理员展示如何规划和执行其Amazon EKS升级策略。它还描述了如何升级自管理节点、托管节点组、Karpenter节点和Fargate节点。它不包括对EKS Anywhere、自管理Kubernetes、AWS Outposts或AWS Local Zones的指导。

## 概述

Kubernetes版本包括控制平面和数据平面。为确保顺利运行,控制平面和数据平面应运行相同的[Kubernetes次版本,如1.24](https://kubernetes.io/releases/version-skew-policy/#supported-versions)。虽然AWS管理和升级控制平面,但更新数据平面上的工作节点是您的责任。

* **控制平面** - 控制平面的版本由Kubernetes API服务器确定。在Amazon EKS集群中,AWS负责管理此组件。可以通过AWS API启动控制平面升级。
* **数据平面** - 数据平面版本与运行在各个节点上的Kubelet版本相关联。同一集群中的节点可能运行不同的版本。您可以通过运行`kubectl get nodes`检查所有节点的版本。

## 升级前

如果您计划在Amazon EKS中升级Kubernetes版本,在开始升级之前,有几个重要的政策、工具和程序需要到位。

* **了解弃用政策** - 深入了解[Kubernetes弃用政策](https://kubernetes.io/docs/reference/using-api/deprecation-policy/)的工作原理。了解可能影响您现有应用程序的任何即将到来的变更。较新版本的Kubernetes通常会淘汰某些API和功能,可能会导致正在运行的应用程序出现问题。
* **审查Kubernetes变更日志** - 仔细查看[Kubernetes变更日志](https://github.com/kubernetes/kubernetes/tree/master/CHANGELOG)以及[Amazon EKS Kubernetes版本](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html),了解可能对您的集群产生影响的任何变更,例如可能影响您工作负载的突破性变更。
* **评估集群插件的兼容性** - Amazon EKS不会在发布新版本或您将集群更新到新的Kubernetes次版本后自动更新插件。查看[更新插件](https://docs.aws.amazon.com/eks/latest/userguide/managing-add-ons.html#updating-an-add-on)以了解任何现有集群插件与您打算升级到的集群版本的兼容性。

# 启用控制平面日志记录 - 启用[控制平面日志记录](https://docs.aws.amazon.com/eks/latest/userguide/control-plane-logs.html)以捕获升级过程中可能出现的日志、错误或问题。考虑查看这些日志是否存在任何异常。在非生产环境中测试集群升级,或将自动化测试集成到您的持续集成工作流中,以评估您的应用程序、控制器和自定义集成与新版本的兼容性。

# 探索 eksctl 进行集群管理 - 考虑使用 [eksctl](https://eksctl.io/) 管理您的 EKS 集群。它提供了[更新控制平面、管理附加组件和处理工作节点更新](https://eksctl.io/usage/cluster-upgrade/)的开箱即用功能。

# 选择托管节点组或 EKS on Fargate - 通过使用 [EKS 托管节点组](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html)或 [EKS on Fargate](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html)简化和自动化工作节点升级。这些选项简化了该过程,减少了手动干预。

# 利用 kubectl Convert 插件 - 利用 [kubectl convert 插件](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-convert-plugin)来促进[Kubernetes 清单文件](https://kubernetes.io/docs/tasks/tools/included/kubectl-convert-overview/)在不同 API 版本之间的转换。这可以帮助确保您的配置与新的 Kubernetes 版本保持兼容。

## 保持集群最新

保持 Kubernetes 更新是亚马逊 EKS 环境安全和高效的关键,反映了共享责任模型。通过将这些策略集成到您的操作工作流中,您将能够维护最新、安全的集群,并充分利用最新的功能和改进。策略:

* **支持的版本政策** - 与 Kubernetes 社区保持一致,亚马逊 EKS 通常提供三个活跃的 Kubernetes 版本,并在每年弃用一个第四个版本。弃用通知将在版本达到其停止支持日期前至少 60 天发布。更多详情,请参考 [EKS 版本常见问题解答](https://aws.amazon.com/eks/eks-version-faq/)。

* **自动升级政策** - 我们强烈建议保持 EKS 集群与 Kubernetes 更新同步。Kubernetes 社区支持,包括错误修复和安全补丁,通常会在一年后停止支持旧版本。弃用的版本也可能缺乏漏洞报告,存在潜在风险。未能及时在版本生命周期结束前升级,将触发自动升级,这可能会中断您的工作负载和系统。更多信息,请参考 [EKS 版本支持政策](https://aws.amazon.com/eks/eks-version-support-policy/)。

# 创建升级手册 — 建立一个记录完善的升级管理流程。作为您主动预防的一部分,开发针对您的升级流程定制的手册和专门工具。这不仅增强了您的准备能力,还简化了复杂的过渡。将集群升级至少每年一次作为标准做法。这种做法使您与持续的技术进步保持一致,从而提高环境的效率和安全性。

## 查看 EKS 发布日历

[查看 EKS Kubernetes 发布日历](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html#kubernetes-release-calendar)了解新版本何时发布,以及特定版本何时停止支持。通常,EKS 每年发布三个 Kubernetes 的次版本,每个次版本的支持时间约为 14 个月。

此外,请查看上游的 [Kubernetes 发布信息](https://kubernetes.io/releases/)。

## 了解共享责任模型如何应用于集群升级

您负责启动集群控制平面和数据平面的升级。[了解如何启动升级。](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html)当您启动集群升级时,AWS 将负责升级集群控制平面。您负责升级数据平面,包括 Fargate pod 和[其他附加组件。](#upgrade-add-ons-and-components-using-the-kubernetes-api)您必须验证并规划工作负载的升级,以确保集群升级后它们的可用性和操作不受影响。

## 就地升级集群

EKS 支持就地集群升级策略。这保留了集群资源,并保持集群配置的一致性(例如 API 端点、OIDC、ENI、负载均衡器)。这对集群用户的影响较小,并且将使用集群中现有的工作负载和资源,而无需重新部署工作负载或迁移外部资源(例如 DNS、存储)。

在执行就地集群升级时,请注意一次只能升级一个次版本(例如,从 1.24 升级到 1.25)。

这意味着,如果需要更新多个版本,将需要进行一系列顺序升级。规划顺序升级更加复杂,并且存在更高的停机风险。在这种情况下,[评估蓝/绿集群升级策略作为就地集群升级的替代方案。](#evaluate-bluegreen-clusters-as-an-alternative-to-in-place-cluster-upgrades)

## 按顺序升级控制平面和数据平面

要升级集群,您需要采取以下操作:

1. [查看 Kubernetes 和 EKS 发布说明。](#use-the-eks-documentation-to-create-an-upgrade-checklist)
2. [备份集群(可选)。](#backup-the-cluster-before-upgrading)

3. [识别和修复工作负载中已弃用和删除的 API 使用。](#identify-and-remediate-removed-api-usage-before-upgrading-the-control-plane)
4. [确保托管节点组（如果使用）与控制平面处于相同的 Kubernetes 版本。](#track-the-version-skew-of-nodes-ensure-managed-node-groups-are-on-the-same-version-as-the-control-plane-before-upgrading) EKS 托管节点组和由 EKS Fargate 配置文件创建的节点只支持控制平面和数据平面之间 1 个次版本的差异。
5. [使用 AWS 控制台或 CLI 升级集群控制平面。](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html)
6. [检查附加组件的兼容性。](#upgrade-add-ons-and-components-using-the-kubernetes-api) 根据需要升级您的 Kubernetes 附加组件和自定义控制器。
7. [更新 kubectl。](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)
8. [升级集群数据平面。](https://docs.aws.amazon.com/eks/latest/userguide/update-managed-node-group.html) 将您的节点升级到与升级后的集群相同的 Kubernetes 次版本。

## 使用 EKS 文档创建升级清单

EKS Kubernetes [版本文档](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html)包含每个版本的详细变更列表。为每次升级构建一个清单。

有关特定 EKS 版本升级指南，请查看文档中每个版本的重要变更和注意事项。

* [EKS 1.27](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html#kubernetes-1.27)
* [EKS 1.26](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html#kubernetes-1.26)
* [EKS 1.25](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html#kubernetes-1.25)
* [EKS 1.24](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html#kubernetes-1.24)
* [EKS 1.23](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html#kubernetes-1.23)
* [EKS 1.22](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html#kubernetes-1.22)

## 使用 Kubernetes API 升级附加组件和组件

在升级集群之前，您应该了解正在使用的 Kubernetes 组件的版本。盘点集群组件，并识别直接使用 Kubernetes API 的组件。这包括关键的集群组件，如监控和日志代理、集群自动缩放器、容器存储驱动程序（例如 [EBS CSI](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html)、[EFS CSI](https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html)）、入口控制器以及任何其他依赖于 Kubernetes API 的工作负载或附加组件。

!!! tip
    关键集群组件通常安装在 `*-system` 命名空间中
    
    ```
    kubectl get ns | grep '-system'
    ```

一旦您确定依赖Kubernetes API的组件,请检查它们的文档以了解版本兼容性和升级要求。例如,请参阅[AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/deploy/installation/)文档以了解版本兼容性。某些组件可能需要在集群升级之前进行升级或配置更改。需要检查的一些关键组件包括[CoreDNS](https://github.com/coredns/coredns)、[kube-proxy](https://kubernetes.io/docs/concepts/overview/components/#kube-proxy)、[VPC CNI](https://github.com/aws/amazon-vpc-cni-k8s)和存储驱动程序。

集群通常包含许多使用Kubernetes API的工作负载,这些工作负载对于工作负载功能(如入口控制器、持续交付系统和监控工具)是必需的。当您升级EKS集群时,您还必须升级您的附加组件和第三方工具,以确保它们与新版本兼容。

以下是一些常见附加组件及其相关升级文档的示例:

* **Amazon VPC CNI:** 有关每个集群版本的Amazon VPC CNI插件推荐版本,请参见[更新Kubernetes自管理附加组件的Amazon VPC CNI插件](https://docs.aws.amazon.com/eks/latest/userguide/managing-vpc-cni.html)。**当作为Amazon EKS附加组件安装时,它只能一次升级一个次版本。**
* **kube-proxy:** 请参见[更新Kubernetes kube-proxy自管理附加组件](https://docs.aws.amazon.com/eks/latest/userguide/managing-kube-proxy.html)。
* **CoreDNS:** 请参见[更新CoreDNS自管理附加组件](https://docs.aws.amazon.com/eks/latest/userguide/managing-coredns.html)。
* **AWS Load Balancer Controller:** AWS Load Balancer Controller需要与您部署的EKS版本兼容。有关更多信息,请参见[安装指南](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html)。
* **Amazon Elastic Block Store (Amazon EBS) Container Storage Interface (CSI) 驱动程序:** 有关安装和升级信息,请参见[作为Amazon EKS附加组件管理Amazon EBS CSI驱动程序](https://docs.aws.amazon.com/eks/latest/userguide/managing-ebs-csi.html)。
* **Amazon Elastic File System (Amazon EFS) Container Storage Interface (CSI) 驱动程序:** 有关安装和升级信息,请参见[Amazon EFS CSI驱动程序](https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html)。
* **Kubernetes Metrics Server:** 有关更多信息,请参见GitHub上的[metrics-server](https://kubernetes-sigs.github.io/metrics-server/)。
[简体中文]

* **Kubernetes 集群自动缩放器:** 要升级 Kubernetes 集群自动缩放器的版本,请更改部署中镜像的版本。集群自动缩放器与 Kubernetes 调度程序紧密耦合。在升级集群时,您始终需要升级它。查看 [GitHub 发布](https://github.com/kubernetes/autoscaler/releases)以找到与您的 Kubernetes 次版本对应的最新版本地址。
* **Karpenter:** 有关安装和升级信息,请参见 [Karpenter 文档](https://karpenter.sh/docs/upgrading/)。

## 在升级之前验证基本的 EKS 要求

AWS 要求您的帐户中存在某些资源才能完成升级过程。如果这些资源不存在,则无法升级集群。控制平面升级需要以下资源:

1. 可用 IP 地址: Amazon EKS 需要从您在创建集群时指定的子网中最多五个可用 IP 地址才能更新集群。如果没有,请在执行版本更新之前更新集群配置以包含新的集群子网。
2. EKS IAM 角色: 控制平面 IAM 角色仍然存在于帐户中,并具有必要的权限。
3. 如果您的集群启用了密钥加密,请确保集群 IAM 角色有权使用 AWS Key Management Service (AWS KMS) 密钥。

### 验证可用 IP 地址

要更新集群,Amazon EKS 需要从您在创建集群时指定的子网中最多五个可用 IP 地址。

要验证您的子网是否有足够的 IP 地址来升级集群,您可以运行以下命令:
```
CLUSTER=<cluster name>
aws ec2 describe-subnets --subnet-ids \
  $(aws eks describe-cluster --name ${CLUSTER} \
  --query 'cluster.resourcesVpcConfig.subnetIds' \
  --output text) \
  --query 'Subnets[*].[SubnetId,AvailabilityZone,AvailableIpAddressCount]' \
  --output table

----------------------------------------------------
|                  DescribeSubnets                 |
+---------------------------+--------------+-------+
|  subnet-067fa8ee8476abbd6 |  us-east-1a  |  8184 |
|  subnet-0056f7403b17d2b43 |  us-east-1b  |  8153 |
|  subnet-09586f8fb3addbc8c |  us-east-1a  |  8120 |
|  subnet-047f3d276a22c6bce |  us-east-1b  |  8184 |
+---------------------------+--------------+-------+
```

[VPC CNI 指标助手](https://github.com/aws/amazon-vpc-cni-k8s/blob/master/cmd/cni-metrics-helper/README.md)可用于创建 CloudWatch 仪表板以监控 VPC 指标。
Amazon EKS 建议在开始 Kubernetes 版本升级之前,如果您在最初创建集群时指定的子网中 IP 地址不足,请使用"UpdateClusterConfiguration"API 更新集群子网。请验证您将获得的新子网:

* 属于在集群创建期间选择的相同可用区集合。
* 属于在集群创建期间提供的相同 VPC。

如果现有 VPC CIDR 块中的 IP 地址耗尽,请考虑关联其他 CIDR 块。AWS 允许将其他 CIDR 块与现有集群 VPC 关联,从而有效地扩展 IP 地址池。此扩展可通过引入其他私有 IP 范围(RFC 1918)或必要时公共 IP 范围(非 RFC 1918)来实现。您必须添加新的 VPC CIDR 块并允许 VPC 刷新完成,然后 Amazon EKS 才能使用新的 CIDR。之后,您可以根据新设置的 CIDR 块更新 VPC 的子网。

### 验证 EKS IAM 角色

要验证 IAM 角色是否可用并在您的帐户中具有正确的信任策略,您可以运行以下命令:
```
CLUSTER=<cluster name>
ROLE_ARN=$(aws eks describe-cluster --name ${CLUSTER} \
  --query 'cluster.roleArn' --output text)
aws iam get-role --role-name ${ROLE_ARN##*/} \
  --query 'Role.AssumeRolePolicyDocument'
  
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "eks.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

## 迁移到 EKS 附加组件

Amazon EKS 自动为每个集群安装附加组件,如 Amazon VPC CNI 插件、`kube-proxy` 和 CoreDNS。附加组件可以是自管理的,也可以作为 Amazon EKS 附加组件安装。Amazon EKS 附加组件是使用 EKS API 管理附加组件的另一种方式。

您可以使用 Amazon EKS 附加组件通过单个命令更新版本。例如:
```
aws eks update-addon —cluster-name my-cluster —addon-name vpc-cni —addon-version version-number \
--service-account-role-arn arn:aws:iam::111122223333:role/role-name —configuration-values '{}' —resolve-conflicts PRESERVE
```

检查您是否有任何 EKS 附加组件:
```
aws eks list-addons --cluster-name <cluster name>
```


!!! warning
      
    EKS 附加组件在控制平面升级期间不会自动升级。您必须启动 EKS 附加组件更新,并选择所需的版本。

    * 您有责任从所有可用版本中选择一个兼容的版本。[查看有关附加组件版本兼容性的指南。](#upgrade-add-ons-and-components-using-the-kubernetes-api)
    * Amazon EKS 附加组件只能逐个次版本升级。

[了解更多关于 EKS 附加组件可用的组件以及如何入门的信息。](https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html)

[了解如何为 EKS 附加组件提供自定义配置。](https://aws.amazon.com/blogs/containers/amazon-eks-add-ons-advanced-configuration/)

## 在升级控制平面之前识别和修复已删除的 API 使用情况

您应该在升级 EKS 控制平面之前识别已删除 API 的使用情况。为此,我们建议使用可以检查正在运行的集群或静态呈现的 Kubernetes 清单文件的工具。

对静态清单文件进行检查通常更准确。如果对实时集群运行,这些工具可能会返回误报。

已弃用的 Kubernetes API 并不意味着该 API 已被删除。您应该查看 [Kubernetes 弃用政策](https://kubernetes.io/docs/reference/using-api/deprecation-policy/)以了解 API 删除如何影响您的工作负载。

### 集群洞察

[集群洞察](https://docs.aws.amazon.com/eks/latest/userguide/cluster-insights.html)是一项功能,可提供有关可能影响升级 EKS 集群到较新 Kubernetes 版本能力的问题的见解。这些见解由 Amazon EKS 策划和管理,并提供如何修复它们的建议。通过利用集群洞察,您可以最大限度地减少升级到较新 Kubernetes 版本所需的工作量。

要查看 EKS 集群的见解,您可以运行命令:
```
aws eks list-insights --region <region-code> --cluster-name <my-cluster>

{
    "insights": [
        {
            "category": "UPGRADE_READINESS", 
            "name": "Deprecated APIs removed in Kubernetes v1.29", 
            "insightStatus": {
                "status": "PASSING", 
                "reason": "No deprecated API usage detected within the last 30 days."
            }, 
            "kubernetesVersion": "1.29", 
            "lastTransitionTime": 1698774710.0, 
            "lastRefreshTime": 1700157422.0, 
            "id": "123e4567-e89b-42d3-a456-579642341238", 
            "description": "Checks for usage of deprecated APIs that are scheduled for removal in Kubernetes v1.29. Upgrading your cluster before migrating to the updated APIs supported by v1.29 could cause application impact."
        }
    ]
}
```

为了获得更加详细的洞见输出,您可以运行以下命令:
```
aws eks describe-insight --region <region-code> --id <insight-id> --cluster-name <my-cluster>
```

您也可以选择在 [Amazon EKS 控制台](https://console.aws.amazon.com/eks/home#/clusters) 中查看见解。在从集群列表中选择您的集群后，见解结果位于 ```升级见解``` 选项卡下。

如果您发现集群见解的 `"status": ERROR`，则必须在执行集群升级之前解决该问题。运行 `aws eks describe-insight` 命令将提供以下补救建议:

受影响的资源:
```
"resources": [
      {
        "insightStatus": {
          "status": "ERROR"
        },
        "kubernetesResourceUri": "/apis/policy/v1beta1/podsecuritypolicies/null"
      }
]
```

API已弃用:
```
"deprecationDetails": [
      {
        "usage": "/apis/flowcontrol.apiserver.k8s.io/v1beta2/flowschemas", 
        "replacedWith": "/apis/flowcontrol.apiserver.k8s.io/v1beta3/flowschemas", 
        "stopServingVersion": "1.29", 
        "clientStats": [], 
        "startServingReplacementVersion": "1.26"
      }
]
```

建议采取的行动:
```
"recommendation": "Update manifests and API clients to use newer Kubernetes APIs if applicable before upgrading to Kubernetes v1.26."
```

通过 EKS 控制台或 CLI 利用集群洞察力可以加快成功升级 EKS 集群版本的过程。请查看以下资源了解更多信息:
* [官方 EKS 文档](https://docs.aws.amazon.com/eks/latest/userguide/cluster-insights.html)
* [集群洞察力发布博客](https://aws.amazon.com/blogs/containers/accelerate-the-testing-and-verification-of-amazon-eks-upgrades-with-upgrade-insights/)。

### Kube-no-trouble

[Kube-no-trouble](https://github.com/doitintl/kube-no-trouble) 是一个开源命令行实用程序,带有 `kubent` 命令。当您运行 `kubent` 而不带任何参数时,它将使用您当前的 KubeConfig 上下文,扫描集群并打印一份报告,其中列出了哪些 API 将被弃用和删除。
```
kubent

4:17PM INF >>> Kube No Trouble `kubent` <<<
4:17PM INF version 0.7.0 (git sha d1bb4e5fd6550b533b2013671aa8419d923ee042)
4:17PM INF Initializing collectors and retrieving data
4:17PM INF Target K8s version is 1.24.8-eks-ffeb93d
4:l INF Retrieved 93 resources from collector name=Cluster
4:17PM INF Retrieved 16 resources from collector name="Helm v3"
4:17PM INF Loaded ruleset name=custom.rego.tmpl
4:17PM INF Loaded ruleset name=deprecated-1-16.rego
4:17PM INF Loaded ruleset name=deprecated-1-22.rego
4:17PM INF Loaded ruleset name=deprecated-1-25.rego
4:17PM INF Loaded ruleset name=deprecated-1-26.rego
4:17PM INF Loaded ruleset name=deprecated-future.rego
__________________________________________________________________________________________
>>> Deprecated APIs removed in 1.25 <<<
------------------------------------------------------------------------------------------
KIND                NAMESPACE     NAME             API_VERSION      REPLACE_WITH (SINCE)
PodSecurityPolicy   <undefined>   eks.privileged   policy/v1beta1   <removed> (1.21.0)
```

它也可用于扫描静态清单文件和 Helm 软件包。建议将 `kubent` 作为持续集成 (CI) 过程的一部分运行,以在部署清单之前识别问题。扫描清单比扫描实时集群更准确。

Kube-no-trouble 提供了一个示例 [Service Account 和 Role](https://github.com/doitintl/kube-no-trouble/blob/master/docs/k8s-sa-and-role-example.yaml),具有扫描集群所需的适当权限。

### Pluto

另一个选择是 [pluto](https://pluto.docs.fairwinds.com/)，它与 `kubent` 类似,因为它支持扫描实时集群、清单文件、Helm 图表,并且有一个可以包含在您的 CI 流程中的 GitHub Action。
```
pluto detect-all-in-cluster

NAME             KIND                VERSION          REPLACEMENT   REMOVED   DEPRECATED   REPL AVAIL  
eks.privileged   PodSecurityPolicy   policy/v1beta1                 false     true         true
```

### 资源

要在升级之前验证您的集群是否不使用已弃用的 API，您应该监控:

* 指标 `apiserver_requested_deprecated_apis` 自 Kubernetes v1.19 起:

    ![any text]
```
kubectl get --raw /metrics | grep apiserver_requested_deprecated_apis

apiserver_requested_deprecated_apis{group="policy",removed_release="1.25",resource="podsecuritypolicies",subresource="",version="v1beta1"} 1
```

* 在 [审核日志](https://docs.aws.amazon.com/eks/latest/userguide/control-plane-logs.html) 中具有 `k8s.io/deprecated` 设置为 `true` 的事件:
```
CLUSTER="<cluster_name>"
QUERY_ID=$(aws logs start-query \
 --log-group-name /aws/eks/${CLUSTER}/cluster \
 --start-time $(date -u --date="-30 minutes" "+%s") # or date -v-30M "+%s" on MacOS \
 --end-time $(date "+%s") \
 --query-string 'fields @message | filter `annotations.k8s.io/deprecated`="true"' \
 --query queryId --output text)

echo "Query started (query id: $QUERY_ID), please hold ..." && sleep 5 # give it some time to query

aws logs get-query-results --query-id $QUERY_ID
```

如果使用已弃用的 API，将输出以下行:
```
{
    "results": [
        [
            {
                "field": "@message",
                "value": "{\"kind\":\"Event\",\"apiVersion\":\"audit.k8s.io/v1\",\"level\":\"Request\",\"auditID\":\"8f7883c6-b3d5-42d7-967a-1121c6f22f01\",\"stage\":\"ResponseComplete\",\"requestURI\":\"/apis/policy/v1beta1/podsecuritypolicies?allowWatchBookmarks=true\\u0026resourceVersion=4131\\u0026timeout=9m19s\\u0026timeoutSeconds=559\\u0026watch=true\",\"verb\":\"watch\",\"user\":{\"username\":\"system:apiserver\",\"uid\":\"8aabfade-da52-47da-83b4-46b16cab30fa\",\"groups\":[\"system:masters\"]},\"sourceIPs\":[\"::1\"],\"userAgent\":\"kube-apiserver/v1.24.16 (linux/amd64) kubernetes/af930c1\",\"objectRef\":{\"resource\":\"podsecuritypolicies\",\"apiGroup\":\"policy\",\"apiVersion\":\"v1beta1\"},\"responseStatus\":{\"metadata\":{},\"code\":200},\"requestReceivedTimestamp\":\"2023-10-04T12:36:11.849075Z\",\"stageTimestamp\":\"2023-10-04T12:45:30.850483Z\",\"annotations\":{\"authorization.k8s.io/decision\":\"allow\",\"authorization.k8s.io/reason\":\"\",\"k8s.io/deprecated\":\"true\",\"k8s.io/removed-release\":\"1.25\"}}"
            },
[...]
```


## 更新 Kubernetes 工作负载。使用 kubectl-convert 更新清单

在确定需要更新哪些工作负载和清单后,您可能需要更改清单文件中的资源类型(例如从 PodSecurityPolicies 到 PodSecurityStandards)。这将需要更新资源规范并进行额外研究,具体取决于正在替换的资源。

如果资源类型保持不变,但 API 版本需要更新,您可以使用 `kubectl-convert` 命令自动转换您的清单文件。例如,将较旧的 Deployment 转换为 `apps/v1`。有关更多信息,请参见 Kubernetes 网站上的[安装 kubectl convert 插件](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-convert-plugin)。

`kubectl-convert -f <file> --output-version <group>/<version>`

## 配置 PodDisruptionBudgets 和 topologySpreadConstraints 以确保在数据平面升级期间工作负载的可用性

确保您的工作负载具有适当的 [PodDisruptionBudgets](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/#pod-disruption-budgets) 和 [topologySpreadConstraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints),以确保在数据平面升级期间工作负载的可用性。并非每个工作负载都需要相同级别的可用性,因此您需要验证工作负载的规模和要求。

确保工作负载分布在多个可用性区域和多个主机上,并使用拓扑传播,这将提高工作负载自动迁移到新数据平面的信心,而不会出现任何问题。

以下是一个示例工作负载,它将始终保持 80% 的副本可用,并在区域和主机之间传播副本:
```
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp
spec:
  minAvailable: "80%"
  selector:
    matchLabels:
      app: myapp
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 10
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - image: public.ecr.aws/eks-distro/kubernetes/pause:3.2
        name: myapp
        resources:
          requests:
            cpu: "1"
            memory: 256M
      topologySpreadConstraints:
      - labelSelector:
          matchLabels:
            app: host-zone-spread
        maxSkew: 2
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
      - labelSelector:
          matchLabels:
            app: host-zone-spread
        maxSkew: 2
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
```


[AWS 弹性枢纽](https://aws.amazon.com/resilience-hub/)已添加 Amazon Elastic Kubernetes Service (Amazon EKS) 作为受支持的资源。弹性枢纽提供了一个单一的地方来定义、验证和跟踪应用程序的弹性,以避免由于软件、基础设施或操作中断而造成的不必要的停机时间。

## 使用托管节点组或 Karpenter 简化数据平面升级

托管节点组和 Karpenter 都简化了节点升级,但它们采取了不同的方法。

托管节点组自动化了节点的配置和生命周期管理。这意味着您可以使用单个操作创建、自动更新或终止节点。

在默认配置中,Karpenter 会自动使用最新的兼容 EKS 优化 AMI 创建新节点。随着 EKS 发布更新的 EKS 优化 AMI 或集群升级,Karpenter 将自动开始使用这些镜像。[Karpenter 还实现了节点到期,以更新节点。](#enable-node-expiry-for-karpenter-managed-nodes)

[Karpenter 可以配置为使用自定义 AMI。](https://karpenter.sh/docs/concepts/nodeclasses/)如果您使用 Karpenter 的自定义 AMI,您需要负责 kubelet 的版本。

## 确认与现有节点和控制平面的版本兼容性

在 Amazon EKS 中进行 Kubernetes 升级之前,确保托管节点组、自管理节点和控制平面之间的兼容性至关重要。兼容性由您使用的 Kubernetes 版本决定,并根据不同的场景而有所不同。策略:

* **Kubernetes v1.28+** — **** 从 Kubernetes 版本 1.28 及以后,对核心组件的版本策略更加宽松。具体来说,Kubernetes API 服务器和 kubelet 之间支持的版本偏差已从 n-2 扩展到 n-3。例如,如果您的 EKS 控制平面版本为 1.28,您可以安全地使用最早 1.25 版本的 kubelet。这种版本偏差在 [AWS Fargate](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html)、[托管节点组](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html)和[自管理节点](https://docs.aws.amazon.com/eks/latest/userguide/worker.html)中都得到支持。我们强烈建议保持 [Amazon 机器镜像 (AMI)](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-amis.html) 版本的最新状态,以确保安全性。较旧的 kubelet 版本可能会由于潜在的常见漏洞和暴露 (CVE) 而带来安全风险,这可能会抵消使用较旧 kubelet 版本的好处。

[简体中文]

* **Kubernetes < v1.28** — 如果您使用的版本低于 v1.28，API 服务器和 kubelet 之间支持的版本偏差为 n-2。例如，如果您的 EKS 版本是 1.27，您可以使用的最旧的 kubelet 版本是 1.25。这种版本偏差适用于 [AWS Fargate](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html)、[托管节点组](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html)和[自管理节点](https://docs.aws.amazon.com/eks/latest/userguide/worker.html)。

## 为 Karpenter 托管节点启用节点过期

Karpenter 实现节点升级的一种方式是使用节点过期的概念。这减少了节点升级所需的规划。当您在 provisioner 中设置 **ttlSecondsUntilExpired** 值时，这会激活节点过期。在节点达到定义的秒数后，它们会被安全地排空和删除。即使它们正在使用中也是如此，这允许您用新配置的升级实例替换节点。当节点被替换时，Karpenter 使用最新的 EKS 优化 AMI。有关更多信息，请参见 Karpenter 网站上的[减配](https://karpenter.sh/docs/concepts/deprovisioning/#methods)。

Karpenter 不会自动为此值添加抖动。为了防止工作负载中断过多，请定义一个[pod 中断预算](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)，如 Kubernetes 文档所示。

如果您在 provisioner 上配置了 **ttlSecondsUntilExpired**，这将应用于与 provisioner 关联的现有节点。

## 对 Karpenter 托管节点使用漂移功能

[Karpenter 的漂移功能](https://karpenter.sh/docs/concepts/deprovisioning/#drift)可以自动升级 Karpenter 配置的节点，以保持与 EKS 控制平面的同步。Karpenter 漂移当前需要使用[功能门](https://karpenter.sh/docs/concepts/settings/#feature-gates)来启用。Karpenter 的默认配置使用与 EKS 集群控制平面相同的主版本和次版本的最新 EKS 优化 AMI。

EKS 集群升级完成后，Karpenter 的漂移功能将检测到 Karpenter 配置的节点使用的是上一个集群版本的 EKS 优化 AMI，并自动隔离、排空和替换这些节点。为了支持 pod 迁移到新节点，请遵循 Kubernetes 最佳实践，设置适当的 pod [资源配额](https://kubernetes.io/docs/concepts/policy/resource-quotas/)和使用 [pod 中断预算](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)(PDB)。Karpenter 的减配将根据 pod 资源请求预先启动替换节点，并在减配节点时尊重 PDB。

## 使用 eksctl 自动升级自管理节点组

自管理节点组是在您的帐户中部署并附加到集群之外的 EKS 服务的 EC2 实例。这些通常由某种形式的自动化工具部署和管理。要升级自管理节点组,您应该参考您的工具文档。

例如,eksctl 支持[删除和排空自管理节点](https://eksctl.io/usage/managing-nodegroups/#deleting-and-draining)。

一些常见的工具包括:

* [eksctl](https://eksctl.io/usage/nodegroup-upgrade/)
* [kOps](https://kops.sigs.k8s.io/operations/updates_and_upgrades/)
* [EKS Blueprints](https://aws-ia.github.io/terraform-aws-eks-blueprints/node-groups/#self-managed-node-groups)

## 在升级之前备份集群

Kubernetes 的新版本为您的 Amazon EKS 集群带来了重大变化。升级集群后,您无法降级。

[Velero](https://velero.io/) 是一个社区支持的开源工具,可用于备份现有集群并将备份应用于新集群。

请注意,您只能为 EKS 当前支持的 Kubernetes 版本创建新集群。如果您当前运行的版本仍受支持且升级失败,您可以使用原始版本创建一个新集群并恢复数据平面。请注意,Velero 不包括 AWS 资源(包括 IAM)在备份中。这些资源需要重新创建。

## 在升级控制平面后重新启动 Fargate 部署

要升级 Fargate 数据平面节点,您需要重新部署工作负载。您可以通过使用 `-o wide` 选项列出所有 pod 来识别在 Fargate 节点上运行的工作负载。任何以 `fargate-` 开头的节点名称都需要在集群中重新部署。

## 评估蓝/绿集群作为替代原地集群升级的方法

一些客户更喜欢使用蓝/绿升级策略。这可能会带来好处,但也包括应该考虑的缺点。

优点包括:

* 可以一次性更改多个 EKS 版本(例如从 1.23 升级到 1.25)
* 能够切换回旧集群
* 创建一个新集群,可以使用更新的系统(例如 Terraform)进行管理
* 工作负载可以逐个迁移

一些缺点包括:

* API 端点和 OIDC 发生变化,需要更新使用者(例如 kubectl 和 CI/CD)
* 需要在迁移期间并行运行两个集群,这可能会很昂贵并限制区域容量
* 如果工作负载相互依赖,需要更多协调才能一起迁移
* 负载均衡器和外部 DNS 无法轻松跨多个集群

虽然这种策略是可行的,但它比原地升级更昂贵,需要更多的协调和工作负载迁移时间。在某些情况下可能需要这样做,但应该仔细规划。

使用高度自动化和声明式系统(如GitOps)可能会更容易实现这一点。您需要采取额外的预防措施来处理有状态的工作负载,以确保数据得到备份并迁移到新的集群。

查看以下博客文章以获取更多信息:

* [Kubernetes集群升级:蓝绿部署策略](https://aws.amazon.com/blogs/containers/kubernetes-cluster-upgrade-the-blue-green-deployment-strategy/)
* [用于无状态ArgoCD工作负载的蓝/绿或金丝雀Amazon EKS集群迁移](https://aws.amazon.com/blogs/containers/blue-green-or-canary-amazon-eks-clusters-migration-for-stateless-argocd-workloads/)

## 跟踪Kubernetes项目中计划的重大变更 - 提前思考

不要只关注下一个版本。审查Kubernetes的新版本,并识别主要变更。例如,一些应用程序直接使用Docker API,而在Kubernetes `1.24`中删除了对Docker(也称为Dockershim)的Container Runtime Interface (CRI)的支持。这种变更需要更多时间来准备。

审查您要升级到的版本的所有记录的变更,并注意任何所需的升级步骤。此外,请注意Amazon EKS托管集群特有的任何要求或程序。

* [Kubernetes变更日志](https://github.com/kubernetes/kubernetes/tree/master/CHANGELOG)

## 关于功能删除的具体指导

### 1.25中删除Dockershim - 使用Docker Socket检测器(DDS)

EKS优化AMI 1.25不再包含对Dockershim的支持。如果您依赖于Dockershim,例如您正在挂载Docker套接字,则需要在将工作节点升级到1.25之前删除这些依赖项。

在升级到1.25之前,找出您在哪里依赖于Docker套接字。我们建议使用[Docker Socket检测器(DDS),一个kubectl插件](https://github.com/aws-containers/kubectl-detector-for-docker-socket)。

### 1.25中删除PodSecurityPolicy - 迁移到Pod安全标准或策略即代码解决方案

`PodSecurityPolicy`在Kubernetes 1.21中[被弃用](https://kubernetes.io/blog/2021/04/06/podsecuritypolicy-deprecation-past-present-and-future/),并在Kubernetes 1.25中被删除。如果您在集群中使用PodSecurityPolicy,则必须在将集群升级到1.25版本之前迁移到内置的Kubernetes Pod安全标准(PSS)或策略即代码解决方案,以避免您的工作负载中断。

AWS在EKS文档中发布了[详细的常见问题解答](https://docs.aws.amazon.com/eks/latest/userguide/pod-security-policy-removal-faq.html)。

查看[Pod安全标准(PSS)和Pod安全准入(PSA)](https://aws.github.io/aws-eks-best-practices/security/docs/pods/#pod-security-standards-pss-and-pod-security-admission-psa)最佳实践。
查看 Kubernetes 网站上的 [PodSecurityPolicy 弃用博客文章](https://kubernetes.io/blog/2021/04/06/podsecuritypolicy-deprecation-past-present-and-future/)。

### 1.23 中弃用内置存储驱动程序 - 迁移到容器存储接口 (CSI) 驱动程序

容器存储接口 (CSI) 旨在帮助 Kubernetes 替换其现有的内置存储驱动程序机制。Amazon EBS 容器存储接口 (CSI) 迁移功能默认在 Amazon EKS `1.23` 及更高版本的集群中启用。如果您在 `1.22` 或更早版本的集群上运行 pod，则必须在将集群升级到 `1.23` 版本之前安装 [Amazon EBS CSI 驱动程序](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html)，以避免服务中断。

查看 [Amazon EBS CSI 迁移常见问题解答](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi-migration-faq.html)。

## 其他资源

### ClowdHaus EKS 升级指南

[ClowdHaus EKS 升级指南](https://clowdhaus.github.io/eksup/)是一个 CLI 工具,可帮助升级 Amazon EKS 集群。它可以分析集群中任何需要修复的潜在问题,以便在升级之前进行修复。

### GoNoGo

[GoNoGo](https://github.com/FairwindsOps/GoNoGo)是一个处于 alpha 阶段的工具,用于确定集群附加组件的升级信心。
