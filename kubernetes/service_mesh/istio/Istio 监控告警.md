# Istio 监控告警

## 告警

建议添加如下告警：

1. 控制面异常
    1. Pod 重启
    2. CPU / 内存使用率过高、磁盘不足等通用告警
2. 数据面异常
    1. ingress/egress 的常规告警
    2. HTTP/gRPC 可用率（即 HTTP 2xx/3xx、gRPC 0 的占比）告警
    3. 其他全局指标的异常突变告警，比如 QPS 突增突降、404/5xx 错误码突增等
    4. 对于 sidecar，还需要考虑是 source 侧的指标还是 destination 侧的指标。有些情况下 source 侧的指标异常，但是因为 destination 侧的 Pod 全挂了

## 监控指标

### 监控中出现 503 可能的原因

应用程序雪崩，线程池跑满，所有请求全部没响应。这会在 istio 中导致大量的 0 状态码和 503 状态码。

通过 istio 监控指标，能看到 503 对应的 [RESPONSE_FLAG](https://www.envoyproxy.io/docs/envoy/v1.21.1/configuration/observability/access_log/usage#config-access-log-format-response-flags)，它给出了 503 更详细的原因。

通过 prometheus 监控能够查到所有 503 响应的 RESPONSE_FLAG 都是 URX，0 状态码则是客户端等太久不耐烦了，主动断开连接。

要进一步验证的话，在容器里直接 curl 容器的任何 http path 也是会卡住的（即使是 404），因为压力太大，所有线程都卡住了，cpu/mem 等指标也会无法采集。

这种情况应该从应用程序或者流量切量上找原因，为啥会雪崩，如何去做熔断限流、或者缓慢切量，以避免雪崩或者恢复服务。


### 自定义监控指标

#### 1. 为 HTTP/gRPC 指标添加 host/method/path 信息

HTTP/gRPC 的指标都可以通过 `istio_requests_total` 查到，但是有一些重要的相关信息，Istio 默认不会放到指标里面：

- request.method
- request.host
- request.url_path - 去掉 query_string 后的 path
  - 添加这个 path 可能会导致指标数据量激增！注意谨慎操作。
  - 官方推荐是通过 AttributeGen 根据 url_path 对请求进行分类，但是要硬编码很多内容，感觉很不方便。

那么它默认包含的信息里，哪些是有价值的呢？先看下 HTTP 协议：

- request_protocol="http"
- response_code - HTTP 状态码
- response_flags - 出现异常时，此参数会提供更多异常信息

对于 gRPC 协议呢：

- request_protocol="grpc"
- response_code - HTTP 状态码对 gRPC 而言没有任何意义，一般是写死的 200
- response_flags - 出现异常时，此参数会提供更多异常信息
- grpc_response_status - gRPC 自定义的状态码

>需要注意到，gRPC 的请求跟响应可能不是一一对应的，这个时候通过 `istio_requests_total` 得到的信息会如何处理，我还不太清楚。可能需要用到专用指标 `istio_request_messages_total` 跟 `istio_response_messages_total`？


可通过如下配置全局添加上述三个指标标签：

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  meshConfig:
    # `defaultConfig`: sidecar 和 gateway 的默认配置(envoy)，Pod 上的 `proxy.istio.io/config` 注解可以覆盖此默认值
    # https://istio.io/latest/docs/reference/config/istio.mesh.v1alpha1/#ProxyConfig
    defaultConfig:
      extraStatTags:  # 如果要在 metrics 中使用非默认的 stat 标签，就需要先补充到这
      - request_host
      - request_method
      - request_url_path
```

也可以在每个服务上单独配置：

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template: # pod template
    metadata:
      annotations:
        sidecar.istio.io/extraStatTags: request_host,request_method,request_url_path
```

然后配置 prometheus 插件，如下配置只在 `foo` 名字空间内生效：

```yaml
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: namespaced-metrics
  namespace: foo
spec:
  # no selector specified, applies to all workloads in the namespace
  metrics:
  - providers:
    - name: prometheus
    overrides:
    - match:
        # metric 的更多选项参见 https://istio.io/latest/docs/reference/config/telemetry/#MetricSelector-IstioMetric
        metric: ALL_METRICS
        # 三选一 CLIENT_AND_SERVER / CLIENT / SERVER
        mode: CLIENT_AND_SERVER
      tagOverrides:
        # 在所有指标上添加两个新标签
        request_method:
          value: "request.method"
        request_host:
          value: "request.host"
    # 暂时只在 istio_requests_total 上添加 request_url_path 标签
    - match:
        metric: REQUEST_COUNT  # 对应 p8s 指标 istio_requests_total
      tagOverrides:
        request_url_path:
          value: "request.url_path"
```


## 参考

- [Customizing Istio Metrics](https://istio.io/latest/docs/tasks/observability/metrics/customize-metrics/)
- [Classifying Metrics Based on Request or Response](https://istio.io/latest/docs/tasks/observability/metrics/classify-metrics/)
