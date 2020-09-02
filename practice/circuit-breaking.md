---
authors: ["ikingye"]
reviewers: ["wangfakang", "rootsongjc"]
---

# 熔断

熔断（Circuit Breaker），原是指当电流超过规定值时断开电路，进行短路保护或严重过载保护的机制。后来熔断也广泛应用于金融领域，指当股指波幅达到规定的熔断点时，交易所为控制风险采取的暂停交易措施。而在软件系统领域，熔断则是指当服务到达系统负载阈值时，为避免整个软件系统不可用，而采取的一种主动保护措施。

对于微服务系统而言，熔断尤为重要，它可以使系统在遭遇某些模块故障时，通过服务降级等方式来提高系统核心功能的可用性，得以应对来自故障、潜在峰值或其他未知网络因素的影响。

Istio 当然也具备了基本的熔断功能。

## Istio 熔断实践

### 开始之前

请确认已经按照本书正确安装了 Istio。

### 部署后端服务

我们使用 httpbin 样例程序，作为本次实践的后端服务。

如果你启用了 sidecar 自动注入，通过以下命令部署 httpbin 服务：

```shell
$ kubectl apply -f samples/httpbin/httpbin.yaml
```

否则，你必须在部署 httpbin 应用程序前进行手动注入，部署命令如下：

```shell
$ kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml)
```

### 配置熔断器

创建一个目标规则，定义 `maxConnections: 1` 和 `http1MaxPendingRequests: 1`，当并发的连接和请求数超过 1 个，熔断功能将会生效。

```shell
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 1
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
EOF
```

### 部署客户端程序

我们使用 [Fortio](https://github.com/istio/fortio) 作为客户端进行测试。Fortio 是一款优秀的负载测试工具，它起初是 Istio 项目的一部分，现已独立进行运营。

首先，我们创建 fortio 实例：

```shell
$ kubectl apply -f samples/httpbin/sample-client/fortio-deploy.yaml
```

获取 fortio pod 名：

```shell
$ FORTIO_POD=$(kubectl get pod | grep fortio | awk '{ print $1 }')
$ echo $FORTIO_POD
```

通过 fortio 请求一次 httpbin 服务：

```shell
$ kubectl exec -it $FORTIO_POD  -c fortio -- /usr/bin/fortio load -curl  http://httpbin:8000/get
```

结果类似下面这样，则说明 fortio 成功请求后端 httpbin 服务。

```
14:37:50 I fortio_main.go:168> Not using dynamic flag watching (use -config to set watch directory)
HTTP/1.1 200 OK
server: envoy
date: Mon, 17 Aug 2020 14:37:50 GMT
content-type: application/json
content-length: 621
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 28

{
  "args": {},
  "headers": {
    "Content-Length": "0",
    "Host": "httpbin:8000",
    "User-Agent": "fortio.org/fortio-1.6.7",
    "X-B3-Parentspanid": "7ef821ce5d7a5e0f",
    "X-B3-Sampled": "1",
    "X-B3-Spanid": "93ae07afe59db6ef",
    "X-B3-Traceid": "1c795f935f47f9b07ef821ce5d7a5e0f",
    "X-Envoy-Attempt-Count": "1",
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/default/sa/httpbin;Hash=65d5e53abc993564ebeab39a8c7347f752de219dc10dc5fd011e735d9b797b22;Subject=\"\";URI=spiffe://cluster.local/ns/default/sa/default"
  },
  "origin": "127.0.0.1",
  "url": "http://httpbin:8000/get"
}
```

### 验证熔断功能

发送并发数为 30 的连接（-c 30），请求 300 次（-n 300）：

```shell
$ kubectl exec -it $FORTIO_POD  -c fortio -- /usr/bin/fortio load -c 30 -qps 0 -n 300 -loglevel Warning http://httpbin:8000/get
```

结果类似下面这样，大概为 3% 的成功率：

```
root@yq01-sys-ote01.yq01.baidu.com:istio-1.6.5 kubectl exec -it $FORTIO_POD  -c fortio -- /usr/bin/fortio load -c 30 -qps 0 -n 300 -loglevel Warning http://httpbin:8000/get
14:50:39 I logger.go:114> Log level is now 3 Warning (was 2 Info)
Fortio 1.6.7 running at 0 queries per second, 56->56 procs, for 300 calls: http://httpbin:8000/get
Starting at max qps with 30 thread(s) [gomax 56] for exactly 300 calls (10 per thread + 0)
14:50:39 W http_client.go:697> Parsed non ok code 503 (HTTP/1.1 503)
（省略大量相同内容）
14:50:39 W http_client.go:697> Parsed non ok code 503 (HTTP/1.1 503)
Ended after 85.303225ms : 300 calls. qps=3516.9
Aggregated Function Time : count 300 avg 0.0071474321 +/- 0.003854 min 0.00035499 max 0.031570732 sum 2.14422963
# range, mid point, percentile, count
>= 0.00035499 <= 0.001 , 0.000677495 , 3.67, 11
> 0.001 <= 0.002 , 0.0015 , 7.67, 12
> 0.002 <= 0.003 , 0.0025 , 11.67, 12
> 0.003 <= 0.004 , 0.0035 , 17.00, 16
> 0.004 <= 0.005 , 0.0045 , 27.33, 31
> 0.005 <= 0.006 , 0.0055 , 43.00, 47
> 0.006 <= 0.007 , 0.0065 , 50.67, 23
> 0.007 <= 0.008 , 0.0075 , 61.33, 32
> 0.008 <= 0.009 , 0.0085 , 72.67, 34
> 0.009 <= 0.01 , 0.0095 , 79.00, 19
> 0.01 <= 0.011 , 0.0105 , 89.33, 31
> 0.011 <= 0.012 , 0.0115 , 92.00, 8
> 0.012 <= 0.014 , 0.013 , 96.67, 14
> 0.014 <= 0.016 , 0.015 , 98.00, 4
> 0.016 <= 0.018 , 0.017 , 98.67, 2
> 0.018 <= 0.02 , 0.019 , 99.00, 1
> 0.02 <= 0.025 , 0.0225 , 99.67, 2
> 0.03 <= 0.0315707 , 0.0307854 , 100.00, 1
# target 50% 0.00691304
# target 75% 0.00936842
# target 90% 0.01125
# target 99% 0.02
# target 99.9% 0.0310995
Sockets used: 292 (for perfect keepalive, would be 30)
Jitter: false
Code 200 : 10 (3.3 %)
Code 503 : 290 (96.7 %)
Response Header Sizes : count 300 avg 7.6866667 +/- 41.39 min 0 max 231 sum 2306
Response Body/Total Sizes : count 300 avg 261.35333 +/- 109.6 min 241 max 852 sum 78406
All done 300 calls (plus 0 warmup) 7.147 ms avg, 3516.9 qps
```

### 解释熔断行为

在 DestinationRule 配置中，我们定义了 `maxConnections: 1` 和 `http1MaxPendingRequests: 1`。
这些规则意味着，如果并发的连接和请求数超过 1 个，在 `istio-proxy` 进行进一步的请求和连接将被阻止。
于是，当我们并发数为 30 时，成功率只有 1/30，也就是 3.3% 左右。

请注意：如果你看到的成功率并非 3.3%，比如是 4.3%，也是正常的。`istio-proxy` 确实允许存在一定的误差。

### 清理实践环境

清理规则:

```shell
$ kubectl delete destinationrule httpbin
```

下线 httpbin 服务和客户端：

```shell
$ kubectl delete deploy httpbin fortio-deploy
$ kubectl delete svc httpbin
```

## Istio 熔断实现

Istio 是通过设置 Envoy 的相关阈值，来实现系统的熔断功能。具体来说，Istio 是通过创建 CRD DestinationRule，设置 `connectionPool` 的各项阈值，分为 TCP 和 HTTP 两种：

- TCP
  - MaxConnections
  - ConnectTimeout
  - TcpKeepalive
- HTTP
  - Http1MaxPendingRequests
  - Http2MaxRequests
  - MaxRequestsPerConnection
  - MaxRetries
  - IdleTimeout
  - H2UpgradePolicy

Istio DestinationRule 与 Envoy 的熔断参数对照表如下所示：

| Envoy paramether            | Envoy upon object        | Istio parameter          | Istio upon ojbect |
| --------------------------- | ------------------------ | ------------------------ | ----------------- |
| max_connections             | cluster.circuit_breakers | maxConnections           | TCPSettings       |
| max_pending_requests        | cluster.circuit_breakers | http1MaxPendingRequests  | HTTPSettings      |
| max_requests                | cluster.circuit_breakers | http2MaxRequests         | HTTPSettings      |
| max_retries                 | cluster.circuit_breakers | maxRetries               | HTTPSettings      |
| connect_timeout_ms          | cluster                  | connectTimeout           | TCPSettings       |
| max_requests_per_connection | cluster                  | maxRequestsPerConnection | HTTPSettings      |

Istio 熔断[源码参考](https://github.com/istio/istio/blob/f508fdd78eb0d3444e2bc2b3f36966d904c5db52/pilot/pkg/networking/core/v1alpha3/cluster.go#L736)：

```go
if settings.Http != nil {
  if settings.Http.Http2MaxRequests > 0 {
      // Envoy 只能控制 HTTP/2 后端的 MaxRequests
      threshold.MaxRequests = &wrappers.UInt32Value{Value: uint32(settings.Http.Http2MaxRequests)}
  }
  if settings.Http.Http1MaxPendingRequests > 0 {
      // Envoy 只能控制 HTTP/1.1 后端的 MaxPendingRequests
      threshold.MaxPendingRequests = &wrappers.UInt32Value{Value: uint32(settings.Http.Http1MaxPendingRequests)}
  }


  if settings.Http.MaxRequestsPerConnection > 0 {
      cluster.MaxRequestsPerConnection = &wrappers.UInt32Value{Value: uint32(settings.Http.MaxRequestsPerConnection)}
  }

  if settings.Http.MaxRetries > 0 {
      threshold.MaxRetries = &wrappers.UInt32Value{Value: uint32(settings.Http.MaxRetries)}
  }

  idleTimeout = settings.Http.IdleTimeout
}

if settings.Tcp != nil {
  if settings.Tcp.ConnectTimeout != nil {
      cluster.ConnectTimeout = gogo.DurationToProtoDuration(settings.Tcp.ConnectTimeout)
  }

  if settings.Tcp.MaxConnections > 0 {
      threshold.MaxConnections = &wrappers.UInt32Value{Value: uint32(settings.Tcp.MaxConnections)}
  }

  applyTCPKeepalive(push, cluster, settings)
}
```

## 参考

- [熔断与异常检测在 Istio 中的应用](https://www.servicemesher.com/blog/circuit-breaking-and-outlier-detection-in-istio/)
