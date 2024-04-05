!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。
# Amazon EKS 优化的 Windows AMI 管理
Windows Amazon EKS 优化的 AMI 基于 Windows Server 2019 和 Windows Server 2022 构建。它们被配置为 Amazon EKS 节点的基础镜像。默认情况下,这些 AMI 包含以下组件:
- [kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)
- [kube-proxy](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-proxy/)
- [AWS IAM Authenticator for Kubernetes](https://github.com/kubernetes-sigs/aws-iam-authenticator)
- [csi-proxy](https://github.com/kubernetes-csi/csi-proxy)
- [containerd](https://containerd.io/)

您可以通过查询 AWS Systems Manager Parameter Store API 以编程方式检索 Amazon EKS 优化 AMI 的 Amazon 机器镜像 (AMI) ID。此参数消除了您手动查找 Amazon EKS 优化 AMI ID 的需要。有关 Systems Manager Parameter Store API 的更多信息,请参见 [GetParameter](https://docs.aws.amazon.com/systems-manager/latest/APIReference/API_GetParameter.html)。您的用户帐户必须具有 ssm:GetParameter IAM 权限才能检索 Amazon EKS 优化 AMI 元数据。

以下示例检索适用于 Windows Server 2019 LTSC Core 的最新 Amazon EKS 优化 AMI 的 AMI ID。AMI 名称中列出的版本号与它所准备的相应 Kubernetes 版本相关。
```bash    
aws ssm get-parameter --name /aws/service/ami-windows-latest/Windows_Server-2019-English-Core-EKS_Optimized-1.21/image_id --region us-east-1 --query "Parameter.Value" --output text
```

# 简化中文翻译

## 标题 2

这是一个段落。这是另一个段落。

![图像描述](image.jpg)

`这是代码块`

## 标题 3

这是一个列表:
- 列表项 1
- 列表项 2
- 列表项 3

> 这是引用块
```
ami-09770b3eec4552d4e
```


## 管理您自己的 Amazon EKS 优化 Windows AMI

维护相同的 Amazon EKS 优化 Windows AMI 和 kubelet 版本是走向生产环境的关键一步。在整个 Amazon EKS 集群中使用相同的版本可以减少故障排查的时间,并提高集群的一致性。[Amazon EC2 Image Builder](https://aws.amazon.com/image-builder/) 可帮助您创建和维护用于 Amazon EKS 集群的自定义 Windows AMI。

使用 Amazon EC2 Image Builder 可以选择 Windows Server 版本、AWS Windows Server AMI 发布日期和/或操作系统版本。构建组件步骤允许您选择现有的 EKS 优化 Windows 工件以及 kubelet 版本。更多信息请参见: https://docs.aws.amazon.com/eks/latest/userguide/eks-custom-ami-windows.html

![](./images/build-components.png)

**注意:** 在选择基础映像之前,请查阅 [Windows Server 版本和许可](licensing.md) 部分,了解有关发布渠道更新的重要详细信息。

## 为自定义 EKS 优化 AMI 配置更快的启动 ##

使用自定义 Windows Amazon EKS 优化 AMI 时,通过启用快速启动功能,Windows 工作节点的启动速度可提高高达 65%。此功能维护一组预置的快照,其中已完成 _Sysprep 专业化_、_Windows 开箱即用 (OOBE)_ 步骤和所需的重新启动。这些快照随后用于后续启动,从而缩短扩展或替换节点的时间。快速启动只能为您拥有的 AMI 启用,可通过 EC2 控制台或 AWS CLI 完成,并且可配置维护的快照数量。

**注意:** 快速启动与默认的 Amazon 提供的 EKS 优化 AMI 不兼容,请先按上述方式创建自定义 AMI 后再尝试启用。

更多信息请参见: [AWS Windows AMI - 为您的 AMI 配置更快的启动](https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/windows-ami-version-history.html#win-ami-config-fast-launch)

## 在自定义 AMI 上缓存 Windows 基础层 ##

Windows 容器镜像比其 Linux 对应物更大。如果您正在运行任何基于 .NET Framework 的容器化应用程序,平均镜像大小约为 8.24GB。在 pod 调度期间,必须完全拉取和提取容器镜像,然后 pod 才能进入 Running 状态。

在此过程中,容器运行时 (containerd) 会并行拉取和提取整个容器镜像。拉取操作是并行进行的,意味着容器运行时会并行拉取容器镜像层。相比之下,提取操作是顺序进行的,并且是 I/O 密集型的。因此,容器镜像的完全提取和准备供容器运行时 (containerd) 使用可能需要超过 8 分钟,从而导致 pod 启动时间长达几分钟。
正如在**修补 Windows Server 和容器**主题中提到的,有一个选项可以使用 EKS 构建自定义 AMI。在 AMI 准备期间,您可以添加一个额外的 EC2 Image Builder 组件,以在本地拉取所有必要的 Windows 容器映像,然后生成 AMI。这种策略将大大减少 pod 达到**运行**状态所需的时间。

在 Amazon EC2 Image Builder 上,创建一个[组件](https://docs.aws.amazon.com/imagebuilder/latest/userguide/manage-components.html)来下载必要的映像,并将其附加到映像配方。以下示例从 ECR 存储库中拉取特定映像。
```
name: ContainerdPull
description: This component pulls the necessary containers images for a cache strategy.
schemaVersion: 1.0

phases:
  - name: build
    steps:
      - name: containerdpull
        action: ExecutePowerShell
        inputs:
          commands:
            - Set-ExecutionPolicy Unrestricted -Force
            - (Get-ECRLoginCommand).Password | docker login --username AWS --password-stdin 111000111000.dkr.ecr.us-east-1.amazonaws.com
            - ctr image pull mcr.microsoft.com/dotnet/framework/aspnet:latest
            - ctr image pull 111000111000.dkr.ecr.us-east-1.amazonaws.com/myappcontainerimage:latest
```

为确保以下组件按预期工作,请检查 EC2 Image Builder (EC2InstanceProfileForImageBuilder) 使用的 IAM 角色是否具有附加的策略:

![](./images/permissions-policies.png)

## 博客文章 ##
在以下博客文章中,您将找到有关如何为自定义 Amazon EKS Windows AMI 实施缓存策略的分步指南:

[使用 EC2 Image Builder 和镜像缓存策略加快 Windows 容器启动时间](https://aws.amazon.com/blogs/containers/speeding-up-windows-container-launch-times-with-ec2-image-builder-and-image-cache-strategy/)
