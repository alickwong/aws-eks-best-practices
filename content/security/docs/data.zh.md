!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。
数据加密和机密管理

静态加密

您可以使用三种不同的 AWS 原生存储选项与 Kubernetes 配合使用：[EBS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AmazonEBS.html)、[EFS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AmazonEFS.html) 和 [FSx for Lustre](https://docs.aws.amazon.com/fsx/latest/LustreGuide/what-is.html)。这三种选项都支持使用服务托管密钥或客户主密钥 (CMK) 进行静态加密。对于 EBS，您可以使用内置存储驱动程序或 [EBS CSI 驱动程序](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)。两者都包含用于加密卷和提供 CMK 的参数。对于 EFS，您可以使用 [EFS CSI 驱动程序](https://github.com/kubernetes-sigs/aws-efs-csi-driver)，但与 EBS 不同的是，EFS CSI 驱动程序不支持动态配置。如果您想在 EKS 中使用 EFS，则需要先预配置并设置文件系统的静态加密，然后再创建 PV。有关 EFS 文件加密的更多信息，请参阅[加密静态数据](https://docs.aws.amazon.com/efs/latest/ug/encryption-at-rest.html)。除了提供静态加密之外，EFS 和 FSx for Lustre 还包括加密传输数据的选项。FSx for Lustre 默认启用此功能。对于 EFS，您可以通过在 PV 的 `mountOptions` 中添加 `tls` 参数来启用传输加密,如下例所示:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  mountOptions:
    - tls
  csi:
    driver: efs.csi.aws.com
    volumeHandle: <file_system_id>
```

[FSx CSI 驱动程序](https://github.com/kubernetes-sigs/aws-fsx-csi-driver)支持动态配置Lustre文件系统。默认情况下,它使用服务托管密钥加密数据,尽管也可以提供自己的CMK,如本示例所示:
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: fsx-sc
provisioner: fsx.csi.aws.com
parameters:
  subnetId: subnet-056da83524edbe641
  securityGroupIds: sg-086f61ea73388fb6b
  deploymentType: PERSISTENT_1
  kmsKeyId: <kms_arn>
```


注意事项
自2020年5月28日起，在EKS Fargate pod中写入临时卷的所有数据默认使用行业标准的AES-256加密算法进行加密。无需对您的应用程序进行任何修改,因为加密和解密由服务自动处理。

加密静态数据
加密静态数据被视为最佳实践。如果您不确定是否需要加密,请加密您的数据。

定期轮换您的CMK
配置KMS以自动轮换您的CMK。这将每年轮换一次您的密钥,同时无限期保存旧密钥,以便您的数据仍可解密。有关更多信息,请参见[轮换客户主密钥](https://docs.aws.amazon.com/kms/latest/developerguide/rotate-keys.html)。

使用EFS访问点简化对共享数据集的访问
如果您有具有不同POSIX文件权限的共享数据集,或者想通过创建不同的挂载点来限制对共享文件系统的访问,请考虑使用EFS访问点。要了解有关使用访问点的更多信息,请参见[https://docs.aws.amazon.com/efs/latest/ug/efs-access-points.html](https://docs.aws.amazon.com/efs/latest/ug/efs-access-points.html)。目前,如果您想使用访问点(AP),您需要在PV的`volumeHandle`参数中引用AP。

注意事项
自2021年3月23日起,EFS CSI驱动程序支持EFS访问点的动态配置。访问点是进入EFS文件系统的特定于应用程序的入口点,可以更轻松地在多个pod之间共享文件系统。每个EFS文件系统最多可以有120个PV。有关更多信息,请参见[介绍Amazon EFS CSI动态配置](https://aws.amazon.com/blogs/containers/introducing-efs-csi-dynamic-provisioning/)。

机密管理
Kubernetes机密用于存储敏感信息,例如用户证书、密码或API密钥。它们以base64编码的字符串形式持久存储在etcd中。在EKS上,etcd节点的EBS卷使用[EBS加密](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSEncryption.html)进行加密。pod可以通过在`podSpec`中引用机密来检索Kubernetes机密对象。这些机密可以映射到环境变量或挂载为卷。有关创建机密的更多信息,请参见[https://kubernetes.io/docs/concepts/configuration/secret/](https://kubernetes.io/docs/concepts/configuration/secret/)。

注意
特定命名空间中的机密可以被该命名空间中的所有pod引用。

注意
节点授权器允许Kubelet读取挂载到节点上的所有机密。

使用AWS KMS进行Kubernetes机密的信封加密
这允许您使用唯一的数据加密密钥(DEK)加密您的机密。然后使用来自 AWS KMS 的密钥加密密钥(KEK)对 DEK 进行加密,该密钥可以根据定期计划自动轮换。通过 Kubernetes 的 KMS 插件,所有 Kubernetes 机密都以密文而不是纯文本存储在 etcd 中,只能由 Kubernetes API 服务器解密。
有关更多详细信息,请参见[使用 EKS 加密提供程序支持进行深度防御](https://aws.amazon.com/blogs/containers/using-eks-encryption-provider-support-for-defense-in-depth/)

### 审核 Kubernetes 机密的使用

在 EKS 上,启用审核日志记录并创建 CloudWatch 指标过滤器和警报,以在使用机密时发出警报(可选)。以下是 Kubernetes 审核日志的指标过滤器示例,`{($.verb="get") && ($.objectRef.resource="secret")}`。您也可以使用以下查询与 CloudWatch 日志洞察:
```bash
fields @timestamp, @message
| sort @timestamp desc
| limit 100
| stats count(*) by objectRef.name as secret
| filter verb="get" and objectRef.resource="secrets"
```

以上查询将显示在特定时间范围内密钥被访问的次数。
```bash
fields @timestamp, @message
| sort @timestamp desc
| limit 100
| filter verb="get" and objectRef.resource="secrets"
| display objectRef.namespace, objectRef.name, user.username, responseStatus.code
```


这个查询将显示密钥、尝试访问密钥的用户的命名空间和用户名以及响应代码。

### 定期轮换您的密钥

Kubernetes 不会自动轮换密钥。如果您需要轮换密钥,请考虑使用外部密钥存储,例如 Vault 或 AWS Secrets Manager。

### 使用单独的命名空间来隔离不同应用程序的密钥

如果您有一些密钥不能在同一个命名空间中的应用程序之间共享,请为这些应用程序创建一个单独的命名空间。

### 使用卷挂载而不是环境变量

环境变量的值可能会意外出现在日志中。以卷的形式挂载的密钥会以 tmpfs 卷(基于 RAM 的文件系统)的形式实例化,并在 pod 被删除时自动从节点上删除。

### 使用外部密钥提供程序

有几种可行的替代方案来代替使用 Kubernetes 密钥,包括 [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/) 和 Hashicorp 的 [Vault](https://www.hashicorp.com/blog/injecting-vault-secrets-into-kubernetes-pods-via-a-sidecar/)。这些服务提供了细粒度的访问控制、强加密和自动轮换密钥等功能,这些在 Kubernetes 密钥中是无法获得的。Bitnami 的 [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) 是另一种方法,它使用非对称加密来创建"密封的密钥"。公钥用于加密密钥,而用于解密密钥的私钥保存在集群内,允许您安全地将密封的密钥存储在 Git 等源代码控制系统中。请参阅[使用 Sealed Secrets 在 Kubernetes 中管理密钥部署](https://aws.amazon.com/blogs/opensource/managing-secrets-deployment-in-kubernetes-using-sealed-secrets/)以了解更多信息。

随着外部密钥存储的使用日益增加,与 Kubernetes 集成的需求也随之增加。[Secret Store CSI Driver](https://github.com/kubernetes-sigs/secrets-store-csi-driver) 是一个社区项目,它使用 CSI 驱动程序模型从外部密钥存储中获取密钥。目前,该驱动程序支持 [AWS Secrets Manager](https://github.com/aws/secrets-store-csi-driver-provider-aws)、Azure、Vault 和 GCP。AWS 提供程序同时支持 AWS Secrets Manager **和** AWS Parameter Store。它还可以配置为在密钥过期时轮换密钥,并将 AWS Secrets Manager 密钥同步到 Kubernetes 密钥。当您需要将密钥作为环境变量引用而不是从卷中读取它们时,密钥同步会很有用。

!!! note
    当 secret store CSI 驱动程序需要获取密钥时,它会假定分配给引用密钥的 pod 的 IRSA 角色。这个操作的代码可以在[这里](https://github.com/aws/secrets-store-csi-driver-provider-aws/blob/main/auth/auth.go)找到。
关于 AWS Secrets & Configuration Provider (ASCP) 的更多信息,请参考以下资源:

- [如何将 AWS Secrets Configuration Provider 与 Kubernetes Secret Store CSI 驱动程序一起使用](https://aws.amazon.com/blogs/security/how-to-use-aws-secrets-configuration-provider-with-kubernetes-secrets-store-csi-driver/)
- [将 Secrets Manager 密钥与 Kubernetes Secrets Store CSI 驱动程序集成](https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrating_csi_driver.html)

[external-secrets](https://github.com/external-secrets/external-secrets) 是另一种将外部密钥存储与 Kubernetes 一起使用的方式。与 CSI 驱动程序类似,external-secrets 可以与各种不同的后端进行交互,包括 AWS Secrets Manager。不同之处在于,external-secrets 不是从外部密钥存储中检索密钥,而是将这些后端的密钥复制到 Kubernetes 作为 Secrets。这使您可以使用首选的密钥存储来管理密钥,并以 Kubernetes 原生的方式与密钥进行交互。

## 工具和资源

- [Amazon EKS 安全沉浸式研讨会 - 数据加密和密钥管理](https://catalog.workshops.aws/eks-security-immersionday/en-US/13-data-encryption-and-secret-management)
