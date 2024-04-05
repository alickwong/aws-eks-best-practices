!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。
# 审计和日志记录

收集和分析[审计]日志对于各种不同的原因都很有用。日志可以帮助进行根本原因分析和归因,即将某个变更归因于特定用户。当收集了足够多的日志后,它们也可用于检测异常行为。在 EKS 上,审计日志被发送到 Amazon Cloudwatch Logs。EKS 的审计策略如下:
```yaml
apiVersion: audit.k8s.io/v1beta1
kind: Policy
rules:
  # Log aws-auth configmap changes
  - level: RequestResponse
    namespaces: ["kube-system"]
    verbs: ["update", "patch", "delete"]
    resources:
      - group: "" # core
        resources: ["configmaps"]
        resourceNames: ["aws-auth"]
    omitStages:
      - "RequestReceived"
  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
      - group: "" # core
        resources: ["endpoints", "services", "services/status"]
  - level: None
    users: ["kubelet"] # legacy kubelet identity
    verbs: ["get"]
    resources:
      - group: "" # core
        resources: ["nodes", "nodes/status"]
  - level: None
    userGroups: ["system:nodes"]
    verbs: ["get"]
    resources:
      - group: "" # core
        resources: ["nodes", "nodes/status"]
  - level: None
    users:
      - system:kube-controller-manager
      - system:kube-scheduler
      - system:serviceaccount:kube-system:endpoint-controller
    verbs: ["get", "update"]
    namespaces: ["kube-system"]
    resources:
      - group: "" # core
        resources: ["endpoints"]
  - level: None
    users: ["system:apiserver"]
    verbs: ["get"]
    resources:
      - group: "" # core
        resources: ["namespaces", "namespaces/status", "namespaces/finalize"]
  - level: None
    users:
      - system:kube-controller-manager
    verbs: ["get", "list"]
    resources:
      - group: "metrics.k8s.io"
  - level: None
    nonResourceURLs:
      - /healthz*
      - /version
      - /swagger*
  - level: None
    resources:
      - group: "" # core
        resources: ["events"]
  - level: Request
    users: ["kubelet", "system:node-problem-detector", "system:serviceaccount:kube-system:node-problem-detector"]
    verbs: ["update","patch"]
    resources:
      - group: "" # core
        resources: ["nodes/status", "pods/status"]
    omitStages:
      - "RequestReceived"
  - level: Request
    userGroups: ["system:nodes"]
    verbs: ["update","patch"]
    resources:
      - group: "" # core
        resources: ["nodes/status", "pods/status"]
    omitStages:
      - "RequestReceived"
  - level: Request
    users: ["system:serviceaccount:kube-system:namespace-controller"]
    verbs: ["deletecollection"]
    omitStages:
      - "RequestReceived"
  # Secrets, ConfigMaps, and TokenReviews can contain sensitive & binary data,
  # so only log at the Metadata level.
  - level: Metadata
    resources:
      - group: "" # core
        resources: ["secrets", "configmaps"]
      - group: authentication.k8s.io
        resources: ["tokenreviews"]
    omitStages:
      - "RequestReceived"
  - level: Request
    resources:
      - group: ""
        resources: ["serviceaccounts/token"]
  - level: Request
    verbs: ["get", "list", "watch"]
    resources: 
      - group: "" # core
      - group: "admissionregistration.k8s.io"
      - group: "apiextensions.k8s.io"
      - group: "apiregistration.k8s.io"
      - group: "apps"
      - group: "authentication.k8s.io"
      - group: "authorization.k8s.io"
      - group: "autoscaling"
      - group: "batch"
      - group: "certificates.k8s.io"
      - group: "extensions"
      - group: "metrics.k8s.io"
      - group: "networking.k8s.io"
      - group: "policy"
      - group: "rbac.authorization.k8s.io"
      - group: "scheduling.k8s.io"
      - group: "settings.k8s.io"
      - group: "storage.k8s.io"
    omitStages:
      - "RequestReceived"
  # Default level for known APIs
  - level: RequestResponse
    resources: 
      - group: "" # core
      - group: "admissionregistration.k8s.io"
      - group: "apiextensions.k8s.io"
      - group: "apiregistration.k8s.io"
      - group: "apps"
      - group: "authentication.k8s.io"
      - group: "authorization.k8s.io"
      - group: "autoscaling"
      - group: "batch"
      - group: "certificates.k8s.io"
      - group: "extensions"
      - group: "metrics.k8s.io"
      - group: "networking.k8s.io"
      - group: "policy"
      - group: "rbac.authorization.k8s.io"
      - group: "scheduling.k8s.io"
      - group: "settings.k8s.io"
      - group: "storage.k8s.io"
    omitStages:
      - "RequestReceived"
  # Default level for all other requests.
  - level: Metadata
    omitStages:
      - "RequestReceived"
```

[简体中文翻译:]

## 建议

### 启用审核日志

审核日志是由 EKS 管理的 Kubernetes 控制平面日志的一部分。有关启用/禁用控制平面日志(包括 Kubernetes API 服务器、控制器管理器和调度程序的日志)以及审核日志的说明,请参见此处: [https://docs.aws.amazon.com/eks/latest/userguide/control-plane-logs.html#enabling-control-plane-log-export](https://docs.aws.amazon.com/eks/latest/userguide/control-plane-logs.html#enabling-control-plane-log-export)。

!!! info
    启用控制平面日志记录时,您将产生[存储日志](https://aws.amazon.com/cloudwatch/pricing/)在 CloudWatch 中的成本。这引发了一个更广泛的问题,即安全性的持续成本。最终,您将不得不权衡这些成本与安全漏洞造成的损失(如财务损失、声誉损害等)。您可能会发现,通过仅实施本指南中的部分建议,就可以充分保护您的环境。

!!! warning
    CloudWatch Logs 条目的最大大小为 [256KB](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/cloudwatch_limits_cwl.html),而 Kubernetes API 请求的最大大小为 1.5MiB。大于 256KB 的日志条目将被截断或仅包含请求元数据。

### 利用审核元数据

Kubernetes 审核日志包含两个注解,指示请求是否被授权(`authorization.k8s.io/decision`)以及决定的原因(`authorization.k8s.io/reason`)。使用这些属性来确定为什么允许特定的 API 调用。

### 创建可疑事件的警报

创建一个警报,以自动提醒您 403 Forbidden 和 401 Unauthorized 响应的增加,然后使用属性如 `host`、`sourceIPs` 和 `k8s_user.username` 来查找这些请求的来源。

### 使用日志洞察分析日志

使用 CloudWatch 日志洞察来监控 RBAC 对象(如 Roles、RoleBindings、ClusterRoles 和 ClusterRoleBindings)的变更。以下是一些示例查询:

列出对 `aws-auth` ConfigMap 的更新:
```bash
fields @timestamp, @message
| filter @logStream like "kube-apiserver-audit"
| filter verb in ["update", "patch"]
| filter objectRef.resource = "configmaps" and objectRef.name = "aws-auth" and objectRef.namespace = "kube-system"
| sort @timestamp desc
```

创建新的或更改验证 webhook 的列表:
```bash
fields @timestamp, @message
| filter @logStream like "kube-apiserver-audit"
| filter verb in ["create", "update", "patch"] and responseStatus.code = 201
| filter objectRef.resource = "validatingwebhookconfigurations"
| sort @timestamp desc
```

列表创建、更新、删除角色的操作:
```bash
fields @timestamp, @message
| sort @timestamp desc
| limit 100
| filter objectRef.resource="roles" and verb in ["create", "update", "patch", "delete"]
```

列表创建、更新、删除对 RoleBindings 的操作:
```bash
fields @timestamp, @message
| sort @timestamp desc
| limit 100
| filter objectRef.resource="rolebindings" and verb in ["create", "update", "patch", "delete"]
```

创建、更新、删除 ClusterRole 的操作:
```bash
fields @timestamp, @message
| sort @timestamp desc
| limit 100
| filter objectRef.resource="clusterroles" and verb in ["create", "update", "patch", "delete"]
```

创建、更新、删除 ClusterRoleBindings 的操作:
```bash
fields @timestamp, @message
| sort @timestamp desc
| limit 100
| filter objectRef.resource="clusterrolebindings" and verb in ["create", "update", "patch", "delete"]
```

未经授权的读取机密操作图表:
```bash
fields @timestamp, @message
| sort @timestamp desc
| limit 100
| filter objectRef.resource="secrets" and verb in ["get", "watch", "list"] and responseStatus.code="401"
| stats count() by bin(1m)
```

匿名请求失败列表:
```bash
fields @timestamp, @message, sourceIPs.0
| sort @timestamp desc
| limit 100
| filter user.username="system:anonymous" and responseStatus.code in ["401", "403"]
```


### 审核您的 CloudTrail 日志

使用 IAM 角色进行服务帐户 (IRSA) 的 pod 调用的 AWS API 会自动记录到 CloudTrail 中,并记录服务帐户的名称。如果未明确授权调用 API 的服务帐户名称出现在日志中,可能表示 IAM 角色的信任策略配置不当。总的来说,CloudTrail 是将 AWS API 调用归因于特定 IAM 主体的好方法。

### 使用 CloudTrail Insights 发现可疑活动

CloudTrail Insights 自动分析 CloudTrail 跟踪中的写入管理事件,并提醒您异常活动。这可以帮助您识别 AWS 帐户中写入 API 调用量增加的情况,包括使用 IRSA 承担 IAM 角色的 pod。有关更多信息,请参见[宣布推出 CloudTrail Insights:识别和响应异常 API 活动](https://aws.amazon.com/blogs/aws/announcing-cloudtrail-insights-identify-and-respond-to-unusual-api-activity/)。

### 其他资源

随着日志量的增加,使用 Log Insights 或其他日志分析工具进行解析和过滤可能会变得无效。作为替代方案,您可能想考虑运行 [Sysdig Falco](https://github.com/falcosecurity/falco) 和 [ekscloudwatch](https://github.com/sysdiglabs/ekscloudwatch)。Falco 分析审核日志并标记异常或滥用行为。ekscloudwatch 项目将 CloudWatch 中的审核日志事件转发到 Falco 进行分析。Falco 提供了一组[默认审核规则](https://github.com/falcosecurity/plugins/blob/master/plugins/k8saudit/rules/k8s_audit_rules.yaml),并可以添加您自己的规则。

另一个选择是将审核日志存储在 S3 中,并使用 SageMaker [Random Cut Forest](https://docs.aws.amazon.com/sagemaker/latest/dg/randomcutforest.html) 算法检测需要进一步调查的异常行为。

## 工具和资源

以下商业和开源项目可用于评估集群是否符合既定的最佳实践:

- [Amazon EKS 安全沉浸式研讨会 - 检测控制](https://catalog.workshops.aws/eks-security-immersionday/en-US/5-detective-controls)
- [kubeaudit](https://github.com/Shopify/kubeaudit)
- [kube-scan](https://github.com/octarinesec/kube-scan) 根据 Kubernetes 常见配置评分系统框架为集群中运行的工作负载分配风险评分
- [kubesec.io](https://kubesec.io/)
- [polaris](https://github.com/FairwindsOps/polaris)
- [Starboard](https://github.com/aquasecurity/starboard)
- [Snyk](https://support.snyk.io/hc/en-us/articles/360003916138-Kubernetes-integration-overview)
- [Kubescape](https://github.com/kubescape/kubescape) Kubescape是一个开源的Kubernetes安全工具,可扫描集群、YAML文件和Helm图表。它根据多个框架(包括[NSA-CISA](https://www.armosec.io/blog/kubernetes-hardening-guidance-summary-by-armo/?utm_source=github&utm_medium=repository)和[MITRE ATT&CK®](https://www.microsoft.com/security/blog/2021/03/23/secure-containerized-environments-with-updated-threat-matrix-for-kubernetes/))检测配置错误。
