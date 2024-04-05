
# Amazon EKS 可靠性最佳实践指南

本节提供了关于使 EKS 上运行的工作负载具有弹性和高可用性的指导。

## 如何使用本指南

本指南面向希望在 EKS 中开发和运营高可用和容错服务的开发人员和架构师。该指南按不同的主题领域进行组织,以便于使用。每个主题都以简要概述开始,然后列出可靠性建议和最佳实践。

## 简介

EKS 可靠性最佳实践已归类为以下主题:

* 应用程序
* 控制平面
* 数据平面

---

什么使系统可靠? 如果一个系统能够在其环境发生变化的情况下持续运行并满足需求,那么它就可以被称为可靠的。为实现这一点,系统必须能够检测故障、自动修复自身,并根据需求进行扩缩容。

客户可以使用 Kubernetes 作为可靠运行关键任务应用程序和服务的基础。但是除了采用基于容器的应用程序设计原则外,可靠地运行工作负载还需要可靠的基础设施。在 Kubernetes 中,基础设施包括控制平面和数据平面。

EKS 提供了一个设计为高可用和容错的生产级 Kubernetes 控制平面。

在 EKS 中,AWS 负责 Kubernetes 控制平面的可靠性。EKS 将 Kubernetes 控制平面跨三个可用区在 AWS 区域内运行。它自动管理 Kubernetes API 服务器和 etcd 集群的可用性和可扩展性。

数据平面的可靠性责任由您(客户)和 AWS 共同承担。EKS 为 Kubernetes 数据平面提供三个选项。Fargate 是最托管的选项,负责数据平面的配置和扩缩容。第二个选项是托管节点组,负责数据平面的配置和更新。最后,自管理节点是数据平面最不托管的选项。您使用的 AWS 托管数据平面越多,您的责任就越少。

[托管节点组](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html)自动化了 EC2 节点的配置和生命周期管理。您可以使用 EKS API(通过 EKS 控制台、AWS API、AWS CLI、CloudFormation、Terraform 或 `eksctl`)创建、扩缩和升级托管节点。托管节点在您的账户中运行 EKS 优化的 Amazon Linux 2 EC2 实例,您可以通过启用 SSH 访问来安装自定义软件包。当您配置托管节点时,它们会作为跨多个可用区的 EKS 托管 Auto Scaling 组的一部分运行;您可以通过提供创建托管节点时的子网来控制这一点。EKS 还会自动为托管节点打标签,以便与集群自动缩放器一起使用。

> Amazon EKS 遵循共享责任模型来处理 CVE 和托管节点组上的安全补丁。由于托管节点运行 Amazon EKS 优化的 AMI,因此 Amazon EKS 负责构建这些 AMI 的修补版本。但是,您负责将这些修补后的 AMI 版本部署到您的托管节点组。

EKS 也[管理节点的更新](https://docs.aws.amazon.com/eks/latest/userguide/update-managed-node-group.html),尽管您必须启动更新过程。[更新托管节点](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-update-behavior.html)的过程在 EKS 文档中有解释。

如果您运行自管理节点,可以使用 [Amazon EKS 优化的 Linux AMI](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html) 创建工作节点。您负责修补和升级 AMI 和节点。使用 `eksctl`、CloudFormation 或基础设施即代码工具来配置自管理节点是最佳实践,因为这将使您更容易[升级自管理节点](https://docs.aws.amazon.com/eks/latest/userguide/update-workers.html)。在更新工作节点时,考虑[迁移到新节点](https://docs.aws.amazon.com/eks/latest/userguide/migrate-stack.html),因为迁移过程会将旧节点组标记为 `NoSchedule`,并在新堆栈准备好接受现有 pod 工作负载后排空节点。但是,您也可以对自管理节点执行[就地升级](https://docs.aws.amazon.com/eks/latest/userguide/update-stack.html)。

![Shared Responsibility Model - Fargate](./images/SRM-Fargate.jpeg)

![Shared Responsibility Model - MNG](./images/SRM-MNG.jpeg)

本指南包含一组建议,您可以使用它们来提高 EKS 数据平面、Kubernetes 核心组件和应用程序的可靠性。

## 反馈

本指南在 GitHub 上发布,以收集来自更广泛的 EKS/Kubernetes 社区的直接反馈和建议。如果您有一个最佳实践认为应该包含在指南中,请在 GitHub 存储库中提交问题或提交 PR。我们打算根据服务的新功能或新的最佳实践的出现,定期更新该指南。
