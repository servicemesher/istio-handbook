---
authors: ["hb-chen"]
reviewers: ["rootsongjc"]
---

# JWT 授权

首先了解一下 JWT（ JSON Web Token ），是一种多方传递可信 JSON 数据的方案，一个 JWT token 由`.`分隔的三部分组成：`{Header}.{Payload}.{Signature}`，其中 Header 是 Base64 编码的 JSON 数据，包含令牌类型`typ`、签名算法`alg`以及秘钥 ID `kid`等信息；Payload 是需要传递的 claims 数据，也是 Base64 编码的 JSON 数据，其中有些字段是 JWT 标准已有的字段如：`exp`、`iat`、`iss`、`sub`和`aud`等，也可以根据需求添加自定义字段；Signature 是对前两部分的签名，防止数据被篡改，以此确保 token 信息是可信的，更多参考[Introduction to JSON Web Tokens](https://jwt.io/introduction/)。Istio 中验签所需公钥由 RequestAuthentication 资源的 JWKS 配置提供，详见[章节3.4.1.3. 终端用户认证](/practice/end-users-authentication.html)。

前面介绍了`HTTP`、`TCP`、`gRPC`等不同协议的流量授权，而 JWT 授权则是对**最终用户**的访问控制，试想某个内部服务需要管理员才能够访问，这时候就需要验证**最终用户**的角色是否为管理员，可以在 JWT claims 中带有管理员角色信息，然后在授权策略中对该角色授权。不同协议的流量授权在**操作**`to`方面有比较多的示范，本节则主要在**来源**`from`和**自定义条件**`when`做示范。

本节使用 Istio 示例中的 httpbin 服务做演示，涉及不同场景下 JWT 授权的应用，主要包括：

- 无授权策略情况下的 JWT 认证
- 任意非空的 JWT 授权
- Principal 条件匹配授权
- Claims 条件匹配授权
- 分阶段认证和授权

## 准备工作

- Istio 环境
    - `istioctl manifest apply`默认安装
- 示例下载 Istio Release 1.5.0
    - httpbin 示例

### httpbin 服务部署

#### 创建命名空间

为了后续演示方便，设置命名空间环境变量`NS=authz-jwt`，然后创建命名空间

```bash
$ export NS=authz-jwt
$ kubectl create namespace $NS
```

#### Sidecar 自动注入

为命名空间开启 Sidecar 自动注入

```bash
$ kubectl label namespace $NS istio-injection=enabled
```

#### 部署 httpbin 服务

在下载的 Istio 目录下部署 httpbin 服务，并检查`pod`、`service`的创建情况

```bash
$ cd {istio release path}
kubectl -n $NS apply -f samples/httpbin/httpbin.yaml

$ kubectl -n $NS get po
NAME                       READY   STATUS    RESTARTS   AGE
httpbin-779c54bf49-df6mc   2/2     Running   0          6h9m

$ kubectl -n $NS get svc
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
httpbin   ClusterIP   10.111.151.12   <none>        8000/TCP   6h9m
```

### httpbin 网关

#### 添加 httpbin 服务的 gateway

为了避免与其他网关产生冲突，网关指定 HOST 为 `authz-jwt.local`，这样在测试时通过指定 HOST 确定是路由到 httpbin 服务。

```bash
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
  namespace: $NS
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "authz-jwt.local"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
  namespace: $NS
spec:
  hosts:
  - "*"
  gateways:
  - httpbin-gateway
  http:
  - route:
    - destination:
        host: httpbin
        port:
          number: 8000
EOF
```

#### 获取 Ingress 网关的 IP 和 PORT

> 根据环境不同获取 ingress IP 和 Port 参考[determining-the-ingress-ip-and-ports](https://istio.io/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-ip-and-ports)

```bash
# 演示使用的是 minikube 环境
$ export INGRESS_IP=$(minikube ip)
$ export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
```

#### 验证网关

请求正常，相应`200`

```bash
$ curl -I -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 200 OK
```

现在我们完成准备工作，部署了一个可以通过 ingress 网关访问的 httpbin 服务，接下来开始 JWT 授权相关的内容。

## 无授策略权情况下的 JWT 认证

要使用 JWT 授权的前提是有有效的 JWT 最终身份认证，所以在使用 JWT 授权前首先要为服务添加**最终身份认证**即 RequestAuthentication ，更多参考[章节3.4.1. 认证](/istio-handbook/practice/authentication.html)。

### 添加 RequestAuthentication

本节使用 Istio 代码库中提供的用于 JWT 演示的配置，包括 JWKS 端点配置，以及两个测试用的 token ： demo 和 groups-scope 。其中 demo 是一个普通 token ，claims 有 JWT 的基础属性；groups-scope 是一个带有自定义属性的 token，claims 除了基础属性还包括 group 和 scope，这两个 token 详细的 claims 结构在后续应用中会有介绍，相关连接：
- [JWKS](https://raw.githubusercontent.com/istio/istio/release-1.5/security/tools/jwt/samples/jwks.json)
- [demo token](https://raw.githubusercontent.com/istio/istio/release-1.5/security/tools/jwt/samples/demo.jwt)
- [groups-scope token](https://raw.githubusercontent.com/istio/istio/release-1.5/security/tools/jwt/samples/groups-scope.jwt)

```bash
$ kubectl apply -f - <<EOF
apiVersion: "security.istio.io/v1beta1"
kind: "RequestAuthentication"
metadata:
  name: "jwt-example"
  namespace: $NS
spec:
  selector:
    matchLabels:
      app: httpbin
  jwtRules:
  - issuer: "testing@secure.istio.io"
    jwksUri: "https://raw.githubusercontent.com/istio/istio/release-1.5/security/tools/jwt/samples/jwks.json"
EOF
```

### JWT 认证测试

默认 JWT 认证的 token 是以 `Bearer ` 为前缀放在 Authorization header 中，如：`Authorization: Bearer token`

使用以下3种方式测试服务，看请求的响应情况:
- 不带 Authorization header
- Authorization header 携带一个无效的 token
- Authorization header 携带一个有效的 token

1.不带 Authorization header 的请求正常，响应`200`

```bash
$ curl -I -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 200 OK
```

2.携带无效 token 的请求被绝，响应`401`

```bash
$ curl -I -H "Authorization: Bearer invalidToken" -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 401 Unauthorized
```

3.测试有效 token 前要先获取 demo token 并设置为环境变量，以便后续演示中使用

```bash
$ export TOKEN=$(curl https://raw.githubusercontent.com/istio/istio/release-1.5/security/tools/jwt/samples/demo.jwt -s)
```

4.携带有效 token 的请求正常，响应`200`

```bash
$ curl -I -H "Authorization: Bearer $TOKEN" -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 200 OK
```

添加 RequestAuthentication 后，并不是要求所有请求都要带有 JWT token，因为 RequestAuthentication 只负责验证 token 的有效性，token 的有无以及是否授权访问由 AuthorizationPolicy 的 JWT 授权策略决定。所以在只有 RequestAuthentication 时，可以同时支持无 token 请求和带有有效 token 的请求，而带有无效 token 的请求将被拒绝，此时 JWT 认证是一个**非必要条件**。

## 任意非空的 JWT 授权

在只有 RequestAuthentication 时不带 token 的请求是可以正常访问的，而需求可能会要求全部请求必须经过认证才能访问，这就需要使用 JWT 授权策略。

AuthorizationPolicy rule 规则中与 JWT 相关的字段包括：

field | sub field | JWT claims
------|------|------
from.source | requestPrincipals | iss/sub
from.source | notRequestPrincipals | iss/sub
when.key | request.auth.principal | iss/sub
when.key | request.auth.audiences | aud
when.key | request.auth.presenter | azp
when.key | request.auth.claims[key] | JWT 全部属性

其中`from.source`的`requestPrincipals` 、`notRequestPrincipals`和`when.key`的`request.auth.principal`都是对 Principal 条件的策略，Principal 由 JWT claims 的`iss`和`sub`用`/`拼接组成`{iss}/{sub}`，`request.auth.audiences`和`request.auth.presenter`分别对应 claims 的`aud`和`azp`属性，`request.auth.claims[key]`则可以通过`key`值获取 JWT claims 中的任意值作为条件。

这些字段的匹配都遵循授权的4种匹配规则:完全匹配、前缀匹配、后缀匹配和存在匹配，详见[章节2.3.2.1.2. 认证策略](/concepts/authentication-policy.html)，其中存在匹配（`*`）表示该字段可以匹配任意内容，但是不能为空，和不指定字段是不一样的，不指定是包括空在内的任意内容，所以使用存在匹配可以满足对**任意非空的 JWT 授权**的需求。

### 添加 AuthorizationPolicy

添加一个 `from.source` 为 `requestPrincipals: ["*"]`的 JWT 授权策略，允许任意非空 Principal 的请求

```bash
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
 name: require-jwt
 namespace: $NS
spec:
 selector:
   matchLabels:
     app: httpbin
 action: ALLOW
 rules:
 - from:
   - source:
       requestPrincipals: ["*"]
EOF
```

### JWT 授权测试

1.不带 Authorization header 的请求被拒绝，响应`403`

```bash
$ curl -I -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 403 Forbidden
```

2.带有有效 token 的请求访问正常，相应`200`

```bash
$ curl -I -H "Authorization: Bearer $TOKEN" -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 200 OK
```

在添加的 AuthorizationPolicy 中带有 JWT 相关条件字段后，不带 token 的请求将被拒绝，此时 JWT 认证变为了**必要条件**

## Principal 条件

前面已经介绍`from.source`和`when.key`中与 Principal 相关的三个字段，这里使用`source.requestPrincipals`做为示例，来看下 Principal 条件的应用。

要设置具体条件首先要看下 JWT 的 claims 信息，通过`echo $TOKEN | cut -d '.' -f2 - | base64 -d -`管道操作解码 token 的 Payload 部分，查看 JWT claims 结构如下:

```bash
$ echo $TOKEN | cut -d '.' -f2 - | base64 -d -
{
  "exp" : 4685989700,
  "foo" : "bar",
  "iss" : "testing@secure.istio.io",
  "sub" : "testing@secure.istio.io",
  "iat" : 1532389700
}
```

### 测试 principal 条件

1.根据 token claims 结构修改`source.requestPrincipals`条件为`testing@secure.istio.io/testing@secure.istio.io`

```bash
$ kubectl patch AuthorizationPolicy require-jwt -n $NS --type merge -p '
spec:
  rules:
  - from:
    - source:
        requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
'
```

或者使用等效的自定义条件`when`的`request.auth.principal`

```bash
$ kubectl patch AuthorizationPolicy require-jwt -n $NS --type merge -p '
spec:
  rules:
  - when:
    - key: request.auth.principal
      values: ["testing@secure.istio.io/testing@secure.istio.io"]
'
```

2.请求正常，响应`200`
```bash
$ curl -I -H "Authorization: Bearer $TOKEN" -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 200 OK
```

### 拒绝授权示范

1.这时我们将 `requestPrincipals`规则改为其它值，如`requestPrincipals: ["testing@secure.istio.io/none"]`

```bash
kubectl patch AuthorizationPolicy require-jwt -n $NS --type merge -p '
spec:
  rules:
  - from:
    - source:
        requestPrincipals: ["testing@secure.istio.io/none"]
'
```

2.请求被拒绝，响应`403`

```bash
$ curl -I -H "Authorization: Bearer $TOKEN" -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 403 Forbidden
```

3.恢复正常授权

```bash
kubectl patch AuthorizationPolicy require-jwt -n $NS --type merge -p '
spec:
  rules:
  - from:
    - source:
        requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
'
```

## Claims 条件

更多的有关 JWT 属性的规则可以通过自定义条件`when`补充，其中`request.auth.principal`与`source.requestPrincipals`一致已经演示，
`request.auth.audiences`和`request.auth.presenter`的使用不再赘述，接下来在 Principal 条件基础上增加 claims 条件，看下自定义条件`request.auth.claims[]`的应用。

### groups-scope token 测试 claims 条件

为了丰富 JWT claims 信息，增加另一个 JWT token： groups-scope

1.获取 groups-scope token，并解码 JWT claims，其中包括两个自定义的 claims ：`scope`和`groups`

```bash
$ export TOKEN_GROUP=$(curl https://raw.githubusercontent.com/istio/istio/release-1.5/security/tools/jwt/samples/groups-scope.jwt -s) && echo $TOKEN_GROUP | cut -d '.' -f2 - | base64 -d -
{
  "exp" : 3537391104,
  "scope" : [
    "scope1",
    "scope2"
  ],
  "iss" : "testing@secure.istio.io",
  "groups" : [
    "group1",
    "group2"
  ],
  "sub" : "testing@secure.istio.io",
  "iat" : 1537391104
}
```

2.结合 JWT claims 结构这里使用 groups 作为自定义条件，如：仅允许 group1

```bash
$ kubectl patch AuthorizationPolicy require-jwt -n $NS --type merge -p '
spec:
  rules:
  - from:
    - source:
        requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
    when:
    - key: request.auth.claims[groups]
      values: ["group1"]
'
```

3.测试 $TOKEN 请求被拒绝，响应`403`

```bash
curl -I -H "Authorization: Bearer $TOKEN" -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 403 Forbidden
```

4.测试 $TOKEN_GROUP 请求正常，响应`200`

```bash
$ curl -I -H "Authorization: Bearer $TOKEN_GROUP" -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 200 OK
```

### 拒绝授权示范

1.尝试一个不在 groups-scope token 内的 group 值，如：`values: ["group3"]`

```bash
$ kubectl patch AuthorizationPolicy require-jwt -n $NS --type merge -p '
spec:
  rules:
  - from:
    - source:
        requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
    when:
    - key: request.auth.claims[groups]
      values: ["group3"]
'
```

2.测试 $TOKEN_GROUP 请求被拒绝，响应`403`

```bash
curl -I -H "Authorization: Bearer $TOKEN_GROUP" -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 403 Forbidden
```

3.恢复正常授权

```bash
$ kubectl patch AuthorizationPolicy require-jwt -n $NS --type merge -p '
spec:
  rules:
  - from:
    - source:
        requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
    when:
    - key: request.auth.claims[groups]
      values: ["group1"]
'
```

## 分阶段认证和授权

现在每次请求对 JWT 的认证和授权都是在 httpbin 服务上，而对于真实场景的请求到达内部服务，往往要经过 n 个服务，如果恰巧这个验证是最后一个服务，当因为 token 无效或者没有 token 导致请求失败时，服务的响应时间大大延长，并造成资源的浪费，所以可以将 token 的验证前置到 Ingress 网关。
通过前面的实践可以知道添加 RequestAuthentication 仅对带有 Authorization header 请求做认证，不影响无 Authorization header 的请求。具体是否需要分阶段验证，以及在什么位置验证，需要根据业务场景考虑，一般越是顶层条件越靠前如：`from.source.requestPrincipals`、`to.operation.hosts`，而`when.request.auth.claims[group/scope]`和`to.operation.methods/paths`组合可以在相关服务做详细的访问控制。

另外需要注意的是如果调用链路有多次使用同一个 token，则必须在 RequestAuthentication 的`jwtRules`中开启`forwardOriginalToken: true`以将 Authorization header 向下传递，也可以通过 fromHeaders / fromParams 携带多个不同场景的 token，具体参考 [JWTRule](https://istio.io/docs/reference/config/security/jwt/#JWTRule) 。说到 token 的传递，Authorization header 也可以在服务与服务间调用时添加，所以**最终用户**的定义并不限定为客户端，任何一个发起调用的服务都是一个**最终用户**。 

### Ingress JWT 认证

1.测试当前无效 token 请求，响应 `401`，并且在响应的 header 中有上游主机处理请求消耗的时间`x-envoy-upstream-service-time: 1`，通过这个 header 的有无可以确定请求是否是被** httpbin 服务**拒绝，有是被** httpbin 服务**拒绝，没有则是被**网关**拒绝。

```bash
$ curl -I -H "Authorization: Bearer invalidToken" -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 401 Unauthorized
...
x-envoy-upstream-service-time: 1
```

2.为 Ingress 开启 JWT 认证

```bash
$ kubectl apply -f - <<EOF
apiVersion: "security.istio.io/v1beta1"
kind: "RequestAuthentication"
metadata:
  name: "jwt-gateway"
  namespace: istio-system
spec:
  selector:
    matchLabels:
        app: istio-ingressgateway
  jwtRules:
  - issuer: "testing@secure.istio.io"
    jwksUri: "https://raw.githubusercontent.com/istio/istio/release-1.5/security/tools/jwt/samples/jwks.json"
    forwardOriginalToken: true
EOF
```

3.再测试无效 token 请求，同样响应 `401`，但没有了`x-envoy-upstream-service-time` header，说明请求是在**网关**被拒绝

```bash
$ curl -I -H "Authorization: Bearer invalidToken" -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 401 Unauthorized
```

### Ingress JWT 授权

接下来看下在 Ingress 和 httpbin 服务使用不同策略时的响应情况。

1.前面只是 Ingress 的 JWT 认证，结合网关的入口特点可以添加根据 HOST 的不同限定`scope`的授权，如：访问`host=authz-jwt.local`要求`scpoe=scope1`

```bash
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: istio-system
spec:
  selector:
    matchLabels:
      app: istio-ingressgateway
  action: ALLOW
  rules:
  - when:
    - key: request.auth.claims[scope]
      values: ["scope1"]
    to:
    - operation:
        hosts:
        - authz-jwt.local
EOF
```

2.token 使用 $TOKEN_GROUP 请求正常，响应 `200`
 
```bash
$ curl -I -H "Authorization: Bearer $TOKEN_GROUP" -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 200 OK
```

3.token 使用 $TOKEN 请求被**网关**拒绝，响应 `403`
```bash
$ curl -I -H "Authorization: Bearer $TOKEN" -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 403 Forbidden
```

4.结合 Claims 条件的拒绝授权示范，httpbin AuthorizationPolicy 对`group=group3`授权

```bash
$ kubectl patch AuthorizationPolicy require-jwt -n $NS --type merge -p '
spec:
  rules:
  - from:
    - source:
        requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
    when:
    - key: request.auth.claims[groups]
      values: ["group3"]
'
```

5.token 使用 $TOKEN_GROUP 请求被** httpbin 服务**拒绝，响应`403`

```bash
$ curl -I -H "Authorization: Bearer $TOKEN_GROUP" -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 403 Forbidden
...
x-envoy-upstream-service-time: 1
```

6.恢复正常授权

```bash
$ kubectl patch AuthorizationPolicy require-jwt -n $NS --type merge -p '
spec:
  rules:
  - from:
    - source:
        requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
    when:
    - key: request.auth.claims[groups]
      values: ["group1"]
'
```

总结如下表，通过 ingress 和 httpbin 分阶段授权策略的搭配，可以将不同授权的 token 在不同阶段进行验证拦截。

ingress 状态 | httpbin 策略 | Token | Ingress 状态 | httpbin 状态
------ | ------ | ------ | ------ | ------
scpoe=scope1 | group=group1 | $GROUP_TOKEN | √ | √
scpoe=scope1 | group=group1 | $TOKEN | 拒绝 | -
scpoe=scope1 | group=group3 | $GROUP_TOKEN | √ | 拒绝

# 小结

本节我们主要实践了在**来源**`from`和**自定义条件**`when`中与**最终用户**相关的属性条件，通过 JWT 标准的 `iss`、`sub`、`aud`和`azp` 以及合理的自定义 claims 设计可以满足大部分访问控制场景的需求，既可以做签发者这样基础的授权，也可以做**最终用户**到服务**接口/方法**级的访问控制。

# 参考

- [Introduction to JSON Web Tokens](https://jwt.io/introduction/)
- [RFC 7519 - JSON Web Token (JWT)](https://tools.ietf.org/html/rfc7519#section-4)
- [JSON Web Key Set](https://auth0.com/docs/tokens/concepts/jwks)
- [Istio Concepts / Security#Request authentication](https://istio.io/docs/concepts/security/#request-authentication)
- [Istio Tasks / Authorization with JWT](https://istio.io/docs/tasks/security/authorization/authz-jwt/)
- [Istio Reference / Authorization Policy](https://istio.io/docs/reference/config/security/authorization-policy/)
- [Istio Reference / Authorization Policy Conditions](https://istio.io/docs/reference/config/security/conditions/)
- [OpenID Connect | Google Identity Platform | Google Developers](https://developers.google.com/identity/protocols/oauth2/openid-connect#an-id-tokens-payload)
