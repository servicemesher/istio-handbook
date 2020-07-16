---
authors: ["mcsos"]
reviewers: ["rootsongjc","GuangmingLuo"]
---

# 授权策略

## 概述

授权功能是 Istio 中安全体系的一个重要组成部分，它用来实现访问控制的功能，即判断一个请求是否允许通过，这个请求可以是从外部进入 Istio 内部的请求，也可以是在 Istio 内部从服务 A 到服务 B 的请求。可以把授权功能近似地认为是一种四层到七层的“防火墙”，它会像传统防火墙一样，对数据流进行分析和匹配，然后执行相应的动作。

本节所有概念和操作都基于 Istio 1.6 版本。

授权功能是通过授权策略 (AuthorizationPolicy) 来进行配置和使用的，下面是一个完整的授权策略示例：

``` yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
 name: httpbin-policy
 namespace: foo
spec:
  selector:
    matchLabels:
      app: httpbin
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/sleep"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/info*"]
    when:
    - key: request.auth.claims[iss]
      values: ["https://foo.com"]
```

这个授权策略的含义是：筛选出 `foo` 这个 namespace 中含有 `app: httpbin` label 的 pod，对发送到这些 pod 的请求进行匹配，如果匹配成功，则放行当前请求，匹配规则如下：发起请求的 pod 的 Service Account 需要是 `cluster.local/ns/default/sa/sleep` ，请求使用 HTTP 协议，请求的具体方法类型是 `GET` ，请求的URL为 `/info*` ，并且请求中需要包含由 `https://foo.com` 签发的有效的 JWT Token。

从这个例子中可以看出一个授权策略主要包含以下几个部分：

- **name**。授权策略的名称，仅用于标识授权策略本身，不会影响规则的匹配和执行。
- **namespace**。当前授权策略对象所在的 namespace ，可以使用这个字段配置不同作用范围的授权策略，详见 [授权策略的作用范围](#授权策略的作用范围)。
- **selector**。使用 label 来选择当前授权策略作用于哪些 pod 上。注意，这里设置的是服务端的 pod ，因为最终这些规则会转换成 Envoy 规则由服务端的 Envoy Proxy 来具体执行。例如有 client 和 server 两个 service ，它们的 pod 对应的 label 分别为 `app: client` 和 `app: server` ，为了针对从 client 到 server 的请求进行配置授权策略，这里的 selector 应该设置为 `app: server`。
- **action**。可以为 `ALLOW` (默认值)或者 `DENY`。
- **rules**。匹配规则，如果匹配成功，就会执行对应的 action ，详见 [授权策略的规则详解](#授权策略的规则详解)。

## 授权策略的作用范围

授权策略可以按照作用域的大小分成三个不同的类型：全局策略、某个 namespace 内的局部策略和具有明确 match label 的授权策略。下面分别进行说明。

### 全局策略

授权策略位于 istio 的 root namespace 中(例如 `istio-system` )，且匹配所有的 pod。这种规则会作用于整个集群中的所有 pod。

下面的例子中有3个全局策略，第一个是全局 `ALLOW` ，第二个和第三个是全局 `DENY` ，后面这两个作用类似，但又有重要的区别，详见 [授权策略的匹配算法](#授权策略的匹配算法)。

``` bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: global-allow
  namespace: istio-system
spec:
  action: ALLOW
  rules:
  - {}
EOF

kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: global-deny
  namespace: istio-system
spec:
  action: DENY
  rules:
  - {}
EOF

kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: global-deny
  namespace: istio-system
spec:
  {}
EOF
```

### 某个 namespace 内的局部策略

授权策略位于除了 root namespace 之外的任何一个 namespace 中，且匹配所有的 pod ，这种情况下，这个策略会作用于当前 namespace 中的所有 pod。

下面的例子中是3个 namespace 级别的策略，第一个是 `ALLOW` ，第二个和第三个是 `DENY` ，像全局策略一样，后面这两个作用类似，但又有重要的区别，详见 [授权策略的匹配算法](#授权策略的匹配算法)。

``` bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: foo-namespace-allow
  namespace: foo
spec:
  action: ALLOW
  rules:
  - {}
EOF

kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: foo-namespace-deny
  namespace: foo
spec:
  action: DENY
  rules:
  - {}
EOF

kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: foo-namespace-deny
  namespace: foo
spec:
  {}
EOF
```

### 具有明确 match label 的授权策略

这种授权策略仅作用于当前 namespace 下使用 `selector` 字段匹配到的 pod。

``` bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: httpbin-allow
  namespace: foo
spec:
  selector:
    matchLabels:
      app: httpbin
  action: ALLOW
  rules:
  - {}
EOF
```

## 授权策略的匹配算法

针对某一个请求，会按照一定的匹配算法来执行相应的授权策略：

1. 如果有任何一条 `DENY` 授权策略匹配当前请求，则拒绝当前请求。
2. 针对当前 pod，如果没有任何 `ALLOW` 授权策略，则放行当前请求。
3. 如果有任何一条 `ALLOW` 授权策略匹配当前请求，则放行当前请求。
4. 拒绝当前请求。

也就意味着，如果同时有 `ALLOW` 和 `DENY` 策略作用于同一个 pod 上，则 `DENY` 策略会优先执行，其它的 `ALLOW` 规则就会被忽略。

注意这个顺序非常重要，有时又会比较隐晦，因此在配置比较复杂策略的时候需要多加小心。

在上文 [授权策略的作用范围](#授权策略的作用范围) 中提到授权策略在配置时，有一些细节上的差异，现结合授权策略的匹配算法进行一些分析。

``` yaml
spec:
  {}
```

这是一个 `DENY` 策略，作用于全局策略或者 namespace 级别(取决于策略所在 namespace 是否为 root namespace )。但是它并没有对当前请求进行匹配，也就意味着按照授权策略的匹配算法在匹配的时候并不会优先匹配到这条规则，因此可以将其作为一个“后备”策略，即全局或者 namespaces 级别的一个默认策略。

``` yaml
spec:
  action: DENY
  rules:
  - {}
```

这条规则会真正地匹配当前的请求，又由于它是 `DENY` 规则，按照授权策略的匹配算法，它会首先得到执行，也就意味着如果配置了一条这种全局或者 namespace 级别的规则，那么所有的其它 `ALLOW` 规则都不会得到执行。因此这条规则在实际中并没有什么价值。

``` yaml
spec:
  action: ALLOW
  rules:
  - {}
```

这条规则和上一条规则类似，但是它是 `ALLOW` 规则，因此按照授权策略的匹配算法，它的优先级会低一些，因此也可以像第一条规则一样作为一个全局或者 namespace 级别的默认策略。

## 授权策略的规则详解

授权策略中最重要的是其中的 `rule` 字段，它指定了如何针对当前的请求进行匹配。如果一个授权策略中指定了多条 `rule` 规则，则它们之间是`或`的关系，即只要其中任意一条规则匹配成功，那么整个授权策略匹配成功，就会执行相应的 action ，下面是 [概述](#概述) 中提到的一个授权策略的例子：

``` yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
 name: httpbin-policy
 namespace: foo
spec:
  selector:
    matchLabels:
      app: httpbin
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/sleep"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/info*"]
    when:
    - key: request.auth.claims[iss]
      values: ["https://foo.com"]
```

这里的 `rules` 是一个 `rule` 的列表。每一条 `rule` 规则包括三部分： from 、 to 和 when 。类似于防火墙规则， from 和 to 匹配当前请求从哪里来、到哪里去， when 会增加一些额外的检测，当这些条件都满足时，就会认为当前规则匹配成功。如果其中某一部分未进行配置，则认为其可以匹配成功。

在 `rule` 中进行配置时，所有的字符串类型都支持类似于通配符的匹配模式，例如 `abc*` 匹配 "abc" 和 "abcd" 等， `*xyz` 匹配 "xyz" 和 "axyz" 等，单独的 `*` 匹配非空的字符串。

下面针对具体的字段详细进行说明。

- **from**。针对请求的发送方进行匹配，主要包括 principals 、 requestPrincipals 、 namespaces 和 ipBlocks 四个部分。
  - **principals**。匹配发送方的身份，在 Kubernetes 中可以认为是 pod 的 Service Account。使用这个字段时，首先需要开启 mTLS 功能，关于这部分内容可参见 [对等认证](peer-authentication.md) 。例如，当前请求是从 default namespace 中的 pod 中发出，且 pod 使用的 Service Account 名为 `sleep` ，针对这个请求进行匹配，可将 principals 配置为[`cluster.local/ns/default/sa/sleep`]。
  - **requestPrincipals**。匹配请求中的 JWT Token 的 `<issuer>/<subject>` 字段组合。
  - **namespaces**。匹配发送方 pod 所在的 namespace。
  - **ipBlocks**。匹配请求的源 IP 地址段。
- **to**。针对请求的接收方进行匹配。除了请求接收方，还会对请求本身进行匹配。包括以下字段：
  - **hosts**。目的 host。
  - **ports**。目的 port。
  - **methods**。是指当前请求执行的 HTTP Method 。针对 gRPC 服务，这个字段需要设置为 `POST`。注意这个字段必须在 HTTP 协议时才进行匹配，如果不是请求不是 HTTP 协议，则认为匹配失败。
  - **paths**。当前请求执行的 HTTP URL Path 。针对 gRPC 服务，需要配置为 `/package.service/method` 格式。
- **when**。这是一个 key/value 格式的 list 。这个字段会针对请求进行一些额外的检测，当这些检测全部匹配时才会认证当前规则匹配成功。例如 `key: request.headers[User-Agent]` 可以匹配 HTTP Header 中的 `User-Agent` 字段。所有可配置项可参见 [Authorization Policy Conditions](https://istio.io/latest/docs/reference/config/security/conditions/)。

针对以上字段，还有对应的反向匹配操作，即“取反”匹配，包括 notPrincipals 、 notNamespaces 等。例如 `notNamespaces: ["bar"]` 表示当发送请求的 pod 不位于 "bar" 这个 namespace 中的时候匹配成功。

另外，在 `rule` 中会有非常多针对 JWT Token 进行匹配的字段，关于这部分可以查看 [JWT 授权](jwt-authorization.md) 里的详细分析。

下面针对上文列出来的授权策略给出一些实际的例子，一方面可以在实际环境中是如何使用这些策略的，另一方面也可以验证前文所述的各种匹配字段、授权策略的匹配算法和授权策略的作用域。

## 操作示例

### 创建应用

首先，创建客户端和服务器端的 service 和对应的 pod，使用的例子位于 istio 源代码中 [samples](https://github.com/istio/istio/tree/master/samples)：

``` bash
$ kubectl create ns foo
$ kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml) -n foo
$ kubectl apply -f <(istioctl kube-inject -f samples/sleep/sleep.yaml) -n foo
```

确认 pod 已正常运行：

``` bash
$ kubectl get pod -n foo --show-labels
NAME                       READY   STATUS    RESTARTS   AGE   LABELS
httpbin-5d5df46d48-jndgh   2/2     Running   0          57s   app=httpbin,istio.io/rev=,pod-template-hash=5d5df46d48,security.istio.io/tlsMode=isti
o,version=v1
sleep-545684d78b-29x74     2/2     Running   0          56s   app=sleep,istio.io/rev=,pod-template-hash=545684d78b,security.istio.io/tlsMode=istio
$
```

确认可以正常访问，且启用了 mTLS 功能，关于 mTLS 功能的开启和验证请参见 [对等认证](peer-authentication.md)：

``` bash
$ kubectl exec $(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name}) -c sleep -n foo -- curl "http://httpbin.foo:8000/ip" -s -o /dev/null -w "sleep.foo to httpbin.foo: %{http_code}\n"
sleep.foo to httpbin.foo: 200
$
$ kubectl exec $(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name}) -c sleep -n foo -- curl http://httpbin.foo:8000/headers -s | grep X-Forwarded-Client-Cert
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/foo/sa/httpbin;Hash=e0f2132eb6ae920cec4b2ea16b9baa33ca388b719a2648636f7a75542852ff0e;Subject=\"\";URI=spiffe://cluster.local/ns/foo/sa/sleep"
```

### 全局策略测试

接下来创建一个全局默认的拒绝策略：

``` bash
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: global-deny
  namespace: istio-system
spec:
  {}
EOF
```

这时再进行验证连通性：

``` bash
$ kubectl exec $(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name}) -c sleep -n foo -- curl "http://httpbin.foo:8000/ip" -s -o /dev/null -w "sleep.foo to httpbin.foo: %{http_code}\n"
sleep.foo to httpbin.foo: 403
```

服务拒绝，表明全局拒绝策略生效。

接下来创建一个 httpbin pod 的 `ALLOW` 策略：

``` bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: httpbin-allow-policy
  namespace: foo
spec:
  selector:
    matchLabels:
      app: httpbin
  action: ALLOW
  rules:
  - to:
    - operation:
        methods: ["GET"]
EOF
```

按照授权策略的匹配算法，应该可以匹配到第3条规则，因此会执行 `ALLOW` 动作，运行下面的命令进行验证：

``` bash
$ kubectl exec $(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name}) -c sleep -n foo -- curl "http://httpbin.foo:8000/ip" -s -o /dev/null -w "sleep.foo to httpbin.foo: %{http_code}\n"
sleep.foo to httpbin.foo: 200
```

### 测试Rule中的字段

下面我们以 Service Account 为例来进行说明。首先来检查 sleep pod 所使用的的 Service Account：

``` bash
$ kubectl get pod -l app=sleep -n foo -o jsonpath={.items...serviceAccountName}
sleep
```

根据 namespace 和 Service Account 构造出 principals 字段，更新授权策略：

``` bash
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: httpbin-allow-policy
  namespace: foo
spec:
  selector:
    matchLabels:
      app: httpbin
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/foo/sa/sleep"]
    to:
    - operation:
        methods: ["GET"]
EOF
```

进行验证：

``` bash
$ kubectl exec $(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name}) -c sleep -n foo -- curl "http://httpbin.foo:8000/ip" -s -o /dev/null -w "sleep.foo to httpbin.foo: %{http_code}\n"
sleep.foo to httpbin.foo: 200
```

访问仍然是放行状态，说明刚才的授权策略是生效的。

将授权策略中的 Service Account 改为一个其它值：

``` bash
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: httpbin-allow-policy
  namespace: foo
spec:
  selector:
    matchLabels:
      app: httpbin
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/foo/sa/other-sa"]
    to:
    - operation:
        methods: ["GET"]
EOF
```

``` bash
$ kubectl exec $(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name}) -c sleep -n foo -- curl "http://httpbin.foo:8000/ip" -s -o /dev/null -w "sleep.foo to httpbin.foo: %{http_code}\n"
sleep.foo to httpbin.foo: 403
```

访问失败，因为授权策略中配置 Service Account 字段与实际的 Service Account 不匹配。

同样地，可以配置 From 、 To 和 When 中的其它字段进行测试。

### 授权策略的匹配算法测试

首先，删除之前创建的名为 httpbin-allow-policy 的授权策略，目前系统中仅存在一个全局的默认 `DENY` 策略。

``` bash
kubectl delete authorizationpolicies httpbin-allow-policy -n foo
```

接下来创建一个匹配 "GET" 方法的 `ALLOW` 策略，名为 httpbin-allow-get：

``` bash
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: httpbin-allow-get
  namespace: foo
spec:
  selector:
    matchLabels:
      app: httpbin
  action: ALLOW
  rules:
  - to:
    - operation:
        methods: ["GET"]
EOF
```

这时使用 "GET /ip" 请求进行测试，由于可以和 httpbin-allow-get 策略匹配，因此按照授权策略的匹配算法，可以匹配到第3条规则，因此可以正常访问：

``` bash
$ kubectl exec $(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name}) -c sleep -n foo -- curl "http://httpbin.foo:8000/ip" -s -o /dev/null -w "sleep.foo to httpbin.foo: %{http_code}\n"
sleep.foo to httpbin.foo: 200
```

使用 "POST /ip" 请求进行测试，与 httpbin-allow-get 策略不能匹配，因此会执默认的全局 `DENY` 策略：

``` bash
$ kubectl exec $(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name}) -c sleep -n foo -- curl -X POST "http://httpbin.foo:8000/ip" -s -o /dev/null -w "sleep.foo to httpbin.foo: %{http_code}\n"
sleep.foo to httpbin.foo: 403
```

再创建一个 "/ip" 的 `DENY` 策略，名为 httpbin-deny-ip-url：

``` bash
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: httpbin-deny-ip-url
  namespace: foo
spec:
  selector:
    matchLabels:
      app: httpbin
  action: DENY
  rules:
  - to:
    - operation:
        paths: ["/ip"]
EOF
```

这时使用 "GET /ip" 请求进行测试：

``` bash
$ kubectl exec $(kubectl get pod -l app=sleep -n foo -o jsonpath={.items..metadata.name}) -c sleep -n foo -- curl "http://httpbin.foo:8000/ip" -s -o /dev/null -w "sleep.foo to httpbin.foo: %{http_code}\n"
sleep.foo to httpbin.foo: 403
```

可以看出执行失败。失败的原因是 "GET /ip" 请求与我们刚才创建的 httpbin-allow-get 和 httpbin-deny-ip-url 两个授权策略都会匹配，但是授权策略的匹配算法执行到第1条规则时，会发现匹配 httpbin-deny-ip-url 授权策略，然后就会直接拒绝当前的请求。另一条授权策略 httpbin-allow-get 便无法得到执行。

### 清理

执行以下操作来清理我们创建过的各种资源：

``` bash
$ kubectl delete authorizationpolicies httpbin-deny-ip-url -n foo
$ kubectl delete authorizationpolicies httpbin-allow-get -n foo
$ kubectl delete authorizationpolicies httpbin-allow-policy -n foo
$ kubectl delete authorizationpolicies global-deny -n istio-system
$ kubectl delete -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml) -n foo
$ kubectl delete -f <(istioctl kube-inject -f samples/sleep/sleep.yaml) -n foo
$ kubectl delete ns foo
```

## 总结

本节我们详细分析了授权策略的概念和用法，也可以看出利用这些规则可以组合出非常复杂的场景，因此在使用复杂的授权策略时需要非常小心。另外，在授权策略中会涉及到很多针对请求中的 JWT Token 进行匹配的规则，相关的概念和用法会在下一节进行详细阐述。

## 参考

- [Authorization Policy](https://istio.io/latest/docs/reference/config/security/authorization-policy/)
- [Authorization Policy Conditions](https://istio.io/latest/docs/reference/config/security/conditions/)
- [Authorization](https://istio.io/latest/docs/tasks/security/authorization/)
