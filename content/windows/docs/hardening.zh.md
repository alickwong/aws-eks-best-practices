!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。
[简体中文]

# Windows 工作节点强化

操作系统强化是操作系统配置、修补和删除不必要软件包的组合,旨在锁定系统并减少攻击面。准备自己的 EKS 优化 Windows AMI 并应用公司所需的强化配置是最佳实践。

AWS 每月提供一个新的 EKS 优化 Windows AMI,其中包含最新的 Windows Server 安全补丁。但是,无论使用自管理还是托管节点组,用户仍有责任通过应用必要的操作系统配置来强化其 AMI。

微软提供了一系列工具,如[Microsoft 安全合规工具包](https://www.microsoft.com/en-us/download/details.aspx?id=55319)和[安全基线](https://docs.microsoft.com/en-us/windows/security/threat-protection/windows-security-baselines),可帮助您根据安全策略需求实现强化。[CIS 基准](https://learn.cisecurity.org/benchmarks?_gl=1*eoog69*_ga*MTgzOTM2NDE0My4xNzA0NDgwNTcy*_ga_3FW1B1JC98*MTcwNDQ4MDU3MS4xLjAuMTcwNDQ4MDU3MS4wLjAuMA..*_ga_N70Z2MKMD7*MTcwNDQ4MDU3MS4xLjAuMTcwNDQ4MDU3MS42MC4wLjA.)也可用,应在 Amazon EKS 优化 Windows AMI 的基础上实施,用于生产环境。

## 使用 Windows Server Core 减少攻击面

Windows Server Core 是一个最小安装选项,可作为 [EKS 优化 Windows AMI](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-windows-ami.html) 的一部分使用。部署 Windows Server Core 有几个好处。首先,它的磁盘占用较小,Server Core 为 6GB,而带桌面体验的 Windows Server 为 10GB。其次,由于代码库和可用 API 较小,它的攻击面也较小。

AWS 每月为客户提供新的 Amazon EKS 优化 Windows AMI,其中包含最新的 Microsoft 安全补丁,无论 Amazon EKS 支持的版本如何。作为最佳实践,Windows 工作节点必须根据最新的 Amazon EKS 优化 AMI 进行替换。任何运行时间超过 45 天且未进行更新或节点替换的节点都缺乏安全最佳实践。

## 避免 RDP 连接

远程桌面协议 (RDP) 是微软开发的一种连接协议,用于通过网络为用户提供连接到另一台 Windows 计算机的图形界面。

作为最佳实践,您应该将 Windows 工作节点视为临时主机。这意味着不进行任何管理连接、更新和故障排查。任何修改和更新都应作为新的自定义 AMI 实施,并通过更新自动缩放组进行替换。请参见**修补 Windows 服务器和容器**和**Amazon EKS 优化 Windows AMI 管理**。

通过在部署时将 ssh 属性的值设置为 **false** 来禁用 Windows 节点上的 RDP 连接,如下例所示:
```yaml 
nodeGroups:
- name: windows-ng
  instanceType: c5.xlarge
  minSize: 1
  volumeSize: 50
  amiFamily: WindowsServer2019CoreContainer
  ssh:
    allow: false
```

如果需要访问 Windows 节点,请使用 [AWS System Manager Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html) 通过 AWS 控制台和 SSM 代理建立一个安全的 PowerShell 会话。要了解如何实施该解决方案,请观看 [使用 AWS Systems Manager Session Manager 安全访问 Windows 实例](https://www.youtube.com/watch?v=nt6NTWQ-h6o)

为了使用 System Manager Session Manager,必须将额外的 IAM 策略应用于用于启动 Windows 工作节点的 IAM 角色。下面是一个示例,其中在 `eksctl` 集群清单中指定了 **AmazonSSMManagedInstanceCore**:
```yaml 
 nodeGroups:
- name: windows-ng
  instanceType: c5.xlarge
  minSize: 1
  volumeSize: 50
  amiFamily: WindowsServer2019CoreContainer
  ssh:
    allow: false
  iam:
    attachPolicyARNs:
      - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
      - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
      - arn:aws:iam::aws:policy/ElasticLoadBalancingFullAccess
      - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
      - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```


## Amazon Inspector
> [Amazon Inspector](https://aws.amazon.com/inspector/)是一项自动化安全评估服务,可帮助提高部署在AWS上的应用程序的安全性和合规性。Amazon Inspector自动评估应用程序的暴露、漏洞和偏离最佳实践的情况。在执行评估后,Amazon Inspector会生成一份详细的安全发现列表,按严重程度排序。这些发现可以直接查看,也可以作为详细的评估报告的一部分,通过Amazon Inspector控制台或API获取。

Amazon Inspector可用于在Windows工作节点上运行CIS基准评估,并可通过执行以下任务在Windows Server Core上安装:

1. 下载以下.exe文件:
https://inspector-agent.amazonaws.com/windows/installer/latest/AWSAgentInstall.exe
2. 将代理传输到Windows工作节点。
3. 在PowerShell上运行以下命令安装Amazon Inspector代理: `.\AWSAgentInstall.exe /install`

下面是第一次运行后的输出。如您所见,它根据[CVE](https://cve.mitre.org/)数据库生成了发现。您可以使用这些发现来加固您的工作节点或基于加固配置创建AMI。

![](./images/inspector-agent.png)

有关Amazon Inspector的更多信息,包括如何安装Amazon Inspector代理、设置CIS基准评估和生成报告,请观看[使用Amazon Inspector提高Windows工作负载的安全性和合规性](https://www.youtube.com/watch?v=nIcwiJ85EKU)视频。

## Amazon GuardDuty
> [Amazon GuardDuty](https://aws.amazon.com/guardduty/)是一项威胁检测服务,可持续监控恶意活动和未经授权的行为,以保护您的AWS帐户、工作负载和存储在Amazon S3中的数据。借助云计算,帐户和网络活动的收集和聚合变得更加简单,但安全团队持续分析事件日志数据以检测潜在威胁可能会耗费大量时间。

通过使用Amazon GuardDuty,您可以了解针对Windows工作节点的恶意活动,如RDP暴力攻击和端口探测攻击。

观看[使用Amazon GuardDuty检测Windows工作负载的威胁](https://www.youtube.com/watch?v=ozEML585apQ)视频,了解如何在优化的EKS Windows AMI上实施和运行CIS基准。

## Amazon EC2 for Windows的安全性
阅读[Amazon EC2 Windows实例的安全最佳实践](https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/ec2-security.html),在每一层实施安全控制。
