!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。
[持久性存储选项]

## 什么是内置(in-tree)与外置(out-of-tree)卷插件?

在容器存储接口(CSI)引入之前,所有卷插件都是内置的,意味着它们是与核心Kubernetes二进制文件一起构建、链接、编译和发布的,并扩展了核心Kubernetes API。这意味着向Kubernetes添加新的存储系统(卷插件)需要将代码检入核心Kubernetes代码存储库。

外置卷插件独立于Kubernetes代码库开发,并作为扩展部署(安装)在Kubernetes集群上。这使供应商能够独立于Kubernetes发布周期更新驱动程序。这在很大程度上是可能的,因为Kubernetes创建了一个存储接口或CSI,为供应商提供了一种与k8s标准化接口的方式。

您可以在https://docs.aws.amazon.com/eks/latest/userguide/storage.html上查看有关Amazon Elastic Kubernetes Services (EKS)存储类和CSI驱动程序的更多信息

## Windows的内置卷插件
Kubernetes卷使具有数据持久性要求的应用程序能够部署在Kubernetes上。持久卷的管理包括卷的配置/取消配置/调整大小、将卷附加/分离到/从Kubernetes节点,以及将卷挂载/卸载到/从pod中的单个容器。实现特定存储后端或协议的这些卷管理操作的代码以Kubernetes卷插件的形式发布**(内置卷插件)**。在Amazon Elastic Kubernetes Services (EKS)上,以下类别的Kubernetes卷插件在Windows上受支持:

*内置卷插件:* [awsElasticBlockStore](https://kubernetes.io/docs/concepts/storage/volumes/#awselasticblockstore)

为了在Windows节点上使用内置卷插件,有必要创建一个额外的StorageClass来使用NTFS作为fsType。在EKS上,默认的StorageClass使用ext4作为默认的fsType。

StorageClass为管理员提供了一种描述他们提供的"存储类"的方式。不同的类可能映射到不同的服务质量级别、备份策略或集群管理员确定的任意策略。Kubernetes对类代表什么没有任何意见。这个概念在其他存储系统中有时被称为"配置文件"。

您可以通过运行以下命令来检查:
```bash
kubectl describe storageclass gp2
```


# 简化中文翻译

## 简介

本文档提供了一个简单的 Markdown 文件格式转换为简体中文的示例。以下是一些要求:

- 如果行中包含 `![任何文本]`，则表示图像,不需要翻译
- 始终以以下格式回复: "您的翻译是: \n[翻译文本]"
- 不要尝试更改/修复 Markdown 符号
- 仅返回翻译后的文本
- 确保使用正式语气和行业专用术语
- 不要添加额外信息,如额外标题,只需要翻译
- 如果文本只有图像 Markdown 标记,仅翻译 `[]` 符号内的文本,不要添加额外标题
- 不要翻译 `` 符号内的文本
```bash
Name:            gp2
IsDefaultClass:  Yes
Annotations:     kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"storage.k8s.io/v1","kind":"StorageClas
","metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"},"name":"gp2"},"parameters":{"fsType"
"ext4","type":"gp2"},"provisioner":"kubernetes.io/aws-ebs","volumeBindingMode":"WaitForFirstConsumer"}
,storageclass.kubernetes.io/is-default-class=true
Provisioner:           kubernetes.io/aws-ebs
Parameters:            fsType=ext4,type=gp2
AllowVolumeExpansion:  <unset>
MountOptions:          <none>
ReclaimPolicy:         Delete
VolumeBindingMode:     WaitForFirstConsumer
Events:                <none>
```

要创建支持 **NTFS** 的新 StorageClass，请使用以下清单:
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: gp2-windows
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
  fsType: ntfs
volumeBindingMode: WaitForFirstConsumer
```

通过运行以下命令创建StorageClass:
```bash 
kubectl apply -f NTFSStorageClass.yaml
```

下一步是创建一个持久卷声明(PVC)。

持久卷(PV)是集群中的一块存储,由管理员提供或使用PVC动态配置。它是集群中的一种资源,就像节点是集群资源一样。这个API对象捕获了存储实现的细节,无论是NFS、iSCSI还是特定于云提供商的存储系统。

持久卷声明(PVC)是用户对存储的请求。声明可以请求特定的大小和访问模式(例如,它们可以被挂载为ReadWriteOnce、ReadOnlyMany或ReadWriteMany)。

用户需要具有不同属性(如性能)的持久卷,以满足不同的用例。集群管理员需要能够提供各种持久卷,这些持久卷在大小和访问模式以外的方式有所不同,而不需要让用户了解这些卷的实现细节。为了满足这些需求,有StorageClass资源。

在下面的示例中,PVC已在windows命名空间中创建。
```yaml 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ebs-windows-pv-claim
  namespace: windows
spec: 
  accessModes:
    - ReadWriteOnce
  storageClassName: gp2-windows
  resources: 
    requests:
      storage: 1Gi
```

创建 PVC 通过运行以下命令:
```bash 
kubectl apply -f persistent-volume-claim.yaml
```

以下清单创建了一个 Windows Pod，将 VolumeMount 设置为 `C:\Data`，并使用 PVC 作为附加到 `C:\Data` 的存储。
```yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: windows-server-ltsc2019
  namespace: windows
spec:
  selector:
    matchLabels:
      app: windows-server-ltsc2019
      tier: backend
      track: stable
  replicas: 1
  template:
    metadata:
      labels:
        app: windows-server-ltsc2019
        tier: backend
        track: stable
    spec:
      containers:
      - name: windows-server-ltsc2019
        image: mcr.microsoft.com/windows/servercore:ltsc2019
        ports:
        - name: http
          containerPort: 80
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - mountPath: "C:\\data"
          name: test-volume
      volumes:
        - name: test-volume
          persistentVolumeClaim:
            claimName: ebs-windows-pv-claim
      nodeSelector:
        kubernetes.io/os: windows
        node.kubernetes.io/windows-build: '10.0.17763'
```

测试通过 PowerShell 访问 Windows pod 的结果:
```bash 
kubectl exec -it podname powershell -n windows
```

在 Windows Pod 内运行: `ls`

输出:
```bash 
PS C:\> ls


    Directory: C:\


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----          3/8/2021   1:54 PM                data
d-----          3/8/2021   3:37 PM                inetpub
d-r---          1/9/2021   7:26 AM                Program Files
d-----          1/9/2021   7:18 AM                Program Files (x86)
d-r---          1/9/2021   7:28 AM                Users
d-----          3/8/2021   3:36 PM                var
d-----          3/8/2021   3:36 PM                Windows
-a----         12/7/2019   4:20 AM           5510 License.txt
```


**数据目录**由 EBS 卷提供。

## Windows 的树外

与 CSI 插件相关的代码作为树外脚本和二进制文件发货,通常作为容器镜像分发,并使用标准的 Kubernetes 构造(如 DaemonSets 和 StatefulSets)进行部署。CSI 插件在 Kubernetes 中处理广泛的卷管理操作。CSI 插件通常由节点插件(作为 DaemonSet 在每个节点上运行)和控制器插件组成。

CSI 节点插件(特别是那些作为块设备或通过共享文件系统公开的持久卷)需要执行各种特权操作,如扫描磁盘设备、挂载文件系统等。这些操作因主机操作系统而异。对于 Linux 工作节点,容器化的 CSI 节点插件通常部署为特权容器。对于 Windows 工作节点,使用 [csi-proxy](https://github.com/kubernetes-csi/csi-proxy)(一个由社区管理的独立二进制文件,需要预先安装在每个 Windows 节点上)支持容器化的 CSI 节点插件的特权操作。

[Amazon EKS 优化的 Windows AMI](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-windows-ami.html)从 2022 年 4 月开始包含 CSI-proxy。客户可以在 Windows 节点上使用 [SMB CSI 驱动程序](https://github.com/kubernetes-csi/csi-driver-smb)访问 [Amazon FSx for Windows File Server](https://aws.amazon.com/fsx/windows/)、[Amazon FSx for NetApp ONTAP SMB 共享](https://aws.amazon.com/fsx/netapp-ontap/)和/或 [AWS Storage Gateway – File Gateway](https://aws.amazon.com/storagegateway/file/)。

以下[博客](https://aws.amazon.com/blogs/modernizing-with-aws/using-smb-csi-driver-on-amazon-eks-windows-nodes/)提供了有关如何设置 SMB CSI 驱动程序以使用 Amazon FSx for Windows File Server 作为 Windows Pods 的持久存储的实施细节。

## Amazon FSx for Windows File Server

一个选择是通过一个名为 [SMB 全局映射](https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/persistent-storage)的 SMB 功能使用 Amazon FSx for Windows File Server,这使得可以在主机上挂载 SMB 共享,然后将该共享上的目录传递到容器中。容器不需要配置特定的服务器、共享、用户名或密码 - 这些都在主机上处理。容器的工作方式与使用本地存储相同。

> SMB 全局映射对编排器是透明的,并通过 HostPath 挂载,这可能会带来安全隐患。

在下面的示例中,路径 `G:\Directory\app-state` 是 Windows 节点上的 SMB 共享。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-fsx
spec:
  containers:
  - name: test-fsx
    image: mcr.microsoft.com/windows/servercore:ltsc2019
    command:
      - powershell.exe
      - -command
      - "Add-WindowsFeature Web-Server; Invoke-WebRequest -UseBasicParsing -Uri 'https://dotnetbinaries.blob.core.windows.net/servicemonitor/2.0.1.6/ServiceMonitor.exe' -OutFile 'C:\\ServiceMonitor.exe'; echo '<html><body><br/><br/><marquee><H1>Hello EKS!!!<H1><marquee></body><html>' > C:\\inetpub\\wwwroot\\default.html; C:\\ServiceMonitor.exe 'w3svc'; "
    volumeMounts:
      - mountPath: C:\dotnetapp\app-state
        name: test-mount
  volumes:
    - name: test-mount
      hostPath: 
        path: G:\Directory\app-state
        type: Directory
  nodeSelector:
      beta.kubernetes.io/os: windows
      beta.kubernetes.io/arch: amd64
```

以下[博客](https://aws.amazon.com/blogs/containers/using-amazon-fsx-for-windows-file-server-on-eks-windows-containers/)提供了如何将Amazon FSx for Windows File Server设置为Windows Pods的持久性存储的实施细节。
