!!! 注意
    本页面的内容是基于英文版本使用 Claude 3 生成的。如有差异,以英文版本为准。

# 日志记录

容器化应用程序通常将应用程序日志定向到 STDOUT。容器运行时捕获这些日志并对其进行处理 - 通常写入文件。这些文件存储的位置取决于容器运行时和配置。

与 Windows pod 的一个基本区别是它们不会生成 STDOUT。您可以运行 [LogMonitor](https://github.com/microsoft/windows-container-tools/tree/master/LogMonitor) 来检索来自正在运行的 Windows 容器的 ETW (Windows 事件跟踪)、Windows 事件日志和其他特定于应用程序的日志,并将格式化的日志输出管道到 STDOUT。然后可以使用 fluent-bit 或 fluentd 将这些日志流式传输到您所需的目标,例如 Amazon CloudWatch。

日志收集机制从 Kubernetes pod 检索 STDOUT/STDERR 日志。[DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) 是收集容器日志的常见方式。它使您能够独立于应用程序管理日志路由/过滤/丰富。可以使用 fluentd DaemonSet 将这些日志以及任何其他应用程序生成的日志流式传输到所需的日志聚合器。

有关从 Windows 工作负载向 CloudWatch 流式传输日志的更详细信息,请参阅[此处](https://aws.amazon.com/blogs/containers/streaming-logs-from-amazon-eks-windows-pods-to-amazon-cloudwatch-logs-using-fluentd/)。

## 日志记录建议

在 Kubernetes 中运行 Windows 工作负载时,一般日志记录最佳实践并没有什么不同。

* 始终记录**结构化日志条目**(JSON/SYSLOG),这使处理日志条目更加容易,因为对于这种结构化格式有许多预先编写的解析器。
* **集中**日志 - 可以使用专门的日志记录容器来收集和转发来自所有容器的日志消息到目标位置。
* 保持**日志详细程度**较低,除非进行调试。详细程度会给日志基础设施带来很大压力,重要事件可能会被噪音淹没。
* 始终记录**应用程序信息**以及**事务/请求 ID**以实现可追溯性。Kubernetes 对象不携带应用程序名称,因此例如 pod 名称 `windows-twryrqyw` 在调试日志时可能没有任何意义。这有助于使用您汇总的日志进行应用程序的可追溯性和故障排除。

    如何生成这些事务/关联 ID 取决于编程构造。但一个非常常见的模式是使用日志记录方面/拦截器,它可以使用 [MDC](https://logging.apache.org/log4j/1.2/apidocs/org/apache/log4j/MDC.html)(映射诊断上下文)为每个传入请求注入一个唯一的事务/关联 ID,如下所示:
```java   
import org.slf4j.MDC;
import java.util.UUID;
Class LoggingAspect { //interceptor

    @Before(value = "execution(* *.*(..))")
    func before(...) {
        transactionId = generateTransactionId();
        MDC.put(CORRELATION_ID, transactionId);
    }

    func generateTransactionId() {
        return UUID.randomUUID().toString();
    }
}
```

