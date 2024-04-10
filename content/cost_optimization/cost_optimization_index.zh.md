# Amazon EKS 成本优化最佳实践指南

成本优化是以最低价格实现您的业务目标。通过遵循本指南中的文档,您将优化您的 Amazon EKS 工作负载。

# 一般指南

在云中,有一些一般性指南可以帮助您实现微服务的成本优化:
+ 确保在 Amazon EKS 上运行的工作负载独立于运行容器的特定基础设施类型,这将在使用最便宜的基础设施类型运行它们方面提供更大的灵活性。在使用 Amazon EKS 和 EC2 时,当我们有需要特定类型 EC2 实例的工作负载时,例如[需要 GPU](https://docs.aws.amazon.com/eks/latest/userguide/gpu-ami.html)或其他实例类型,可能会有例外,这是由于工作负载的性质。
+ 选择优化配置的容器实例 - 对生产或预生产环境进行配置分析,并使用 [Amazon CloudWatch Container Insights for Amazon EKS](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/deploy-container-insights-EKS.html) 或 Kubernetes 生态系统中可用的第三方工具监控关键指标,如 CPU 和内存。这将确保我们可以分配正确数量的资源,避免资源浪费。
+ 利用 AWS 提供的不同采购选项来运行 EKS 和 EC2,例如按需、Spot 和 Savings Plan。

# EKS 成本优化最佳实践

云中成本优化有三个一般的最佳实践领域:

+ 成本效益资源(自动扩展、缩小规模、策略和采购选项)
+ 支出意识(使用 AWS 和第三方工具)
+ 随时间优化(合理调整)

与任何指南一样,都存在权衡。确保您与组织合作,了解此工作负载的优先级,并确定哪些最佳实践最为重要。

## 如何使用本指南

本指南面向负责实施和管理 EKS 集群及其支持的工作负载的 devops 团队。该指南按不同的最佳实践领域进行组织,以便于使用。每个主题都有一个建议列表、要使用的工具和成本优化 EKS 集群的最佳实践。这些主题不需要按特定顺序阅读。

### 关键 AWS 服务和 Kubernetes 功能
成本优化得到了以下 AWS 服务和功能的支持:
+ EC2 实例类型、Savings Plan(以前的预留实例)和 Spot 实例,价格不同。
+ 自动扩展以及 Kubernetes 原生自动扩展策略。考虑 Savings Plan(以前的预留实例)用于可预测的工作负载。使用 EBS 和 EFS 等托管数据存储,以实现应用程序数据的弹性和持久性。
+ 账单和成本管理控制台仪表板以及 AWS Cost Explorer 提供了 AWS 使用情况的概览。使用 AWS Organizations 获取细粒度的账单详细信息。还分享了几种第三方工具的详细信息。
+ Amazon CloudWatch Container Metrics 提供了有关 EKS 集群资源使用情况的指标。除了 Kubernetes 仪表板,Kubernetes 生态系统中还有几种工具可用于减少浪费。

本指南包含一组建议,您可以使用它们来提高 Amazon EKS 集群的成本优化。

## 反馈
本指南在 GitHub 上发布,以便从更广泛的 EKS/Kubernetes 社区收集直接反馈和建议。如果您有一个最佳实践认为应该包含在指南中,请在 GitHub 存储库中提交问题或提交 PR。我们的目的是根据服务的新功能或新的最佳实践的出现,定期更新该指南。