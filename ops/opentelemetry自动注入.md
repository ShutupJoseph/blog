### 安装opentelemetry
```
helm install opentelemetry-operator open-telemetry/opentelemetry-operator --set "manager.collectorImage.repository=otel/opentelemetry-collector-k8s" --set admissionWebhooks.certManager.enabled=false --set admissionWebhooks.autoGenerateCert.enabled=true --namespace kube-otel --create-namespace --set manager.extraArgs={"--enable-go-instrumentation=true"}
```
开启golang的自动注入，默认是关闭的
```
 --set manager.extraArgs={"--enable-go-instrumentation=true"}
```

### 创建opentelemetry 所需资源
Instrumentation 可以输出到 Collector再输出到 signoz，Instrumentation也可以直接输出到 signoz，注意协议端口是否匹配。
#### 创建Collector
```
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otlp
  namespace: kube-otel
spec:
  mode: deployment
  config: |
    receivers:
      otlp:
        protocols:
          http:
          grpc:
    processors:
      memory_limiter:
        check_interval: 1s
        limit_percentage: 75
        spike_limit_percentage: 15
      batch:
        send_batch_size: 10000
        timeout: 10s
    exporters:
      otlp:
        endpoint: http://10.49.1.5:4317
        tls:
          insecure: true
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [otlp]
        metrics:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [otlp]
```
#### 创建Instrumentation
```
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: otlp-instrumentation
  namespace: kube-otel
spec:
  exporter:
    endpoint: http://signoz-otel-collector.kube-otel:4318
  propagators:
    - tracecontext
    - baggage
    - b3
  sampler:
    type: always_on
  go:
    resourceRequirements:
      limits:
        cpu: 100m
        memory: 128Mi
      requests:
        cpu: 50m #
        memory: 62Mi
  java:
    env:
      - name: OTEL_EXPORTER_OTLP_ENDPOINT
        value: http://signoz-otel-collector.kube-otel:4317
```

### 开启自动注入
在 pod 的 功能注释，也就是annotate 中，启用
```
instrumentation.opentelemetry.io/inject-go: kube-otel/otlp-instrumentation
instrumentation.opentelemetry.io/otel-go-auto-target-exe: golang可执行文件路径
```
java 仅需一行
```
instrumentation.opentelemetry.io/inject-java: kube-otel/otlp-instrumentation
```

### 安装signoz
infra 可以自动采集k8s metrics
```
helm repo add signoz https://charts.signoz.io
helm upgrade --install signoz signoz/signoz --namespace kube-otel
helm upgrade --install k8s-infra signoz/k8s-infra --set otelCollectorEndpoint=signoz-otel-collector.kube-otel:4317 --namespace kube-otel
```

#### java jmx 采集文档：
https://github.com/open-telemetry/opentelemetry-java-instrumentation/blob/main/instrumentation/jmx-metrics/javaagent/README.md
#### signoz 图形模板：
https://github.com/SigNoz/dashboards
#### 详情参考：
https://github.com/ShutupJoseph/blog/blob/main/ops/kubernetes/45.OpenTelemetry/45.4OpenTelemetryOperator.md

#### 效果：
截至：2024年10月中旬
golang很多东西都采集不到，不推荐，java的数据很全面，但是我个人更喜欢skywalking
