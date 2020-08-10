---
authors: ["mcsos"]
reviewers: ["wangfakang"]
---

# 授权

为了保障集群中服务的安全，Istio 提供了一系列开箱即用的安全机制，对服务进行访问控制就是其中非常重要的一个部分，这个功能被称为 **授权**。授权功能会按照预定义的配置针对特定的请求进行匹配，匹配成功之后会执行对应的动作，例如放行请求或者拒绝请求。

由于作用的对象是服务，因此，授权功能主要适用于四至七层（相比较而言，传统的防火墙主要用于二至四层），例如 gRPC 、 HTTP 、 HTTPS 和 HTTP2 以及 TCP 协议等，对基于这些协议的请求进行授权检测，Istio 都可以提供原生支持。

从数据流的角度来讲，授权功能可以用于多种场景，包括从集群外部访问集群内部的服务、从集群内部的一个服务访问集群内部的另一个服务、以及从集群内部访问集群外部的服务。

就像实现 [流量控制](traffic-control.md) 功能一样，Istio 中授权功能的实现也是非侵入式的，可以在不影响现有业务逻辑的情况下，通过一系列自定义的 [授权策略](authorization-policy.md) 在 Istio 集群中启用授权功能，实现业务的安全加固。

用户可以通过配置 **授权策略** 这种 CRD 对象来实现授权功能。授权策略按照作用域大小，可以分为三类：

- 作用于整个集群的全局策略。
- 作用于某个 namespace 的局部策略。
- 作用于某些 pod 的具体策略。

下面是一个具体的例子：

``` yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-read
  namespace: default
spec:
  selector:
    matchLabels:
      app: products
  action: ALLOW
  rules:
  - to:
    - operation:
         methods: ["GET", "HEAD"]
```

这条授权策略会找出 default 这个 namespace 中含有 `app: products` label 的 pod，针对发送到这些 pod 的请求进行匹配，如果这些请求使用 HTTP 协议，且请求方法为 "GET" 或者 "HEAD"，则放行这些请求。

我们接下来会在 [授权策略](authorization-policy.md) 一节中对授权策略进行详细的分析和举例说明。另外，授权策略除了匹配常规的协议字段之外，还有非常多针对请求中的 JWT Token 进行匹配的选项，因此这部分内容会在单独的一节中进行说明，详见 [JWT 授权](jwt-authorization.md) 。

# 参考

- [Security](https://istio.io/latest/docs/concepts/security/)
