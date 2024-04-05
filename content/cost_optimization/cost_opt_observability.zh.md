
成本优化 - 可观察性

## 简介

可观察性工具可帮助您有效地检测、补救和调查您的工作负载。随着您使用 EKS 的增加,遥测数据的成本自然会增加。有时,平衡您的运营需求和衡量对您的业务至关重要的事项,同时保持可观察性成本在可控范围内,可能会很有挑战性。本指南重点介绍了日志、指标和跟踪这三个可观察性支柱的成本优化策略。这些最佳实践中的每一个都可以独立应用,以符合您组织的优化目标。

## 日志记录

日志记录在监控和排查集群中应用程序方面发挥着关键作用。有几种策略可用于优化日志记录成本。下面列出的最佳实践策略包括检查您的日志保留策略以实施细粒度控制,根据重要性将日志数据发送到不同的存储选项,以及利用日志过滤来缩小存储的日志消息类型。有效管理日志遥测可为您的环境带来成本节省。

## EKS 控制平面

### 优化您的控制平面日志

Kubernetes 控制平面是管理集群的[一组组件](https://kubernetes.io/docs/concepts/overview/components/#control-plane-components),这些组件将不同类型的信息作为日志流发送到 [Amazon CloudWatch](https://aws.amazon.com/cloudwatch/) 中的日志组。虽然启用所有控制平面日志类型确实有好处,但您应该了解每个日志的信息以及存储所有日志遥测的相关成本。您需要为发送到 Amazon CloudWatch Logs 的日志支付[标准 CloudWatch Logs 数据摄取和存储成本](https://aws.amazon.com/cloudwatch/pricing/)。在启用它们之前,请评估每个日志流是否必要。

例如,在非生产集群中,仅选择性地启用特定的日志类型(如 api 服务器日志),并在分析后停用。但对于生产集群,您可能无法重现事件,解决问题需要更多日志信息,因此可以启用所有日志类型。有关更多控制平面成本优化实施细节,请参阅[此博客文章](https://aws.amazon.com/blogs/containers/understanding-and-cost-optimizing-amazon-eks-control-plane-logs/)。

#### 将日志流式传输到 S3

另一个成本优化的最佳实践是通过 CloudWatch Logs 订阅将控制平面日志流式传输到 S3。利用 CloudWatch Logs [订阅](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Subscriptions.html)允许您选择性地将日志转发到 S3,这提供了更加经济高效的长期存储,相比于无限期地保留在 CloudWatch 中。例如,对于生产集群,您可以创建一个关键日志组,并利用订阅在 15 天后将这些日志流式传输到 S3。这将确保您可以快速访问日志进行分析,同时也通过将日志移动到更加经济高效的存储来节省成本。

!!! attention
    截至 2023 年 9 月 5 日,EKS 日志在 Amazon CloudWatch Logs 中被归类为供应商日志。供应商日志是由 AWS 服务代表客户本地发布的特定 AWS 服务日志,可以享受批量折扣价格。请访问 [Amazon CloudWatch 定价页面](https://aws.amazon.com/cloudwatch/pricing/)了解更多关于供应商日志定价的信息。

## EKS 数据平面

### 日志保留

Amazon CloudWatch 的默认保留策略是无限期保留日志,永不过期,这会产生与您所在 AWS 区域相关的存储成本。为了降低存储成本,您可以根据工作负载需求自定义每个日志组的保留策略。

在开发环境中,长时间的保留期可能并不必要。但在生产环境中,您可以设置更长的保留策略,以满足故障排查、合规性和容量规划的需求。例如,如果您在节假日季节运行电子商务应用程序,系统会承受更大的负载,可能会出现一些不太明显的问题,这时您会希望设置更长的日志保留期,以便进行详细的故障排查和事后分析。

您可以在 AWS CloudWatch 控制台或 [AWS API](https://docs.aws.amazon.com/cli/latest/reference/logs/put-retention-policy.html) 中[配置您的保留期](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Working-with-log-groups-and-streams.html#SettingLogRetention),时长从 1 天到 10 年不等,具体取决于每个日志组的需求。灵活的保留期可以节省日志存储成本,同时也能保留关键日志。

### 日志存储选项

存储是观察成本的一大驱动因素,因此优化日志存储策略至关重要。您的策略应该与工作负载需求保持一致,同时保持性能和可扩展性。减少存储日志成本的一种策略是利用 AWS S3 存储桶及其不同的存储层。

#### 直接将日志转发到 S3
考虑将较不重要的日志,如开发环境,直接转发到S3,而不是Cloudwatch。这可以立即影响日志存储成本。一种选择是使用Fluentbit直接将日志转发到S3。您可以在`[OUTPUT]`部分定义这一点,这是FluentBit传输容器日志以进行保留的目标位置。请查看[此处](https://docs.fluentbit.io/manual/pipeline/outputs/s3#worker-support)的其他配置参数。
```
[OUTPUT]
        Name eks_to_s3
        Match application.* 
        bucket $S3_BUCKET name
        region us-east-2
        store_dir /var/log/fluentbit
        total_file_size 30M
        upload_timeout 3m
```


#### 仅将日志转发到 CloudWatch 进行短期分析

对于更关键的日志,例如生产环境中您可能需要立即分析数据的情况,请考虑将日志转发到 CloudWatch。您可以在 `[OUTPUT]` 部分定义此内容,这是 FluentBit 用于保留容器日志的目标位置。请查看其他配置参数 [here](https://docs.fluentbit.io/manual/pipeline/outputs/cloudwatch)。
```
[OUTPUT]
        Name eks_to_cloudwatch_logs
        Match application.*
        region us-east-2
        log_group_name fluent-bit-cloudwatch
        log_stream_prefix from-fluent-bit-
        auto_create_group On
```

然而,这不会立即影响您的成本节省。为了获得更多的节省,您将不得不将这些日志导出到 Amazon S3。

#### 从 CloudWatch 导出到 Amazon S3

为了长期存储 Amazon CloudWatch 日志,我们建议将您的 Amazon EKS CloudWatch 日志导出到 Amazon Simple Storage Service (Amazon S3)。您可以通过在[控制台](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/S3ExportTasksConsole.html)或 API 上创建导出任务来将日志转发到 Amazon S3 存储桶。这样做后,Amazon S3 提供了许多选项来进一步降低成本。您可以定义自己的 [Amazon S3 生命周期规则](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)来将您的日志移动到符合您需求的存储类,或利用 [Amazon S3 Intelligent-Tiering](https://aws.amazon.com/s3/storage-classes/intelligent-tiering/) 存储类让 AWS 根据您的使用模式自动将数据移动到长期存储。有关更多详细信息,请参阅此[博客](https://aws.amazon.com/blogs/containers/understanding-and-cost-optimizing-amazon-eks-control-plane-logs/)。例如,对于您的生产环境日志,它们在 CloudWatch 中驻留超过 30 天,然后导出到 Amazon S3 存储桶。如果您需要在稍后时间查看日志,您可以使用 Amazon Athena 查询 Amazon S3 存储桶中的数据。

### 减少日志级别

为您的应用程序实践选择性日志记录。您的应用程序和节点默认输出日志。对于您的应用程序日志,请调整日志级别以与工作负载和环境的重要性相匹配。例如,下面的 Java 应用程序输出 `INFO` 日志,这是典型的默认应用程序配置,并且根据代码可能会导致大量的日志数据。
```java hl_lines="7"
import org.apache.log4j.*;

public class LogClass {
   private static org.apache.log4j.Logger log = Logger.getLogger(LogClass.class);
   
   public static void main(String[] args) {
      log.setLevel(Level.INFO);

      log.debug("This is a DEBUG message, check this out!");
      log.info("This is an INFO message, nothing to see here!");
      log.warn("This is a WARN message, investigate this!");
      log.error("This is an ERROR message, check this out!");
      log.fatal("This is a FATAL message, investigate this!");
   }
}
```

在开发环境中,将日志级别更改为`DEBUG`,这可以帮助您调试问题或在问题进入生产环境之前发现潜在问题。
```java
      log.setLevel(Level.DEBUG);
```

在生产环境中,请考虑将日志级别修改为 `ERROR` 或 `FATAL`。这将仅在您的应用程序出现错误时输出日志,减少日志输出,并帮助您关注应用程序状态的重要数据。
```java
      log.setLevel(Level.ERROR);
```

您可以微调各种 Kubernetes 组件的日志级别。例如，如果您使用 [Bottlerocket](https://bottlerocket.dev/) 作为 EKS 节点操作系统，则有配置设置可以调整 kubelet 进程的日志级别。下面是此配置设置的一个片段。请注意默认的[日志级别](https://github.com/bottlerocket-os/bottlerocket/blob/3f716bd68728f7fd825eb45621ada0972d0badbb/README.md?plain=1#L528)为 **2**，它调整了 `kubelet` 进程的日志详细程度。
```toml hl_lines="2"
[settings.kubernetes]
log-level = "2"
image-gc-high-threshold-percent = "85"
image-gc-low-threshold-percent = "80"
```

对于开发环境,您可以将日志级别设置为大于 **2** 以查看更多事件,这有助于调试。对于生产环境,您可以将级别设置为 **0** 以仅查看关键事件。

### 利用过滤器

当使用默认的 EKS Fluentbit 配置将容器日志发送到 Cloudwatch 时,FluentBit 会捕获并发送 **所有** 应用程序容器日志,并使用 Kubernetes 元数据对其进行丰富,如下所示的 `[INPUT]` 配置块所示。
```
    [INPUT]
        Name                tail
        Tag                 application.*
        Exclude_Path        /var/log/containers/cloudwatch-agent*, /var/log/containers/fluent-bit*, /var/log/containers/aws-node*, /var/log/containers/kube-proxy*
        Path                /var/log/containers/*.log
        Docker_Mode         On
        Docker_Mode_Flush   5
        Docker_Mode_Parser  container_firstline
        Parser              docker
        DB                  /var/fluent-bit/state/flb_container.db
        Mem_Buf_Limit       50MB
        Skip_Long_Lines     On
        Refresh_Interval    10
        Rotate_Wait         30
        storage.type        filesystem
        Read_from_Head      ${READ_FROM_HEAD}
```

上面的 `[INPUT]` 部分正在摄取所有容器日志。这可能会产生大量可能不必要的数据。过滤掉这些数据可以减少发送到 CloudWatch 的日志数据量,从而降低成本。您可以在日志输出到 CloudWatch 之前应用过滤器。Fluentbit 在 `[FILTER]` 部分定义了这一点。例如,过滤掉附加到日志事件的 Kubernetes 元数据可以减少日志量。
```
    [FILTER]
        Name                nest
        Match               application.*
        Operation           lift
        Nested_under        kubernetes
        Add_prefix          Kube.

    [FILTER]
        Name                modify
        Match               application.*
        Remove              Kube.<Metadata_1>
        Remove              Kube.<Metadata_2>
        Remove              Kube.<Metadata_3>
    
    [FILTER]
        Name                nest
        Match               application.*
        Operation           nest
        Wildcard            Kube.*
        Nested_under        kubernetes
        Remove_prefix       Kube.
```


## 指标

[指标](https://aws-observability.github.io/observability-best-practices/signals/metrics/)提供了关于系统性能的宝贵信息。通过将所有与系统相关或可用资源的指标集中在一个中心位置,您可以比较和分析性能数据。这种集中式方法使您能够做出更明智的战略决策,例如扩大或缩小资源。此外,指标在评估资源健康状况方面发挥关键作用,使您能够在必要时采取主动措施。通常,可观察性成本随遥测数据收集和保留而增加。以下是一些您可以实施的策略,以降低指标遥测的成本:仅收集重要的指标,降低遥测数据的基数,并微调遥测数据收集的粒度。

### 监控重要的内容并仅收集所需的内容

第一个降低成本的策略是减少您正在收集的指标数量,从而降低保留成本。

1. 首先从您和/或您的利益相关方的需求出发,确定[最重要的指标](https://aws-observability.github.io/observability-best-practices/guides/#monitor-what-matters)。成功指标因人而异!知道什么是*好的*,并据此进行测量。
2. 考虑深入了解您正在支持的工作负载,并确定其关键绩效指标(KPI),即"黄金信号"。这些应该与业务和利益相关方的要求保持一致。使用Amazon CloudWatch和指标数学计算SLI、SLO和SLA对于管理服务可靠性至关重要。遵循[指南](https://aws-observability.github.io/observability-best-practices/guides/operational/business/key-performance-indicators/#10-understanding-kpis-golden-signals)中概述的最佳实践,有效地监控和维护您的EKS环境的性能。
3. 然后继续通过不同的基础设施层,将EKS集群、节点和其他基础设施指标[连接和关联](https://aws-observability.github.io/observability-best-practices/signals/metrics/#correlate-with-operational-metric-data)到您的工作负载KPI。将您的业务指标和运营指标存储在一个可以相互关联并根据观察到的影响做出结论的系统中。
4. EKS公开了来自控制平面、集群kube-state-metrics、pod和节点的指标。所有这些指标的相关性都取决于您的需求,但您可能不需要跨不同层面的每个单一指标。您可以使用这个[EKS关键指标](https://aws-observability.github.io/observability-best-practices/guides/containers/oss/eks/best-practices-metrics-collection/)指南作为监控EKS集群和您的工作负载整体健康状况的基准。
这里是一个示例 Prometheus 抓取配置,我们使用 `relabel_config` 仅保留 kubelet 指标,并使用 `metric_relabel_config` 删除所有容器指标。
```yaml
  kubernetes_sd_configs:
  - role: endpoints
    namespaces:
      names:
      - kube-system
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  tls_config:
    insecure_skip_verify: true
  relabel_configs:
  - source_labels: [__meta_kubernetes_service_label_k8s_app]
    regex: kubelet
    action: keep

  metric_relabel_configs:
  - source_labels: [__name__]
    regex: container_(network_tcp_usage_total|network_udp_usage_total|tasks_state|cpu_load_average_10s)
    action: drop
```

[简化中文]

### 在适用的情况下减少基数

基数是指与其维度(例如 Prometheus 标签)相结合的数据值的唯一性,对于特定的指标集而言。高基数指标具有许多维度,每个维度指标组合都具有更高的唯一性。较高的基数会导致更大的指标遥测数据大小和存储需求,从而增加成本。

在下面的高基数示例中,我们可以看到指标 Latency 具有维度 RequestID、CustomerID 和 Service,每个维度都有许多唯一值。基数是每个维度可能值的组合数量的度量。在 Prometheus 中,每组唯一的维度/标签都被视为一个新的指标,因此高基数意味着更多指标。

![high cardinality](../images/high-cardinality.png)

在具有许多指标和每个指标的维度/标签(集群、命名空间、服务、Pod、容器等)的 EKS 环境中,基数倾向于增长。为了优化成本,请仔细考虑您收集的指标的基数。例如,如果您正在为可视化聚合特定指标,则可以删除较低层级的标签,如命名空间标签。

为了识别 Prometheus 中的高基数指标,您可以运行以下 PROMQL 查询来确定哪些抓取目标具有最高的指标数(基数):
```promql 
topk_max(5, max_over_time(scrape_samples_scraped[1h]))
```

以下 PROMQL 查询可帮助您确定哪些抓取目标具有最高的指标流失率(给定抓取中创建了多少新的指标序列):
```promql
topk_max(5, max_over_time(scrape_series_added[1h]))
```

如果您正在使用 Grafana，您可以使用 Grafana Lab 的 Mimirtool 来分析您的 Grafana 仪表板和 Prometheus 规则，以识别未使用的高基数指标。请按照[此指南](https://grafana.com/docs/grafana-cloud/account-management/billing-and-usage/control-prometheus-metrics-usage/usage-analysis-mimirtool/?pg=blog&plcmt=body-txt#analyze-and-reduce-metrics-usage-with-grafana-mimirtool)了解如何使用 `mimirtool analyze` 和 `mimirtool analyze prometheus` 命令来识别您的仪表板中未引用的活动指标。

### 考虑指标粒度

以每秒与每分钟的更高粒度收集指标可能会对收集和存储的遥测数据量产生重大影响，从而增加成本。确定合理的抓取或指标收集间隔,在足够的粒度以查看瞬时问题和足够低的成本效益之间取得平衡。对于用于容量规划和较长时间窗口分析的指标,请降低粒度。

下面是来自默认 AWS Distro for Opentelemetry (ADOT) EKS Addon Collector [配置](https://docs.aws.amazon.com/eks/latest/userguide/deploy-deployment.html)的一个片段。

!!! attention
    全局 Prometheus 抓取间隔设置为 15 秒。此抓取间隔可以增加,从而减少 Prometheus 收集的指标数据量。
```yaml hl_lines="22"
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: my-collector-amp

...

  config: |
    extensions:
      sigv4auth:
        region: "<YOUR_AWS_REGION>"
        service: "aps"

    receivers:
      #
      # Scrape configuration for the Prometheus Receiver
      # This is the same configuration used when Prometheus is installed using the community Helm chart
      # 
      prometheus:
        config:
          global:
  scrape_interval: 15s
            scrape_timeout: 10s
```

跟踪

跟踪的主要成本来自于跟踪存储生成。通过跟踪,目标是收集足够的数据来诊断和理解性能方面的问题。然而,由于X-Ray跟踪的成本基于转发到X-Ray的数据,在转发后擦除跟踪不会降低您的成本。让我们回顾一下如何在保持数据以进行适当分析的同时降低跟踪成本的方法。

应用采样规则

X-Ray的采样率默认是保守的。定义采样规则,您可以控制收集的数据量。这将提高性能效率,同时降低成本。通过[降低采样率](https://docs.aws.amazon.com/xray/latest/devguide/xray-console-sampling.html#xray-console-custom),您可以仅从请求中收集您的工作负载所需的跟踪,同时保持较低的成本结构。

例如,您有一个Java应用程序,您想调试一个有问题的路由的所有请求的跟踪。

**通过SDK配置从JSON文档加载采样规则**
```json
{
"version": 2,
  "rules": [
    {
"description": "debug-eks",
      "host": "*",
      "http_method": "PUT",
      "url_path": "/history/*",
      "fixed_target": 0,
      "rate": 1,
      "service_type": "debug-eks"
    }
  ],
  "default": {
"fixed_target": 1,
    "rate": 0.1
  }
}
```

**通过控制台**

![console](../images/console.png)

### 使用 AWS 分布式 OpenTelemetry (ADOT) 应用尾部采样

ADOT 尾部采样允许您控制摄入服务的跟踪量。但是,尾部采样允许您在请求的所有跨度完成后而不是在开始时定义采样策略。这进一步限制了传输到 CloudWatch 的原始数据量,从而降低了成本。

例如,如果您对登录页面的流量采样 1%,对支付页面的请求采样 10%,这可能会在 30 分钟内留下 300 个跟踪。使用 ADOT 尾部采样规则过滤特定错误,您可能只剩下 200 个跟踪,从而减少了存储的跟踪数量。
```yaml hl_lines="5"
processors:
  groupbytrace:
    wait_duration: 10s
    num_traces: 300 
    tail_sampling:
    decision_wait: 1s # This value should be smaller than wait_duration
    policies:
      - ..... # Applicable policies**
  batch/tracesampling:
    timeout: 0s # No need to wait more since this will happen in previous processors
    send_batch_max_size: 8196 # This will still allow us to limit the size of the batches sent to subsequent exporters

service:
  pipelines:
    traces/tailsampling:
      receivers: [otlp]
      processors: [groupbytrace, tail_sampling, batch/tracesampling]
      exporters: [awsxray]
```


### 利用 Amazon S3 存储选项

您应该利用 AWS S3 存储桶及其不同的存储类来存储跟踪数据。在保留期到期之前,将跟踪数据导出到 S3。使用 Amazon S3 生命周期规则将跟踪数据移动到满足您需求的存储类。

例如,如果您有 90 天前的跟踪数据,[Amazon S3 Intelligent-Tiering](https://aws.amazon.com/s3/storage-classes/intelligent-tiering/) 可以根据您的使用模式自动将数据移动到长期存储。如果您需要在稍后参考跟踪数据,可以使用 [Amazon Athena](https://aws.amazon.com/athena/) 查询 Amazon S3 中的数据。这可以进一步降低您的分布式跟踪成本。

## 其他资源:

* [可观察性最佳实践指南](https://aws-observability.github.io/observability-best-practices/guides/)
* [最佳实践指标收集](https://aws-observability.github.io/observability-best-practices/guides/containers/oss/eks/)
* [AWS re:Invent 2022 - 亚马逊的可观察性最佳实践 (COP343)](https://www.youtube.com/watch?v=zZPzXEBW4P8)
* [AWS re:Invent 2022 - 可观察性:现代应用程序的最佳实践 (COP344)](https://www.youtube.com/watch?v=YiegAlC_yyc)
