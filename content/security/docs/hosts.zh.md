# 保护基础设施(主机)

与保护容器镜像同样重要的是,保护运行它们的基础设施也同样重要。本节探讨了如何缓解针对主机直接发起的攻击的不同方法。这些指南应与[运行时安全](runtime.md)部分概述的指南一起使用。

## 建议

### 使用针对容器运行优化的操作系统

考虑使用 Flatcar Linux、Project Atomic、RancherOS 和 [Bottlerocket](https://github.com/bottlerocket-os/bottlerocket/)，这是 AWS 专门为运行 Linux 容器而设计的操作系统。它包括减少的攻击面、在启动时验证的磁盘镜像以及使用 SELinux 强制执行的权限边界。

或者,对于您的 Kubernetes 工作节点,使用 [EKS 优化的 AMI][eks-ami]。EKS 优化的 AMI 定期发布,包含运行容器化工作负载所需的最小操作系统软件包和二进制文件。

[eks-ami]: https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-amis.html

请参考 [Amazon EKS AMI RHEL Build Specification](https://github.com/aws-samples/amazon-eks-ami-rhel),了解可用于使用 Hashicorp Packer 构建在 Red Hat Enterprise Linux 上运行的自定义 Amazon EKS AMI 的示例配置脚本。此脚本可进一步用于构建符合 STIG 要求的自定义 EKS AMI。

### 保持工作节点操作系统更新

无论您使用像 Bottlerocket 这样针对容器优化的主机操作系统,还是像 EKS 优化的 AMI 这样更大但仍极简的 Amazon 机器镜像,最佳做法都是保持这些主机操作系统映像与最新的安全补丁保持同步。

对于 EKS 优化的 AMI,请定期检查 [CHANGELOG][eks-ami-changes] 和/或 [发行说明渠道][eks-ami-releases],并将更新的工作节点映像自动部署到您的集群中。

[eks-ami-changes]: https://github.com/awslabs/amazon-eks-ami/blob/master/CHANGELOG.md
[eks-ami-releases]: https://github.com/awslabs/amazon-eks-ami/releases

### 将基础设施视为不可变,并自动替换工作节点

不要进行就地升级,而是在有新的补丁或更新可用时替换您的工作节点。这可以通过几种方式实现。您可以将实例添加到使用最新 AMI 的现有自动缩放组中,并依次隔离和排空节点,直到组中的所有节点都已替换为最新的 AMI。或者,您可以将实例添加到一个新的节点组中,同时依次隔离和排空旧节点组中的节点,直到所有节点都被替换。EKS [托管节点组](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html)使用第一种方法,并在控制台中显示消息,提示您在有新的 AMI 可用时升级工作节点。`eksctl` 也有一种机制,可以使用最新的 AMI 创建节点组,并优雅地隔离和排空节点组中的 pod,然后终止实例。如果您决定使用其他方法替换工作节点,强烈建议您自动化该过程,以最小化人工监督,因为您可能需要定期替换工作节点,以跟上新的更新/补丁发布以及控制平面升级。

对于 EKS Fargate,AWS 将在更新可用时自动更新底层基础设施。通常这可以无缝进行,但有时更新可能会导致 pod 被重新调度。因此,我们建议您在将应用程序作为 Fargate pod 运行时,创建具有多个副本的部署。

### 定期运行 kube-bench 以验证是否符合 [Kubernetes CIS 基准](https://www.cisecurity.org/benchmark/kubernetes/)

kube-bench 是 Aqua 的一个开源项目,用于评估您的集群是否符合 Kubernetes CIS 基准。该基准描述了保护未托管 Kubernetes 集群的最佳实践。CIS Kubernetes 基准涵盖了控制平面和数据平面。由于 Amazon EKS 提供了完全托管的控制平面,CIS Kubernetes 基准中并非所有建议都适用。为确保该范围反映了 Amazon EKS 的实现方式,AWS 创建了 *CIS Amazon EKS 基准*。EKS 基准继承自 CIS Kubernetes 基准,并从社区获取了有关 EKS 集群特定配置注意事项的其他输入。

在对 EKS 集群运行 [kube-bench](https://github.com/aquasecurity/kube-bench) 时,请遵循 Aqua Security 提供的[这些说明](https://github.com/aquasecurity/kube-bench/blob/main/docs/running.md#running-cis-benchmark-in-an-eks-cluster)。有关更多信息,请参见[介绍 CIS Amazon EKS 基准](https://aws.amazon.com/blogs/containers/introducing-cis-amazon-eks-benchmark/)。

### 最小化对工作节点的访问

不要启用 SSH 访问,而是使用 [SSM Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html) 远程访问主机。与可能丢失、复制或共享的 SSH 密钥不同,Session Manager 允许您使用 IAM 控制对 EC2 实例的访问。此外,它还提供了对在实例上运行的命令的审计跟踪和日志。

从 2020 年 8 月 19 日开始,托管节点组支持自定义 AMI 和 EC2 启动模板。这允许您将 SSM 代理嵌入到 AMI 中,或在引导工作节点时安装它。如果您不想修改优化的 AMI 或 ASG 的启动模板,可以使用 [此示例](https://github.com/aws-samples/ssm-agent-daemonset-installer) 中的 DaemonSet 安装 SSM 代理。

#### 用于基于 SSM 的 SSH 访问的最小 IAM 策略

`AmazonSSMManagedInstanceCore` AWS 托管策略包含了一些不需要用于 SSM Session Manager / SSM RunCommand 的权限,如果您只是想避免 SSH 访问。特别值得关注的是 `ssm:GetParameter(s)` 的 `*` 权限,这将允许该角色访问参数存储中的所有参数(包括使用 AWS 托管 KMS 密钥配置的 SecureStrings)。

以下 IAM 策略包含了通过 SSM Systems Manager 访问节点所需的最小权限集。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnableAccessViaSSMSessionManager",
      "Effect": "Allow",
      "Action": [
        "ssmmessages:OpenDataChannel",
        "ssmmessages:OpenControlChannel",
        "ssmmessages:CreateDataChannel",
        "ssmmessages:CreateControlChannel",
        "ssm:UpdateInstanceInformation"
      ],
      "Resource": "*"
    },
    {
      "Sid": "EnableSSMRunCommand",
      "Effect": "Allow",
      "Action": [
        "ssm:UpdateInstanceInformation",
        "ec2messages:SendReply",
        "ec2messages:GetMessages",
        "ec2messages:GetEndpoint",
        "ec2messages:FailMessage",
        "ec2messages:DeleteMessage",
        "ec2messages:AcknowledgeMessage"
      ],
      "Resource": "*"
    }
  ]
}
```

在应用此策略并安装 [Session Manager 插件](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)后,您可以运行

```bash
aws ssm start-session --target [EKS_NODE_INSTANCE_ID]
```

来访问节点。

!!! note
    您可能还想考虑添加权限来[启用 Session Manager 日志记录](https://docs.aws.amazon.com/systems-manager/latest/userguide/getting-started-create-iam-instance-profile.html#create-iam-instance-profile-ssn-logging)。

### 将工作节点部署到私有子网

通过将工作节点部署到私有子网,您可以最大程度地减少它们对互联网的暴露,互联网通常是攻击的来源。从 2020 年 4 月 22 日开始,托管节点组中节点的公共 IP 地址分配将由它们部署到的子网控制。在此之前,托管节点组中的节点会自动分配一个公共 IP。如果您选择将工作节点部署到公共子网,请实施限制性的 AWS 安全组规则,以限制它们的暴露。

### 运行 Amazon Inspector 评估主机的暴露、漏洞和偏离最佳实践的情况

您可以使用 [Amazon Inspector](https://docs.aws.amazon.com/inspector/latest/user/what-is-inspector.html) 检查您的节点是否存在意外的网络访问和底层 Amazon EC2 实例上的漏洞。

只有在 Amazon EC2 Systems Manager (SSM) 代理已安装并启用的情况下,Amazon Inspector 才能为您的 Amazon EC2 实例提供常见漏洞和暴露(CVE)数据。该代理预装在几个 [Amazon 机器镜像(AMI)](https://docs.aws.amazon.com/systems-manager/latest/userguide/ami-preinstalled-agent.html) 上,包括 [EKS 优化的 Amazon Linux AMI](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html)。无论 SSM 代理的状态如何,都会扫描所有 Amazon EC2 实例的网络可达性问题。有关为 Amazon EC2 配置扫描的更多信息,请参见[扫描 Amazon EC2 实例](https://docs.aws.amazon.com/inspector/latest/user/enable-disable-scanning-ec2.html)。

!!! attention
    Inspector 无法在运行 Fargate pod 的基础设施上运行。

## 替代方案

### 运行 SELinux

!!! info
    可用于 Red Hat Enterprise Linux (RHEL)、CentOS、Bottlerocket 和 Amazon Linux 2023

SELinux 为容器提供了一个额外的安全层,使它们彼此和主机隔离。SELinux 允许管理员为每个用户、应用程序、进程和文件强制执行强制访问控制(MAC)。可以将其视为一个后备,根据一组标签限制对特定资源的操作。在 EKS 上,SELinux 可用于防止容器访问彼此的资源。

容器 SELinux 策略定义在 [container-selinux](https://github.com/containers/container-selinux) 软件包中。Docker CE 需要这个软件包(以及它的依赖项),以便 Docker(或其他容器运行时)创建的进程和文件以有限的系统访问权限运行。容器利用 `container_t` 标签,这是 `svirt_lxc_net_t` 的别名。这些策略有效地防止容器访问主机的某些功能。

当您为 Docker 配置 SELinux 时,Docker 会自动将工作负载标记为 `container_t` 类型,并为每个容器分配一个唯一的 MCS 级别。这将隔离容器彼此之间。如果需要更宽松的限制,您可以在 SELinux 中创建自己的配置文件,授予容器对文件系统的特定区域的权限。这类似于 PSP,您可以为不同的容器/pod 创建不同的配置文件。例如,您可以为一般工作负载创建一个具有一组限制性控制的配置文件,为需要特权访问的内容创建另一个配置文件。

容器的 SELinux 有一组可以配置的选项来修改默认限制。可以根据需要启用或禁用以下 SELinux 布尔值:

| 布尔值 | 默认 | 描述 |
|---|:--:|---|
| `container_connect_any` | `off` | 允许容器访问主机上的特权端口。例如,如果您有一个需要将端口映射到主机上的 443 或 80 的容器。 |
| `container_manage_cgroup` | `off` | 允许容器管理 cgroup 配置。例如,运行 systemd 的容器将需要启用此功能。 |
| `container_use_cephfs` | `off` | 允许容器使用 ceph 文件系统。 |

默认情况下,容器被允许读取/执行 `/usr` 下的内容,并读取 `/etc` 的大部分内容。`/var/lib/docker` 和 `/var/lib/containers` 下的文件具有 `container_var_lib_t` 标签。要查看完整的默认标签列表,请参见 [container.fc](https://github.com/containers/container-selinux/blob/master/container.fc) 文件。

```bash
docker container run -it \
  -v /var/lib/docker/image/overlay2/repositories.json:/host/repositories.json \
  centos:7 cat /host/repositories.json
# cat: /host/repositories.json: Permission denied

docker container run -it \
  -v /etc/passwd:/host/etc/passwd \
  centos:7 cat /host/etc/passwd
# cat: /host/etc/passwd: Permission denied
```

标记为 `container_file_t` 的文件是容器可写的唯一文件。如果您想使卷挂载可写,您需要在末尾指定 `:z` 或 `:Z`。

- `:z` 将重新标记文件,以便容器可以读写
- `:Z` 将重新标记文件,以便**只有**容器可以读写

```bash
ls -Z /var/lib/misc
# -rw-r--r--. root root system_u:object_r:var_lib_t:s0   postfix.aliasesdb-stamp

docker container run -it \
  -v /var/lib/misc:/host/var/lib/misc:z \
  centos:7 echo "Relabeled!"

ls -Z /var/lib/misc
#-rw-r--r--. root root system_u:object_r:container_file_t:s0 postfix.aliasesdb-stamp
```

```bash
docker container run -it \
  -v /var/log:/host/var/log:Z \
  fluentbit:latest
```

在 Kubernetes 中,重新标记略有不同。不是让 Docker 自动重新标记文件,而是指定一个自定义的 MCS 标签来运行 pod。支持重新标记的卷将自动重新标记,以便它们可访问。具有匹配 MCS 标签的 pod 将能够访问该卷。如果需要严格隔离,请为每个 pod 设置不同的 MCS 标签。

```yaml
securityContext:
  seLinuxOptions:
    # 为每个容器提供一个唯一的 MCS 标签
    # 您也可以指定用户、角色和类型
    # 基于类型和级别(svert)的执行
    level: s0:c144:c154
```

在这个例子中,`s0:c144:c154` 对应于容器被允许访问的文件的 MCS 标签。

在 EKS 上,您可以创建允许运行特权容器(如 FluentD)的策略,并创建一个 SELinux 策略,允许它从主机的 /var/log 读取,而无需重新标记主机目录。具有相同标签的 pod 将能够访问相同的主机卷。

我们已经实现了[适用于 Amazon EKS 的示例 AMI](https://github.com/aws-samples/amazon-eks-custom-amis),它们在 CentOS 7 和 RHEL 7 上配置了 SELinux。这些 AMI 是为了演示满足高度监管客户(如 STIG、CJIS 和 C2S)要求的示例实现而开发的。

!!! caution
    SELinux 将忽略类型为 unconfined 的容器。

## 工具和资源

- [SELinux Kubernetes RBAC 和用于内部应用程序的安全策略传输](https://platform9.com/blog/selinux-kubernetes-rbac-and-shipping-security-policies-for-on-prem-applications/)
- [Kubernetes 的迭代强化](https://jayunit100.blogspot.com/2019/07/iterative-hardening-of-kubernetes-and.html)
- [Audit2Allow](https://linux.die.net/man/1/audit2allow)
- [SEAlert](https://linux.die.net/man/8/sealert)
- [使用 Udica 为容器生成 SELinux 策略](https://www.redhat.com/en/blog/generate-selinux-policies-containers-with-udica)描述了一个工具,它查看容器规范文件中的 Linux 功能、端口和挂载点,并生成一组 SELinux 规则,允许容器正常运行
- [AMI 强化](https://github.com/aws-samples/amazon-eks-custom-amis#hardening)剧本,用于满足不同监管要求的操作系统强化
- [Keiko Upgrade Manager](https://github.com/keikoproj/upgrade-manager),一个来自 Intuit 的开源项目,用于编排工作节点的轮换。
- [Sysdig Secure](https://sysdig.com/products/kubernetes-security/)
- [eksctl](https://eksctl.io/)