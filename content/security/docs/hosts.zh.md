!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

# 保护基础设施(主机)

与保护容器镜像同样重要的是,保护运行它们的基础设施也同样重要。本节探讨了如何缓解直接针对主机的攻击所带来的风险。这些指南应与[运行时安全](runtime.md)部分概述的指南一起使用。

## 建议

### 使用针对容器优化的操作系统

考虑使用Flatcar Linux、Project Atomic、RancherOS和[Bottlerocket](https://github.com/bottlerocket-os/bottlerocket/)等,这是AWS专门为运行Linux容器而设计的特殊用途操作系统。它包括减少攻击面、在启动时验证磁盘镜像以及使用SELinux强制执行权限边界。

或者,对于Kubernetes工作节点,使用[EKS优化的AMI][eks-ami]。EKS优化的AMI定期发布,包含运行容器化工作负载所需的最小操作系统软件包和二进制文件。

[eks-ami]: https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-amis.html

请参考[Amazon EKS AMI RHEL Build Specification](https://github.com/aws-samples/amazon-eks-ami-rhel),这是一个使用Hashicorp Packer构建在Red Hat Enterprise Linux上运行的自定义Amazon EKS AMI的示例配置脚本。可以进一步利用此脚本来构建符合STIG标准的自定义EKS AMI。

### 保持工作节点操作系统更新

无论您使用Bottlerocket等针对容器优化的主机操作系统,还是使用更大但仍尽可能精简的Amazon Machine Image,如EKS优化的AMI,最佳做法都是保持这些主机操作系统镜像与最新的安全补丁保持同步。

对于EKS优化的AMI,请定期检查[变更日志][eks-ami-changes]和/或[发布说明渠道][eks-ami-releases],并将更新的工作节点镜像自动部署到您的集群中。

[eks-ami-changes]: https://github.com/awslabs/amazon-eks-ami/blob/master/CHANGELOG.md
[eks-ami-releases]: https://github.com/awslabs/amazon-eks-ami/releases

### 将基础设施视为不可变,并自动替换工作节点

而不是执行就地升级,当有新的补丁或更新可用时,请更换您的工作人员。这可以通过几种方式来实现。您可以使用最新的 AMI 将实例添加到现有的自动缩放组,并依次隔离和排空节点,直到组中的所有节点都已使用最新的 AMI 进行了替换。或者,您可以在依次隔离和排空旧节点组中的节点直到全部替换完成的同时,将实例添加到新的节点组。EKS [托管节点组](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html)使用第一种方法,并在控制台中显示消息以在新 AMI 可用时升级您的工作人员。`eksctl`还有一种机制,可用于创建具有最新 AMI 的节点组,并在实例终止之前优雅地隔离和排空节点组中的 pod。如果您决定使用不同的方法来替换您的工作节点,强烈建议您自动化该过程,以最大程度地减少人工监督,因为您可能需要定期替换工作人员,因为发布了新的更新/补丁,并且在控制平面升级时。

使用 EKS Fargate,AWS 将在更新可用时自动更新底层基础设施。这通常可以无缝完成,但有时更新可能会导致您的 pod 被重新调度。因此,我们建议您在将应用程序作为 Fargate pod 运行时创建具有多个副本的部署。

### 定期运行 kube-bench 以验证是否符合 [Kubernetes CIS 基准](https://www.cisecurity.org/benchmark/kubernetes/)

kube-bench 是 Aqua 的一个开源项目,用于根据 Kubernetes CIS 基准评估您的集群。该基准描述了保护未托管 Kubernetes 集群的最佳实践。CIS Kubernetes 基准涵盖了控制平面和数据平面。由于 Amazon EKS 提供了完全托管的控制平面,CIS Kubernetes 基准中的并非所有建议都适用。为了确保此范围反映了 Amazon EKS 的实现方式,AWS 创建了 *CIS Amazon EKS 基准*。EKS 基准继承自 CIS Kubernetes 基准,并从社区获得了有关 EKS 集群特定配置注意事项的其他输入。

在对 EKS 集群运行 [kube-bench](https://github.com/aquasecurity/kube-bench) 时,请遵循 Aqua Security 的[这些说明](https://github.com/aquasecurity/kube-bench/blob/main/docs/running.md#running-cis-benchmark-in-an-eks-cluster)。有关更多信息,请参阅[介绍 CIS Amazon EKS 基准](https://aws.amazon.com/blogs/containers/introducing-cis-amazon-eks-benchmark/)。

### 最小化对工作节点的访问
[不要启用 SSH 访问,而是在需要远程访问主机时使用 [SSM Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)。与可能丢失、复制或共享的 SSH 密钥不同,Session Manager 允许您使用 IAM 控制对 EC2 实例的访问。此外,它还提供了对在实例上运行的命令的审计跟踪和日志。

截至 2020 年 8 月 19 日,托管节点组支持自定义 AMI 和 EC2 启动模板。这允许您将 SSM 代理嵌入 AMI 或在启动工作节点时安装它。如果您不想修改优化的 AMI 或 ASG 的启动模板,您可以按照[此示例](https://github.com/aws-samples/ssm-agent-daemonset-installer)中的方式使用 DaemonSet 安装 SSM 代理。

#### 基于 SSM 的 SSH 访问的最小 IAM 策略

`AmazonSSMManagedInstanceCore` AWS 托管策略包含了一些在仅需要避免 SSH 访问时并不需要的权限。特别值得关注的是 `ssm:GetParameter(s)` 的 `*` 权限,这将允许该角色访问参数存储中的所有参数(包括使用 AWS 托管 KMS 密钥配置的 SecureStrings)。

以下 IAM 策略包含了通过 SSM 系统管理器启用节点访问所需的最小权限集。]
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

有了这项政策和[会话管理器插件](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)安装后，您就可以运行
```bash
aws ssm start-session --target [INSTANCE_ID_OF_EKS_NODE]
```

[访问节点。

!!! note
    您还可能希望考虑添加权限以[启用会话管理器日志记录](https://docs.aws.amazon.com/systems-manager/latest/userguide/getting-started-create-iam-instance-profile.html#create-iam-instance-profile-ssn-logging)。

### 将工作人员部署到私有子网

通过将工作人员部署到私有子网，您可以最大限度地减少他们暴露在通常源于互联网的攻击中。从2020年4月22日开始，对托管节点组中节点的公共IP地址分配将由它们部署到的子网控制。在此之前，托管节点组中的节点会自动分配一个公共IP。如果您选择将工作节点部署到公共子网，请实施限制性的AWS安全组规则以限制其暴露。

### 运行Amazon Inspector以评估主机的暴露、漏洞和偏离最佳实践的情况

您可以使用[Amazon Inspector](https://docs.aws.amazon.com/inspector/latest/user/what-is-inspector.html)检查您的节点是否存在意外网络访问和底层Amazon EC2实例的漏洞。

只有在安装并启用Amazon EC2 Systems Manager (SSM)代理的情况下,Amazon Inspector才能为您的Amazon EC2实例提供常见漏洞和暴露(CVE)数据。该代理预安装在几个[Amazon Machine Images (AMIs)](https://docs.aws.amazon.com/systems-manager/latest/userguide/ami-preinstalled-agent.html)上,包括[EKS优化的Amazon Linux AMI](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html)。无论SSM代理状态如何,都会扫描您所有的Amazon EC2实例以检查网络可达性问题。有关配置Amazon EC2扫描的更多信息,请参见[扫描Amazon EC2实例](https://docs.aws.amazon.com/inspector/latest/user/enable-disable-scanning-ec2.html)。

!!! attention
    Inspector无法在用于运行Fargate pod的基础设施上运行。

## 替代方案

### 运行SELinux

!!! info
    可用于Red Hat Enterprise Linux (RHEL)、CentOS、Bottlerocket和Amazon Linux 2023

SELinux提供了一个额外的安全层,可以将容器彼此隔离,并与主机隔离。SELinux允许管理员为每个用户、应用程序、进程和文件强制执行强制访问控制(MAC)。可以将其视为一个后备,根据一组标签限制可以对特定资源执行的操作。在EKS上,SELinux可用于防止容器访问彼此的资源。]

容器 SELinux 策略在 [container-selinux](https://github.com/containers/container-selinux) 软件包中定义。Docker CE 需要此软件包(及其依赖项),以便 Docker(或其他容器运行时)创建的进程和文件以有限的系统访问权限运行。容器利用 `container_t` 标签,这是 `svirt_lxc_net_t` 的别名。这些策略有效地防止容器访问主机的某些功能。

当您为 Docker 配置 SELinux 时,Docker 会自动将工作负载标记为 `container_t` 类型,并为每个容器分配一个唯一的 MCS 级别。这将使容器彼此隔离。如果您需要更宽松的限制,可以在 SELinux 中创建自己的配置文件,该文件授予容器对文件系统特定区域的权限。这类似于 PSP,您可以为不同的容器/pod 创建不同的配置文件。例如,您可以有一个适用于一般工作负载的配置文件,其中包含一组限制性控制,另一个适用于需要特权访问的内容。

容器 SELinux 有一组可配置的选项,用于修改默认限制。可以根据需要启用或禁用以下 SELinux 布尔值:

| 布尔值 | 默认值 | 描述 |
|---|:--:|---|
| `container_connect_any` | `off` | 允许容器访问主机上的特权端口。例如,如果您有一个需要将端口映射到主机上的 443 或 80 的容器。 |
| `container_manage_cgroup` | `off` | 允许容器管理 cgroup 配置。例如,运行 systemd 的容器需要启用此功能。 |
| `container_use_cephfs` | `off` | 允许容器使用 ceph 文件系统。 |

默认情况下,容器被允许在 `/usr` 下读取/执行,并从 `/etc` 读取大部分内容。 `/var/lib/docker` 和 `/var/lib/containers` 下的文件具有 `container_var_lib_t` 标签。要查看默认标签的完整列表,请参见 [container.fc](https://github.com/containers/container-selinux/blob/master/container.fc) 文件。
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


带有 `container_file_t` 标签的文件是容器唯一可写的文件。如果您希望卷挂载可写,则需要在末尾指定 `:z` 或 `:Z`。

- `:z` 将重新标记文件,以便容器可以读/写
- `:Z` 将重新标记文件,以便**仅**容器可以读/写
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

在Kubernetes中,重新标记略有不同。与Docker自动重新标记文件不同,您可以指定自定义MCS标签来运行pod。支持重新标记的卷将自动重新标记,以便可以访问。具有匹配MCS标签的pod将能够访问该卷。如果需要严格隔离,请为每个pod设置不同的MCS标签。
```yaml
securityContext:
  seLinuxOptions:
    # Provide a unique MCS label per container
    # You can specify user, role, and type also
    # enforcement based on type and level (svert)
    level: s0:c144:c154
```


在本示例中，`s0:c144:c154`对应于分配给容器可以访问的文件的MCS标签。

在EKS上，您可以创建允许特权容器运行的策略,如FluentD,并创建一个SELinux策略,允许它从主机上的/var/log读取,而无需重新标记主机目录。具有相同标签的Pod将能够访问相同的主机卷。

我们已经实现了[适用于Amazon EKS的示例AMI](https://github.com/aws-samples/amazon-eks-custom-amis),它们在CentOS 7和RHEL 7上配置了SELinux。这些AMI是为了演示满足高度监管客户(如STIG、CJIS和C2S)要求的示例实现而开发的。

!!! 警告
    SELinux将忽略类型为unconfined的容器。

## 工具和资源

- [SELinux Kubernetes RBAC和为内部应用程序发布安全策略](https://platform9.com/blog/selinux-kubernetes-rbac-and-shipping-security-policies-for-on-prem-applications/)
- [Kubernetes的迭代强化](https://jayunit100.blogspot.com/2019/07/iterative-hardening-of-kubernetes-and.html)
- [Audit2Allow](https://linux.die.net/man/1/audit2allow)
- [SEAlert](https://linux.die.net/man/8/sealert)
- [使用Udica为容器生成SELinux策略](https://www.redhat.com/en/blog/generate-selinux-policies-containers-with-udica)描述了一个工具,它查看容器规范文件中的Linux功能、端口和挂载点,并生成一组SELinux规则,允许容器正常运行
- [AMI强化](https://github.com/aws-samples/amazon-eks-custom-amis#hardening)剧本,用于满足不同监管要求的操作系统强化
- [Keiko Upgrade Manager](https://github.com/keikoproj/upgrade-manager)是Intuit开源的一个项目,用于协调工作节点的轮换。
- [Sysdig Secure](https://sysdig.com/products/kubernetes-security/)
- [eksctl](https://eksctl.io/)
