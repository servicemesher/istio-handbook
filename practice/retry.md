---
authors: ["sunny0826"]
reviewers: [""]
---

# 重试

在网络环境不稳定的情况下，会出现暂时的网络不可达现象，这时需要重试机制，通过多次尝试来获取正确的返回信息。重试逻辑可以写业务代码中，比如 Bookinfo 应用中的`productpage`服务就存在硬编码重试，而 Istio 可以通过简单的配置来实现重试功能，让开发人员无需顾忌重试部分的代码实现，专心实现业务代码。

## 原理

本实践使用 [httpbin](https://github.com/istio/istio/tree/release-1.5/samples/httpbin) 示例，通过访问在集群内访问 `httpbin:8000/status/500` 地址来模拟返回状态码为 500 的现象，通过在 Istio 设置自动重试 3 次，学习重试的设置方式。

## 实践

**环境准备**

本节默认读者已按照**安装与部署**章节中的说明安装 Istio 的 [demo 配置](https://istio.io/zh/docs/setup/additional-setup/config-profiles/)。

**部署 httpbin 应用**

如果您启用了 sidecar 自动注入，通过以下命令部署`httpbin`服务：

```bash
$ kubectl apply -f samples/httpbin/httpbin.yaml
```

如果没有设置自动注入，则必须在部署`httpbin`应用时前进行手动注入：

```bash
$ kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml)
```

**访问 httpbin**

由于`httpbin`服务没有暴露在集群外部，这里借助`dockerqa/curl:ubuntu-trusty`镜像，使用`curl`命令在 Kubernetes 集群内访问`httpbin`服务：

```bash
$ kubectl run -i --rm --restart=Never dummy --image=dockerqa/curl:ubuntu-trusty --command -- curl --silent --head httpbin:8000/status/500
```

查看`httpbin`服务的 Proxy 日志，这里同一时刻只能看到一条访问请求日志：

```bash
$ kubectl logs httpbin-779c54bf49-25sfc -c istio-proxy | grep "HEAD /status/500"
...
[2020-04-28T02:26:32.129Z] "HEAD /status/500 HTTP/1.1" 500 - "-" "-" 0 0 4 3 "-" "curl/7.35.0" "be17043d-04b5-93cd-bf66-a03bdeee37f4" "httpbin:8000" "127.0.0.1:80" inbound|8000|http|httpbin.default.svc.cluster.local 127.0.0.1:46922 10.42.0.39:80 10.42.0.40:36416 outbound_.8000_._.httpbin.default.svc.cluster.local default
```

**设置重试**

为`httpbin`服务设置重试机制，这里设置如果服务在 1 秒内没有返回正确的返回值，就进行重试，重试的条件为返回码为`5xx`，重试 3 次：

```bash
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin-retries
spec:
  hosts:
  - httpbin
  http:
  - route:
    - destination:
        host: httpbin
    retries:
      attempts: 3
      perTryTimeout: 1s
      retryOn: 5xx
EOF
```

再次访问`httpbin`服务：

```bash
$ kubectl run -i --rm --restart=Never dummy --image=dockerqa/curl:ubuntu-trusty --command -- curl --silent --head httpbin:8000/status/500
```

查看`httpbin`服务的 Proxy 日志，可以看到同一时刻有 4 调请求日志，其中第一条是正常请求的访问日志，后三条为重试请求的访问日志：

```bash
$ kubectl logs httpbin-779c54bf49-25sfc -c istio-proxy | grep "HEAD /status/500"
....
[2020-04-28T02:29:53.659Z] "HEAD /status/500 HTTP/1.1" 500 - "-" "-" 0 0 4 3 "-" "curl/7.35.0" "be17043d-04b5-93cd-bf66-a03bdeee37f4" "httpbin:8000" "127.0.0.1:80" inbound|8000|http|httpbin.default.svc.cluster.local 127.0.0.1:46922 10.42.0.39:80 10.42.0.40:36416 outbound_.8000_._.httpbin.default.svc.cluster.local default
[2020-04-28T02:29:53.704Z] "HEAD /status/500 HTTP/1.1" 500 - "-" "-" 0 0 1 0 "-" "curl/7.35.0" "be17043d-04b5-93cd-bf66-a03bdeee37f4" "httpbin:8000" "127.0.0.1:80" inbound|8000|http|httpbin.default.svc.cluster.local 127.0.0.1:46922 10.42.0.39:80 10.42.0.40:36416 outbound_.8000_._.httpbin.default.svc.cluster.local default
[2020-04-28T02:29:53.780Z] "HEAD /status/500 HTTP/1.1" 500 - "-" "-" 0 0 2 2 "-" "curl/7.35.0" "be17043d-04b5-93cd-bf66-a03bdeee37f4" "httpbin:8000" "127.0.0.1:80" inbound|8000|http|httpbin.default.svc.cluster.local 127.0.0.1:46922 10.42.0.39:80 10.42.0.40:36416 outbound_.8000_._.httpbin.default.svc.cluster.local default
[2020-04-28T02:29:53.864Z] "HEAD /status/500 HTTP/1.1" 500 - "-" "-" 0 0 2 2 "-" "curl/7.35.0" "be17043d-04b5-93cd-bf66-a03bdeee37f4" "httpbin:8000" "127.0.0.1:80" inbound|8000|http|httpbin.default.svc.cluster.local 127.0.0.1:46922 10.42.0.39:80 10.42.0.40:36416 outbound_.8000_._.httpbin.default.svc.cluster.local default
```

访问`httpbin`服务的其他地址,如 `http://httpbin:8000/status/400`：

```bash
$ kubectl run -i --rm --restart=Never dummy --image=dockerqa/curl:ubuntu-trusty --command -- curl --silent --head httpbin:8000/status/400
```

并没有出现重试的情况，证明重试设置是生效的，重试仅对状态码为 5xx 的请求生效。

**清理**

清理`httpbin`服务及配置：

```bash
# 清理 httpbin 服务
$ kubectl delete -f samples/httpbin/httpbin.yaml
# 清理 ingress
kubectl delete virtualservice httpbin-retries
```

## 小结

通过本节的实践，我们学习了如何在 Istio 中配置重试，同时也对重试机制有一个完整的了解。

## 参考

- [Httpbin service - github.com](https://github.com/istio/istio/tree/release-1.5/samples/httpbin)