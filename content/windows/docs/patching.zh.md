!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

# 修补 Windows 服务器和容器

修补 Windows Server 是 Windows 管理员的标准管理任务。这可以通过使用不同的工具来完成,如 Amazon System Manager - Patch Manager、WSUS、System Center Configuration Manager 等。但是,Amazon EKS 集群中的 Windows 节点不应被视为普通的 Windows 服务器。它们应该被视为不可变的服务器。简单地说,避免更新现有节点,而是根据新的更新 AMI 启动一个新节点。

使用 [EC2 Image Builder](https://aws.amazon.com/image-builder/) 您可以自动化 AMI 构建,通过创建配方并添加组件。

以下示例显示了**组件**,这些组件可以是 AWS 构建的现有组件(由 Amazon 管理)以及您创建的组件(由我拥有)。请仔细注意名为**update-windows**的 Amazon 管理组件,它会在通过 EC2 Image Builder 管道生成 AMI 之前更新 Windows Server。

![](./images/associated-components.png)

EC2 Image Builder 允许您根据 Amazon 托管的公共 AMI 构建 AMI,并根据您的业务需求对其进行定制。然后,您可以将这些 AMI 与启动模板相关联,这允许您将新的 AMI 链接到 EKS 节点组创建的自动扩展组。完成后,您可以开始终止现有的 Windows 节点,新节点将根据新的更新 AMI 启动。

## 推送和拉取 Windows 镜像
Amazon 发布了 EKS 优化的 AMI,其中包含两个缓存的 Windows 容器镜像。

    mcr.microsoft.com/windows/servercore
    mcr.microsoft.com/windows/nanoserver

![](./images/images.png)

缓存的镜像会随着主操作系统的更新而更新。当 Microsoft 发布直接影响 Windows 容器基础镜像的新 Windows 更新时,该更新将作为普通的 Windows 更新启动。保持环境最新可以提供更安全的节点和容器级环境。

Windows 容器镜像的大小会影响推送/拉取操作,从而导致容器启动时间缓慢。[缓存 Windows 容器镜像](https://aws.amazon.com/blogs/containers/speeding-up-windows-container-launch-times-with-ec2-image-builder-and-image-cache-strategy/)允许在 AMI 构建创建时进行昂贵的 I/O 操作(文件提取),而不是在容器启动时进行。因此,所有必需的镜像层都将在 AMI 上提取,并准备就绪,从而加快 Windows 容器的启动时间和开始接受流量的时间。在推送操作期间,只有构成您的镜像的层才会上传到存储库。

以下示例显示,在 Amazon ECR 上,**fluentd-windows-sac2004**镜像只有**390.18MB**。这是在推送操作期间发生的上传量。
以下示例显示了一个[fluentd Windows ltsc](https://github.com/fluent/fluentd-docker-image/blob/master/v1.14/windows-ltsc2019/Dockerfile)镜像被推送到Amazon ECR存储库。存储在ECR中的层的大小为**533.05MB**。

![](./images/ecr-image.png)

从`docker image ls`的输出中可以看到,fluentd v1.14-windows-ltsc2019-1的磁盘大小为**6.96GB**,但这并不意味着下载和提取了这么多数据。

实际上,在拉取操作期间,只有**压缩后的533.05MB**数据会被下载和提取。
```bash
REPOSITORY                                                              TAG                        IMAGE ID       CREATED         SIZE
111122223333.dkr.ecr.us-east-1.amazonaws.com/fluentd-windows-coreltsc   latest                     721afca2c725   7 weeks ago     6.96GB
fluent/fluentd                                                          v1.14-windows-ltsc2019-1   721afca2c725   7 weeks ago     6.96GB
amazonaws.com/eks/pause-windows                                         latest                     6392f69ae6e7   10 months ago   255MB
```

大小列显示了图像的总体大小为6.96GB。分解如下:

* Windows Server Core 2019 LTSC Base 镜像 = 5.74GB
* Fluentd 未压缩的基础镜像 = 6.96GB
* 磁盘上的差异 = 1.2GB
* Fluentd [压缩的最终镜像 ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/repository-info.html) = 533.05MB

基础镜像已经存在于本地磁盘上,导致磁盘上的总量增加了1.2GB。下次您在大小列中看到GB数量时,不要太担心,很可能已经有超过70%的缓存容器镜像存在于磁盘上了。

## 参考
[使用 EC2 Image Builder 和镜像缓存策略加快 Windows 容器启动时间](https://aws.amazon.com/blogs/containers/speeding-up-windows-container-launch-times-with-ec2-image-builder-and-image-cache-strategy/)
