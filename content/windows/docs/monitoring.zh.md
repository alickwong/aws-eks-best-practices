!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

# 监控

Prometheus，一个[CNCF毕业项目](https://www.cncf.io/projects/)，是目前最受欢迎的监控系统,原生集成到Kubernetes。Prometheus收集有关容器、Pod、节点和集群的指标。此外,Prometheus利用AlertsManager,让您可以编程设置警报,以便在集群出现问题时及时警告您。Prometheus将指标数据存储为时间序列数据,由指标名称和键/值对标识。Prometheus包括一种称为PromQL(Prometheus查询语言)的查询语言。

Prometheus指标收集的高级架构如下所示:

![Prometheus指标收集](./images/prom.png)

Prometheus使用拉取机制,通过导出器从目标拉取指标,并通过[kube state metrics](https://github.com/kubernetes/kube-state-metrics)从Kubernetes API拉取指标。这意味着应用程序和服务必须公开包含Prometheus格式指标的HTTP(S)端点。Prometheus将根据其配置定期从这些HTTP(S)端点拉取指标。

导出器可让您将第三方指标作为Prometheus格式的指标使用。通常在每个节点上部署Prometheus导出器。有关完整的导出器列表,请参考Prometheus[导出器](https://prometheus.io/docs/instrumenting/exporters/)。虽然[节点导出器](https://github.com/prometheus/node_exporter)适用于导出Linux节点的硬件和操作系统指标,但不适用于Windows节点。

在**混合节点EKS集群中使用Windows节点**时,如果使用稳定的[Prometheus Helm图表](https://github.com/prometheus-community/helm-charts),您将在Windows节点上看到失败的Pod,因为该导出器并非针对Windows设计的。您需要将Windows工作池单独处理,并在Windows工作节点组上安装[Windows导出器](https://github.com/prometheus-community/windows_exporter)。

为了设置Windows节点的Prometheus监控,您需要在Windows服务器本身下载并安装WMI导出器,然后在Prometheus配置文件的采集配置中设置目标。
[发布页面](https://github.com/prometheus-community/windows_exporter/releases)提供了所有可用的.msi安装程序,包括相应的功能集和错误修复。安装程序将windows_exporter设置为Windows服务,并在Windows防火墙中创建一个例外。如果不带任何参数运行安装程序,导出器将使用默认设置运行,包括启用的采集器、端口等。
您可以查看本指南中的**调度最佳实践**部分,其中建议使用污点/容忍度或 RuntimeClass 选择性地将节点导出器部署到 Linux 节点,而 Windows 导出器则安装在 Windows 节点上,作为您引导节点或使用您选择的配置管理工具(例如 Chef、Ansible、SSM 等)的一部分。

请注意,与安装为 daemonset 的 Linux 节点上的节点导出器不同,在 Windows 节点上,WMI 导出器是安装在主机本身上的。该导出器将导出 CPU 使用率、内存和磁盘 I/O 使用率等指标,也可用于监控 IIS 站点和应用程序、网络接口和服务。

默认情况下,`windows_exporter` 将公开所有已启用收集器的指标。这是收集指标的推荐方式,以避免错误。但是,对于高级用途,`windows_exporter` 可以传递一个可选的收集器列表来过滤指标。Prometheus 配置中的 `collect[]` 参数可以实现这一点。

Windows 的默认安装步骤包括在引导过程中下载并以服务的形式启动导出器,并传递诸如要过滤的收集器之类的参数。
```powershell 
> Powershell Invoke-WebRequest https://github.com/prometheus-community/windows_exporter/releases/download/v0.13.0/windows_exporter-0.13.0-amd64.msi -OutFile <DOWNLOADPATH> 

> msiexec /i <DOWNLOADPATH> ENABLED_COLLECTORS="cpu,cs,logical_disk,net,os,system,container,memory"
```

默认情况下,可以在端口9182的/metrics端点上抓取指标。
此时,Prometheus可以通过在Prometheus配置中添加以下scrape_config来使用这些指标。
```yaml 
scrape_configs:
    - job_name: "prometheus"
      static_configs: 
        - targets: ['localhost:9090']
    ...
    - job_name: "wmi_exporter"
      scrape_interval: 10s
      static_configs: 
        - targets: ['<windows-node1-ip>:9182', '<windows-node2-ip>:9182', ...]
```

Prometheus 配置使用以下方式重新加载:
```bash 

> ps aux | grep prometheus
> kill HUP <PID> 

```

更好和推荐的添加目标的方法是使用一个称为 ServiceMonitor 的自定义资源定义,它是 [Prometheus 操作员](https://github.com/prometheus-operator/kube-prometheus/releases)的一部分,提供了 ServiceMonitor 对象的定义和一个控制器,用于激活我们定义的 ServiceMonitors 并自动构建所需的 Prometheus 配置。

ServiceMonitor 以声明方式指定应如何监控 Kubernetes 服务组,用于定义我们希望从 Kubernetes 中抓取指标的应用程序。在 ServiceMonitor 中,我们指定操作员可用于识别我们希望监控的 Kubernetes 服务(进而识别 Pod)的 Kubernetes 标签。

为了利用 ServiceMonitor,请创建一个指向特定 Windows 目标的 Endpoint 对象、一个无头服务和一个用于 Windows 节点的 ServiceMonitor。
```yaml
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    k8s-app: wmiexporter
  name: wmiexporter
  namespace: kube-system
subsets:
- addresses:
  - ip: NODE-ONE-IP
    targetRef:
      kind: Node
      name: NODE-ONE-NAME
  - ip: NODE-TWO-IP
    targetRef:
      kind: Node
      name: NODE-TWO-NAME
  - ip: NODE-THREE-IP
    targetRef:
      kind: Node
      name: NODE-THREE-NAME
  ports:
  - name: http-metrics
    port: 9182
    protocol: TCP

---
apiVersion: v1
kind: Service ##Headless Service
metadata:
  labels:
    k8s-app: wmiexporter
  name: wmiexporter
  namespace: kube-system
spec:
  clusterIP: None
  ports:
  - name: http-metrics
    port: 9182
    protocol: TCP
    targetPort: 9182
  sessionAffinity: None
  type: ClusterIP
  
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor ##Custom ServiceMonitor Object
metadata:
  labels:
    k8s-app: wmiexporter
  name: wmiexporter
  namespace: monitoring
spec:
  endpoints:
  - interval: 30s
    port: http-metrics
  jobLabel: k8s-app
  namespaceSelector:
    matchNames:
    - kube-system
  selector:
    matchLabels:
      k8s-app: wmiexporter
```

有关 ServiceMonitor 操作符和使用的更多详细信息,请查看官方[operator](https://github.com/prometheus-operator/kube-prometheus)文档。请注意,Prometheus 确实支持使用许多[服务发现](https://prometheus.io/blog/2015/06/01/advanced-service-discovery/)选项进行动态目标发现。
