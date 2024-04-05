!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。
# 避免 OOM 错误

Windows 没有像 Linux 那样的内存不足进程终止器。Windows 总是将所有用户模式内存分配视为虚拟的，并且页面文件是强制性的。结果是，Windows 不会像 Linux 那样遇到内存不足的情况。进程将改为页面到磁盘，而不会受到内存不足 (OOM) 终止的影响。如果内存过度配置且所有物理内存都耗尽，那么页面操作可能会降低性能。

## 保留系统和 kubelet 内存
与 Linux 不同，`--kubelet-reserve` **捕获** Kubernetes 系统守护进程（如 kubelet、容器运行时等）的资源预留，`--system-reserve` **捕获** OS 系统守护进程（如 sshd、udev 等）的资源预留。在 **Windows** 上，这些标志不会**捕获**和**设置** **kubelet** 或节点上运行的**进程**的内存限制。

但是，您可以结合使用这些标志来管理 **NodeAllocatable**，以减少节点上的容量，并使用 Pod 清单 **内存资源限制**来控制每个 Pod 的内存分配。使用这种策略，您可以更好地控制内存分配,并提供一种机制来最小化 Windows 节点上的内存不足 (OOM)。

在 Windows 节点上,最佳做法是为操作系统和进程保留至少 2GB 内存。使用 `--kubelet-reserve` 和/或 `--system-reserve` 来减少 NodeAllocatable。

根据 [Amazon EKS 自管理 Windows 节点](https://docs.aws.amazon.com/eks/latest/userguide/launch-windows-workers.html)文档,使用 CloudFormation 模板启动一个新的 Windows 节点组,并对 kubelet 配置进行自定义。CloudFormation 有一个名为 `BootstrapArguments` 的元素,它与 `KubeletExtraArgs` 相同。使用以下标志和值:
```bash 
--kube-reserved memory=0.5Gi,ephemeral-storage=1Gi --system-reserved memory=1.5Gi,ephemeral-storage=1Gi --eviction-hard memory.available<200Mi,nodefs.available<10%"
```

如果 eksctl 是部署工具,请查看以下文档以自定义 kubelet 配置 https://eksctl.io/usage/customizing-the-kubelet/

## Windows 容器内存要求
根据 [Microsoft 文档](https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/system-requirements),NANO 的 Windows Server 基础映像至少需要 30MB,而 Server Core 需要 45MB。随着您添加 .NET Framework、Web 服务(如 IIS)和应用程序等 Windows 组件,这些数字会增加。

了解您的 Windows 容器映像(即基础映像加其应用程序层)所需的最小内存量非常重要,并将其设置为 pod 规范中容器的资源/请求。您还应该设置一个限制,以避免应用程序出现问题时 pod 消耗所有可用节点内存。

在下面的示例中,当 Kubernetes 调度程序尝试将 pod 放置在节点上时,将使用 pod 的请求来确定哪个节点有足够的可用资源进行调度。
```yaml 
 spec:
  - name: iis
    image: mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2019
    resources:
      limits:
        cpu: 1
        memory: 800Mi
      requests:
        cpu: .1
        memory: 128Mi
```

## 结论

使用这种方法可以最大程度地减少内存耗尽的风险,但无法完全防止其发生。通过使用Amazon CloudWatch Metrics,您可以设置警报和补救措施,以防内存耗尽发生。
