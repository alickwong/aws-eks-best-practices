# 事故响应和取证

您快速应对事故的能力可以帮助最大限度地减少违规造成的损害。拥有可靠的警报系统,能够警告您存在可疑行为,是良好事故响应计划的第一步。当事故发生时,您必须快速决定是销毁并替换受影响的容器,还是隔离和检查容器。如果您选择隔离容器作为取证调查和根源分析的一部分,则应遵循以下一系列活动:

## 示例事故响应计划

### 识别违规的 Pod 和工作节点

您的首要行动应该是隔离损害。首先确定违规发生的位置,并将该 Pod 及其节点与其余基础设施隔离。

### 使用工作负载名称识别违规的 Pod 和工作节点

如果您知道违规 pod 的名称和命名空间,可以按如下方式识别运行该 pod 的工作节点:

```bash
kubectl get pods <name> --namespace <namespace> -o=jsonpath='{.spec.nodeName}{"\n"}'   
```

如果 [工作负载资源](https://kubernetes.io/docs/concepts/workloads/controllers/) 如 Deployment 已被入侵,那么该工作负载资源中的所有 pod 可能都已被入侵。使用以下命令列出工作负载资源的所有 pod 及其运行所在的节点:

```bash
selector=$(kubectl get deployments <name> \
 --namespace <namespace> -o json | jq -j \
'.spec.selector.matchLabels | to_entries | .[] | "\(.key)=\(.value)"')

kubectl get pods --namespace <namespace> --selector=$selector \
-o json | jq -r '.items[] | "\(.metadata.name) \(.spec.nodeName)"'
```

上述命令适用于 Deployments。您可以对其他工作负载资源(如 ReplicaSets、StatefulSets 等)运行相同的命令。

### 使用服务帐户名称识别违规的 Pod 和工作节点

在某些情况下,您可能会发现服务帐户已被入侵。使用该服务帐户的 pod 很可能也已被入侵。您可以使用以下命令识别使用该服务帐户的所有 pod 及其运行所在的节点:

```bash
kubectl get pods -o json --namespace <namespace> | \
    jq -r '.items[] |
    select(.spec.serviceAccount == "<service account name>") |
    "\(.metadata.name) \(.spec.nodeName)"'
```

### 识别使用易受攻击或被入侵镜像的 Pod 和工作节点

在某些情况下,您可能会发现集群中的 pod 使用的容器镜像是恶意的或已被入侵。如果发现容器镜像包含恶意软件、是已知的恶意镜像或存在已被利用的 CVE,则应认为使用该镜像的所有 pod 都已被入侵。您可以使用以下命令识别使用该镜像的 pod 及其运行所在的节点:

```bash
IMAGE=<Name of the malicious/compromised image>

kubectl get pods -o json --all-namespaces | \
    jq -r --arg image "$IMAGE" '.items[] | 
    select(.spec.containers[] | .image == $image) | 
    "\(.metadata.name) \(.metadata.namespace) \(.spec.nodeName)"'
```

### 通过创建拒绝所有入口和出口流量的网络策略来隔离 Pod

拒绝所有流量的规则可能有助于阻止正在进行的攻击,方法是切断与 pod 的所有连接。以下网络策略将应用于带有标签 `app=web` 的 pod。

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
    如果攻击者已经获得了底层主机的访问权限,网络策略可能会失效。如果您怀疑这种情况已经发生,您可以使用 [AWS 安全组](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html)将受感染的主机与其他主机隔离。更改主机的安全组时,请注意它将影响在该主机上运行的所有容器。

### 如有必要,撤销分配给 pod 或工作节点的临时安全凭证

如果工作节点被分配了一个 IAM 角色,允许 pod 访问其他 AWS 资源,请删除该角色,以防止攻击造成进一步损害。同样,如果 pod 被分配了 IAM 角色,请评估是否可以安全地从该角色中删除 IAM 策略,而不会影响其他工作负载。

### 隔离工作节点

通过隔离受影响的工作节点,您可以告知调度程序避免在该节点上调度 pod。这将允许您在不中断其他工作负载的情况下,移除该节点进行取证研究。

!!! info
    此指南不适用于 Fargate,在 Fargate 中每个 pod 都在自己的沙箱环境中运行。不是隔离,而是通过应用拒绝所有入口和出口流量的网络策略来隔离受影响的 Fargate pod。

### 为受影响的工作节点启用终止保护

攻击者可能会试图通过终止受影响的节点来抹去他们的罪行。启用 [终止保护](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/terminating-instances.html#Using_ChangingDisableAPITermination) 可以防止这种情况发生。[实例缩减保护](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-instance-termination.html#instance-protection) 将保护该节点免受缩减事件的影响。

!!! warning
    您无法对 Spot 实例启用终止保护。

### 使用标签标记违规的 Pod/节点,表示正在进行调查

这将作为警告集群管理员,在调查完成之前不要篡改受影响的 pod/节点。

### 在工作节点上捕获易失性工件

- **捕获操作系统内存**。这将捕获 Docker 守护进程(或其他容器运行时)及其每个容器的子进程。这可以使用 [LiME](https://github.com/504ensicsLabs/LiME) 和 [Volatility](https://www.volatilityfoundation.org/) 等工具完成,或通过 [Automated Forensics Orchestrator for Amazon EC2](https://aws.amazon.com/solutions/implementations/automated-forensics-orchestrator-for-amazon-ec2/) 等更高级别的工具完成,这些工具建立在它们之上。
- **执行进程运行和打开端口的 netstat 树转储**。这将捕获 Docker 守护进程及其每个容器的子进程。
- **运行命令以在证据被改变之前保存容器级别的状态**。您可以利用容器运行时的功能来捕获当前正在运行的容器的信息。例如,使用 Docker,您可以执行以下操作:
  - `docker top CONTAINER` 查看正在运行的进程。
  - `docker logs CONTAINER` 获取守护进程级别保留的日志。
  - `docker inspect CONTAINER` 获取容器的各种信息。

    使用 [nerdctl](https://github.com/containerd/nerdctl) CLI 可以实现与 containerd 相同的功能,取代 `docker` (例如 `nerdctl inspect`)。根据容器运行时的不同,还可以使用一些额外的命令。例如,Docker 有 `docker diff` 查看容器文件系统的更改,或 `docker checkpoint` 保存包括易失性内存(RAM)在内的所有容器状态。请参阅 [此 Kubernetes 博客文章](https://kubernetes.io/blog/2022/12/05/forensic-container-checkpointing-alpha/) 了解有关 containerd 或 CRI-O 运行时类似功能的讨论。

- **暂停容器以进行取证捕获**。
- **快照实例的 EBS 卷**。

### 重新部署受损的 Pod 或工作负载资源

收集完取证分析所需的数据后,您可以重新部署受损的 pod 或工作负载资源。

首先部署修复了漏洞的替换 pod。然后删除易受攻击的 pod。

如果易受攻击的 pod 由较高级别的 Kubernetes 工作负载资源(例如 Deployment 或 DaemonSet)管理,删除它们将安排新的 pod。因此,易受攻击的 pod 将再次启动。在这种情况下,您应该在修复漏洞后部署新的替换工作负载资源,然后删除易受攻击的工作负载。

## 建议

### 查阅 AWS 安全事故响应白皮书

虽然本节简要概述了处理疑似安全漏洞的一些建议,但该主题在 [AWS 安全事故响应](https://docs.aws.amazon.com/whitepapers/latest/aws-security-incident-response-guide/welcome.html) 白皮书中有详尽的介绍。

### 进行安全游戏日

将您的安全从业者分为红队和蓝队。红队将专注于探测系统的各种漏洞,而蓝队将负责防御。如果您没有足够的安全从业者来组建单独的团队,可以考虑聘请具有 Kubernetes 漏洞知识的外部实体。

[Kubesploit](https://github.com/cyberark/kubesploit) 是 CyberArk 开发的一个渗透测试框架,您可以使用它来进行游戏日。与其他工具(扫描集群以发现漏洞)不同,kubesploit模拟了真实世界的攻击。这为您的蓝队提供了实践应对攻击的机会,并评估其有效性。

### 对集群进行渗透测试

定期攻击您自己的集群可以帮助您发现漏洞和配置错误。开始之前,请遵循 [渗透测试指南](https://aws.amazon.com/security/penetration-testing/) 在对集群进行测试之前。

## 工具和资源

- [kube-hunter](https://github.com/aquasecurity/kube-hunter),一个用于 Kubernetes 的渗透测试工具。
- [Gremlin](https://www.gremlin.com/product/#kubernetes),一个混沌工程工具包,您可以用它来模拟对应用程序和基础设施的攻击。
- [Attacking and Defending Kubernetes Installations](https://github.com/kubernetes/sig-security/blob/main/sig-security-external-audit/security-audit-2019/findings/AtredisPartners_Attacking_Kubernetes-v1.0.pdf)
- [kubesploit](https://www.cyberark.com/resources/threat-research-blog/kubesploit-a-new-offensive-tool-for-testing-containerized-environments)
- [NeuVector by SUSE](https://www.suse.com/neuvector/) 开源的零信任容器安全平台,提供漏洞和风险报告以及安全事件通知
- [Advanced Persistent Threats](https://www.youtube.com/watch?v=CH7S5rE3j8w)
- [Kubernetes Practical Attack and Defense](https://www.youtube.com/watch?v=LtCx3zZpOfs)
- [Compromising Kubernetes Cluster by Exploiting RBAC Permissions](https://www.youtube.com/watch?v=1LMo0CftVC4)