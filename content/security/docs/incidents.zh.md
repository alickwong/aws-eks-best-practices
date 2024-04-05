!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。
# 事故响应和取证

您能够快速应对事故,有助于最大限度地减少违规造成的损害。拥有可靠的警报系统,能够警告您存在可疑行为,是良好事故响应计划的第一步。当事故发生时,您必须快速决定是销毁并替换受影响的容器,还是隔离和检查容器。如果您选择隔离容器作为取证调查和根源分析的一部分,则应遵循以下一系列活动:

## 示例事故响应计划

### 识别违规的 Pod 和工作节点

您的首要行动应该是隔离损害。首先确定违规发生的位置,并将该 Pod 及其节点与其余基础设施隔离。

### 使用工作负载名称识别违规的 Pod 和工作节点

如果您知道违规 pod 的名称和命名空间,可以按以下方式识别运行该 pod 的工作节点:
```bash
kubectl get pods <name> --namespace <namespace> -o=jsonpath='{.spec.nodeName}{"\n"}'   
```

如果一个[工作负载资源](https://kubernetes.io/docs/concepts/workloads/controllers/)（如Deployment）被入侵，那么该工作负载资源中的所有Pod很可能也已被入侵。使用以下命令列出工作负载资源的所有Pod及其运行所在的节点:
```bash
selector=$(kubectl get deployments <name> \
 --namespace <namespace> -o json | jq -j \
'.spec.selector.matchLabels | to_entries | .[] | "\(.key)=\(.value)"')

kubectl get pods --namespace <namespace> --selector=$selector \
-o json | jq -r '.items[] | "\(.metadata.name) \(.spec.nodeName)"'
```

以上命令适用于部署。您可以对其他工作负载资源(如副本集、有状态集等)运行相同的命令。

### 使用服务帐户名称识别有问题的 Pod 和工作节点

在某些情况下,您可能会发现某个服务帐户已被入侵。很可能使用该服务帐户的 Pod 也已被入侵。您可以使用以下命令识别使用该服务帐户的所有 Pod 以及它们所在的节点:
```bash
kubectl get pods -o json --namespace <namespace> | \
    jq -r '.items[] |
    select(.spec.serviceAccount == "<service account name>") |
    "\(.metadata.name) \(.spec.nodeName)"'
```

### 识别使用易受攻击或受损镜像的 Pod 和工作节点

在某些情况下,您可能会发现在集群上的 Pod 中使用的容器镜像是恶意的或已受损。如果发现容器镜像包含恶意软件、是已知的恶意镜像或存在已被利用的 CVE,则该容器镜像就被视为恶意或已受损。您应该认为所有使用该容器镜像的 Pod 都已受到损害。您可以使用以下命令识别使用该镜像的 Pod 和它们所在的节点:
```bash
IMAGE=<Name of the malicious/compromised image>

kubectl get pods -o json --all-namespaces | \
    jq -r --arg image "$IMAGE" '.items[] | 
    select(.spec.containers[] | .image == $image) | 
    "\(.metadata.name) \(.metadata.namespace) \(.spec.nodeName)"'
```

通过创建拒绝所有入口和出口流量的网络策略来隔离 Pod

拒绝所有流量的规则可能有助于通过切断与 Pod 的所有连接来阻止正在进行的攻击。以下网络策略将应用于带有标签 `app=web` 的 Pod。
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector:
    matchLabels: 
      app: web
  policyTypes:
  - Ingress
  - Egress
```


!!! attention
    如果攻击者已经获得了底层主机的访问权限,网络策略可能会失效。如果您怀疑这种情况已经发生,您可以使用[AWS安全组](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html)来隔离受感染的主机与其他主机。在更改主机的安全组时,请注意这将影响在该主机上运行的所有容器。

### 如果必要,撤销分配给pod或工作节点的临时安全凭证

如果工作节点被分配了一个IAM角色,允许Pod访问其他AWS资源,请从实例中删除这些角色,以防止攻击造成进一步损害。同样,如果Pod被分配了一个IAM角色,请评估是否可以安全地从该角色中删除IAM策略,而不会影响其他工作负载。

### 隔离受影响的工作节点

通过隔离受影响的工作节点,您可以告知调度程序避免在受影响的节点上调度pod。这将允许您在不中断其他工作负载的情况下,将该节点移除以进行取证分析。

!!! info
    这些指导不适用于Fargate,在Fargate中每个pod都在自己的沙箱环境中运行。相反,通过应用拒绝所有入口和出口流量的网络策略,将受影响的Fargate pod隔离起来。

### 为受影响的工作节点启用终止保护

攻击者可能会试图通过终止受影响的节点来掩盖他们的行为。启用[终止保护](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/terminating-instances.html#Using_ChangingDisableAPITermination)可以防止这种情况发生。[实例缩容保护](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-instance-termination.html#instance-protection)将保护该节点免受缩容事件的影响。

!!! warning
    您无法对Spot实例启用终止保护。

### 使用标签标记涉嫌有问题的Pod/节点

这将作为一个警告,提醒集群管理员在调查完成之前不要干扰受影响的Pod/节点。

### 捕获工作节点上的易失性工件

- **捕获操作系统内存**。这将捕获Docker守护进程(或其他容器运行时)及其每个容器的子进程。这可以使用像[LiME](https://github.com/504ensicsLabs/LiME)和[Volatility](https://www.volatilityfoundation.org/)这样的工具来完成,或者通过像[Automated Forensics Orchestrator for Amazon EC2](https://aws.amazon.com/solutions/implementations/automated-forensics-orchestrator-for-amazon-ec2/)这样的更高级工具来完成,这些工具建立在前两者之上。
- **执行进程运行和开放端口的netstat树转储**。这将捕获Docker守护进程及其每个容器的子进程。

- **在证据被篡改之前保存容器级状态的运行命令**。您可以利用容器运行时的功能来捕获正在运行的容器的信息。例如,使用Docker,您可以执行以下操作:
  - `docker top CONTAINER`查看正在运行的进程。
  - `docker logs CONTAINER`查看守护进程级别保存的日志。
  - `docker inspect CONTAINER`获取有关容器的各种信息。

    使用[nerdctl](https://github.com/containerd/nerdctl) CLI代替`docker`(例如`nerdctl inspect`)也可以实现相同的目标,这是使用containerd的情况。根据容器运行时的不同,还可能有一些其他命令可用。例如,Docker有`docker diff`命令查看容器文件系统的更改,或`docker checkpoint`命令保存包括易失性内存(RAM)在内的所有容器状态。请参阅[此Kubernetes博客文章](https://kubernetes.io/blog/2022/12/05/forensic-container-checkpointing-alpha/)了解有关containerd或CRI-O运行时类似功能的讨论。

- **暂停容器以进行取证采集**。
- **快照实例的EBS卷**。

### 重新部署受损的Pod或工作负载资源

收集完用于取证分析的数据后,您可以重新部署受损的pod或工作负载资源。

首先,部署修复了漏洞的版本,启动新的替换pod。然后删除易受攻击的pod。

如果易受攻击的pod由更高级别的Kubernetes工作负载资源(例如Deployment或DaemonSet)管理,删除它们将会调度新的pod。因此,易受攻击的pod将再次启动。在这种情况下,您应该在修复漏洞后部署新的替换工作负载资源,然后删除易受攻击的工作负载。

## 建议

### 查阅AWS安全事件响应白皮书

虽然本节简要概述了处理疑似安全漏洞的一些建议,但该主题在[AWS安全事件响应](https://docs.aws.amazon.com/whitepapers/latest/aws-security-incident-response-guide/welcome.html)白皮书中有详尽的介绍。

### 进行安全演练

将您的安全从业者分为红队和蓝队。红队将专注于探测不同系统的漏洞,而蓝队将负责防御。如果您没有足够的安全从业者来组建单独的团队,可以考虑聘请具有Kubernetes漏洞知识的外部实体。

[Kubesploit](https://github.com/cyberark/kubesploit)是CyberArk提供的一个渗透测试框架,您可以使用它来进行安全演练。与其他工具(扫描集群漏洞)不同,kubesploit模拟了真实世界的攻击。这为您的蓝队提供了实践应对攻击并评估其有效性的机会。

### 对集群进行渗透测试

定期攻击您自己的集群可以帮助您发现漏洞和配置错误。在开始之前,请遵循[渗透测试指南](https://aws.amazon.com/security/penetration-testing/),然后再对您的集群进行测试。

## 工具和资源

- [kube-hunter](https://github.com/aquasecurity/kube-hunter),一款用于Kubernetes的渗透测试工具。
- [Gremlin](https://www.gremlin.com/product/#kubernetes),一款混沌工程工具包,可用于模拟对您的应用程序和基础设施的攻击。
- [攻击和防御Kubernetes安装](https://github.com/kubernetes/sig-security/blob/main/sig-security-external-audit/security-audit-2019/findings/AtredisPartners_Attacking_Kubernetes-v1.0.pdf)
- [kubesploit](https://www.cyberark.com/resources/threat-research-blog/kubesploit-a-new-offensive-tool-for-testing-containerized-environments)
- [SUSE的NeuVector](https://www.suse.com/neuvector/)开源零信任容器安全平台,提供漏洞和风险报告以及安全事件通知
- [高级持续性威胁](https://www.youtube.com/watch?v=CH7S5rE3j8w)
- [Kubernetes实用攻击和防御](https://www.youtube.com/watch?v=LtCx3zZpOfs)
- [利用RBAC权限破坏Kubernetes集群](https://www.youtube.com/watch?v=1LMo0CftVC4)
