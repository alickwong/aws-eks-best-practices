# Amazon EKS 可靠性最佳实践指南

本节提供了关于使 EKS 上运行的工作负载具有弹性和高可用性的指导。

## 如何使用本指南

本指南面向希望在 EKS 中开发和运营高可用和容错服务的开发人员和架构师。本指南按不同的主题领域进行组织,以便于使用。每个主题都以简要概述开始,后跟 EKS 集群可靠性的一系列建议和最佳实践。

## 简介

EKS 的可靠性最佳实践被归类为以下几个主题:

* 应用程序
* 控制平面
* 数据平面

---

什么使系统可靠? 如果一个系统能够在其环境发生变化的情况下持续运行并满足需求,那么它就可以被称为可靠的。为实现这一点,系统必须能够检测故障、自动修复自身,并根据需求进行扩缩容。

客户可以使用 Kubernetes 作为可靠运行关键任务应用程序和服务的基础。但是,除了采用基于容器的应用程序设计原则外,可靠地运行工作负载还需要可靠的基础设施。在 Kubernetes 中,基础设施包括控制平面和数据平面。

EKS 提供了一个设计为高可用和容错的生产级 Kubernetes 控制平面。

在 EKS 中,AWS 负责 Kubernetes 控制平面的可靠性。EKS 在 AWS 区域内的三个可用区中运行 Kubernetes 控制平面。它自动管理 Kubernetes API 服务器和 etcd 集群的可用性和可扩展性。

数据平面的可靠性责任由您(客户)和 AWS 共同承担。EKS 为 Kubernetes 数据平面提供了三个选项。Fargate 是最受管理的选项,负责数据平面的配置和扩缩容。第二个选项是托管节点组,负责数据平面的配置和更新。最后,自管理节点是数据平面最不受管理的选项。您使用的 AWS 托管数据平面越多,您的责任就越少。

[托管节点组](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html)自动执行 EC2 节点的配置和生命周期管理。您可以使用 EKS API(通过 EKS 控制台、AWS API、AWS CLI、CloudFormation、Terraform 或 `eksctl`)创建、扩缩和升级托管节点。托管节点在您的账户中运行 EKS 优化的 Amazon Linux 2 EC2 实例,您可以通过启用 SSH 访问来安装自定义软件包。当您配置托管节点时,它们会作为跨多个可用区的 EKS 托管 Auto Scaling 组的一部分运行;您可以通过创建托管节点时提供的子网来控制这一点。EKS 还会自动为托管节点打标签,以便与集群自动缩放器一起使用。

> Amazon EKS 遵循共享责任模型来处理托管节点组中的 CVE 和安全补丁。由于托管节点运行 Amazon EKS 优化的 AMI,因此 Amazon EKS 负责构建这些 AMI 的修补版本。但是,您负责将这些修补后的 AMI 版本部署到您的托管节点组。

EKS 还[管理节点的更新](https://docs.aws.amazon.com/eks/latest/userguide/update-managed-node-group.html),尽管您必须启动更新过程。[更新托管节点](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-update-behavior.html)的过程在 EKS 文档中有解释。

如果您运行自管理节点,您可以使用 [Amazon EKS 优化的 Linux AMI](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html)来创建工作节点。您负责修补和升级 AMI 和节点。使用 `eksctl`、CloudFormation 或基础设施即代码工具来配置自管理节点是最佳实践,因为这将使您[升级自管理节点](https://docs.aws.amazon.com/eks/latest/userguide/update-workers.html)变得更加容易。在更新工作节点时,考虑[迁移到新节点](https://docs.aws.amazon.com/eks/latest/userguide/migrate-stack.html),因为迁移过程会将旧节点组标记为 `NoSchedule`,并在新堆栈准备好接受现有 pod 工作负载后排空节点。但是,您也可以执行[自管理节点的就地升级](https://docs.aws.amazon.com/eks/latest/userguide/update-stack.html)。

![共享责任模型 - Fargate](./images/SRM-Fargate.jpeg)

![共享责任模型 - MNG](./images/SRM-MNG.jpeg)

本指南包含了一系列建议,您可以使用它们来提高 EKS 数据平面、Kubernetes 核心组件和您的应用程序的可靠性。

## 反馈

本指南正在 GitHub 上发布,以收集 EKS/Kubernetes 社区的直接反馈和建议。如果您有一个最佳实践认为应该包含在本指南中,请在 GitHub 存储库中提交问题或提交 PR。我们打算根据服务的新功能或新的最佳实践的出现,定期更新本指南。