# Amazon EKS 安全最佳实践指南

本指南提供了有关保护依赖于 EKS 的信息、系统和资产的建议,同时通过风险评估和缓解策略来提供业务价值。本指南是 AWS 发布的一系列最佳实践指南的一部分,旨在帮助客户根据最佳实践实施 EKS。性能、运营卓越、成本优化和可靠性的指南将在未来几个月内推出。

## 如何使用本指南

本指南面向负责实施和监控 EKS 集群及其支持的工作负载的安全控制有效性的安全从业者。本指南按主题领域进行组织,以便于使用。每个主题都以简要概述开始,后跟一系列建议和最佳实践,用于保护 EKS 集群。这些主题无需按特定顺序阅读。

## 了解共享责任模型

使用 EKS 等托管服务时,安全性和合规性被视为共享责任。一般而言,AWS 负责云的"安全性",而您(客户)负责云中的"安全性"。对于 EKS,AWS 负责管理 EKS 托管的 Kubernetes 控制平面。这包括 Kubernetes 控制平面节点、ETCD 数据库和 AWS 提供安全可靠服务所需的其他基础设施。作为 EKS 的消费者,您主要负责本指南中涉及的主题,例如 IAM、Pod 安全性、运行时安全性、网络安全性等。

在基础设施安全性方面,随着您从自管理工作节点过渡到托管节点组再到 Fargate,AWS 将承担更多责任。例如,使用 Fargate 时,AWS 负责保护用于运行 Pod 的底层实例/运行时。

![共享责任模型 - Fargate](images/SRM-EKS.jpg)

AWS 还将负责保持 EKS 优化的 AMI 与 Kubernetes 补丁版本和安全补丁保持同步。使用托管节点组(MNG)的客户负责通过 EKS API、CLI、CloudFormation 或 AWS 控制台将其节点组升级到最新的 AMI。与 Fargate 不同,MNG 不会自动扩展您的基础设施/集群。这可以由[集群自动缩放器](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md)或其他技术(如 [Karpenter](https://karpenter.sh/)、原生 AWS 自动缩放、SpotInst 的 [Ocean](https://spot.io/solutions/kubernetes-2/) 或 Atlassian 的 [Escalator](https://github.com/atlassian/escalator))来处理。

![共享责任模型 - MNG](./images/SRM-MNG.jpg)

在设计系统之前,了解您的责任与服务提供商(AWS)的责任之间的界限非常重要。

有关共享责任模型的更多信息,请参见 [https://aws.amazon.com/compliance/shared-responsibility-model/](https://aws.amazon.com/compliance/shared-responsibility-model/)

## 简介

使用 EKS 等托管 Kubernetes 服务时,有几个与安全性相关的最佳实践领域很重要:

- 身份和访问管理
- Pod 安全性
- 运行时安全性
- 网络安全性
- 多租户
- 多账户多租户
- 检测控制
- 基础设施安全性
- 数据加密和机密管理
- 法规合规性
- 事故响应和取证
- 镜像安全性

在设计任何系统时,您都需要考虑其安全性影响以及可能影响安全态势的实践。例如,您需要控制谁可以对一组资源执行操作。您还需要能够快速识别安全事件、保护系统和服务免受未经授权的访问,并通过数据保护维护数据的机密性和完整性。拥有一套定义明确且经过演练的安全事件响应流程也将提高您的安全态势。这些工具和技术很重要,因为它们支持诸如防止财务损失或遵守监管义务等目标。

AWS 通过提供一系列不断发展的安全服务,帮助组织实现其安全和合规目标,这些服务是基于广泛的安全意识客户的反馈而发展的。通过提供高度安全的基础,客户可以将更多时间集中在实现业务目标上,而不是处理"无差异的繁重工作"。

## 反馈

本指南在 GitHub 上发布,目的是收集来自更广泛的 EKS/Kubernetes 社区的直接反馈和建议。如果您有一个最佳实践认为应该包含在本指南中,请在 GitHub 存储库中提交问题或提交 PR。我们的目标是根据服务的新功能或新出现的最佳实践定期更新本指南。

## 进一步阅读

[Kubernetes 安全白皮书](https://github.com/kubernetes/sig-security/blob/main/sig-security-external-audit/security-audit-2019/findings/Kubernetes%20White%20Paper.pdf)由安全审计工作组赞助,该白皮书描述了 Kubernetes 攻击面和安全架构的关键方面,旨在帮助安全从业者做出合理的设计和实施决策。

CNCF 也发布了一份[白皮书](https://github.com/cncf/tag-security/blob/efb183dc4f19a1bf82f967586c9dfcb556d87534/security-whitepaper/v2/CNCF_cloud-native-security-whitepaper-May2022-v2.pdf)关于云原生安全。该论文检查了技术格局的发展,并倡导采用与 DevOps 流程和敏捷方法论相一致的安全实践。

## 工具和资源

[Amazon EKS 安全沉浸式研讨会](https://catalog.workshops.aws/eks-security-immersionday/en-US)