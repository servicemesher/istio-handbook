---
authors: ["hb-chen"]
reviewers: [""]
---

# JWT 授权

前面介绍了`HTTP`、`TCP`、`gRPC`等不同协议的流量授权，主要为服务间的访问控制，而 JWT 授权则是对**最终用户**的访问控制，试想某个内部服务需要管理员才能够使用，这时候就需要验证**最终用户**的角色是否为管理员。不同协议的流量授权在**操作**`to`方面有比较多的示范，本节将更多在**来源**`from`和**自定义条件**`when`做更多示范。

## 准备工作

- kubernetes
    - minikube
- 安装`istioctl`
- Istio 环境
    - manifest default 安装，带有 IngressGateway
- 下载 Istio Release 1.5.0
    - httpbin 示例

## httpbin 服务部署

### 创建命名空间

设置命名空间环境变量`NS=authz-jwt`，并创建命名空间

```bash
$ export NS=authz-jwt
$ kubectl create namespace $NS
```

### Sidecar 自动注入

为命名空间开启 Sidecar 自动注入

```bash
$ kubectl label namespace $NS istio-injection=enabled
```

### 部署 httpbin 服务

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

## 创建 httpbin 网关

### 获取 Ingress 网关的 IP 和 PORT

> 根据环境不同获取 ingress IP 和 Port 参考[determining-the-ingress-ip-and-ports](https://istio.io/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-ip-and-ports)

```bash
# 演示使用的是 minikube 环境
$ export INGRESS_IP=$(minikube ip)
$ export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
```

### 添加 httpbin 服务的 gateway

为了避免与其他网关产生冲突，为网关指定 HOST 为 `authz-jwt.local`，这样在测试时通过指定 HOST 确定是路由到 httpbin服务。

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

### 验证网关

```bash
$ curl -I -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 200 OK
```

## 添加 JWT 认证

要使用 JWT 授权首先要为服务添加终端身份认证即 RequestAuthentication ，更多参考认证章节`3.4.1`。

### 添加 RequestAuthentication

> 使用 Istio 代码库中的 JWKS 端点 并使用此示例的 deom 和 groups-scope token，包含两个 toke，demo 是一个普通 token 有 JWT 的基础属性，groups-scope 是一个带有自定义属性的 token，属性包括 group 和 scope，相关连接:
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

使用两种方式测试服务:
- Authorization header 携带一个无效的 token
- 不带 Authorization header

1.携带无效 token 的请求被绝，响应`401`

```bash
$ curl -I -H "Authorization: Bearer invalidToken" -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 401 Unauthorized
```

2.不带 Authorization header 的请求仍然有效，响应`200`

这是因为 RequestAuthentication 只负责验证 token 的有效性，token 的有无以及是否授权访问由 AuthorizationPolicy 策略决定，所以在只有 RequestAuthentication 时，没有 Authorization header 的请求仍然可以正常访问。

```bash
$ curl -I -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers -s -o  /dev/null -w "%{http_code}\n"
HTTP/1.1 200 OK
```

## 添加 JWT 授权

### 添加 AuthorizationPolicy

添加一个 `from.source` 为 `requestPrincipals: ["*"]`的授权策略

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
 rules:
 - from:
   - source:
       requestPrincipals: ["*"]
EOF
```

有了 AuthorizationPolicy 后再测试没有 Authorization header 的请求会被拒绝，响应`403`。`requestPrincipals: ["*"]` 虽然不限定 Principal， 但必须带有有效的 token 才允许访问
```bash
$ curl -I -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 403 Forbidden
```

### demo token 测试

1.获取 demo token

获取 demo token 设置环境变量，并查看 JWT token 结构
```bash
$ TOKEN=$(curl https://raw.githubusercontent.com/istio/istio/release-1.5/security/tools/jwt/samples/demo.jwt -s) && echo $TOKEN | cut -d '.' -f2 - | base64 -d -
{"exp":4685989700,"foo":"bar","iss":"testing@secure.istio.io","sub":"testing@secure.istio.io","iat":1532389700}
```

> demo token 结构
```json
{
  "exp" : 4685989700,
  "foo" : "bar",
  "iss" : "testing@secure.istio.io",
  "sub" : "testing@secure.istio.io",
  "iat" : 1532389700
}
```

2.授权测试正常，相应`200`

```bash
$ curl -I -H "Authorization: Bearer $TOKEN" -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 200 OK
```

## Principal 条件

在`from.soruce`中与 JWT 相关的属性为 `requestPrincipals` 和 `notRequestPrincipals`，这里使用`requestPrincipals`进行测试

### 修改`source.requestPrincipals`规则

`source.requestPrincipals`条件是 JWT 属性`iss`和`sub`的组合 `{iss}/{sub}`，测试 JWT token 的 Principal 为 `testing@secure.istio.io/testing@secure.istio.io`

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
       requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
EOF
```

`source.requestPrincipals`与自定义条件`when`的`request.auth.principal`是等效的，所以也可以这样配置

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
  - when:
    - key: request.auth.principal
      values: ["testing@secure.istio.io/testing@secure.istio.io"]
EOF
```

请求正常，响应`200`
```bash
$ curl -I -H "Authorization: Bearer $TOKEN" -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 200 OK
```

### 拒绝授权示范

1.这时如果我们将 `requestPrincipals`规则改为其它值`requestPrincipals: ["testing@secure.istio.io/none"]`

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

`from.source`的规则中与 JWT 相关的属性只有`requestPrincipals`和`notRequestPrincipals`，更多的有关 JWT 属性的规则可以通过自定义条件`when`补充，其中与 JWT 相关的属性对应关系:
when condition name | JWT claims
-----|------
`request.auth.principal` | iss/sub
`request.auth.audiences` | aud
`request.auth.presenter` | azp
`request.auth.claims` | JWT 全部属性

接下来在 Principal 规则基础上增加 claims 规则

### groups-scope token 测试 when 条件

1.获取 groups-scope token

```bash
$ TOKEN_GROUP=$(curl https://raw.githubusercontent.com/istio/istio/release-1.5/security/tools/jwt/samples/groups-scope.jwt -s) && echo $TOKEN_GROUP | cut -d '.' -f2 - | base64 -d -
{"exp":3537391104,"scope":["scope1","scope2"],"iss":"testing@secure.istio.io","groups":["group1","group2"],"sub":"testing@secure.istio.io","iat":1537391104}
```

> groups-scope token 结构
```json
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

2.结合 JWT 结构这里使用 groups 作为自定义条件，如仅允许: group1

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
       requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
    when:
    - key: request.auth.claims[groups]
      values: ["group1"]
EOF
```

3.授权测试
```bash
$ curl -I -H "Authorization: Bearer $TOKEN_GROUP" -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 200 OK
```

### 拒绝授权示范

1.尝试一个不在 groups-scope token 内的 group 值，如`values: ["group3"]`

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

2.再次测试访问被拒绝，响应`403`

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

## 分阶段认证与授权

现在每次请求对 JWT token 的验证都是在 httpbin 服务上，而对于真实场景的请求到达内部服务，往往要经过 n 个服务，如果恰巧这个授权验证是最后一个服务，当因为 token 无效或者没有 token 导致失败时，服务的响应时间大大延长，并造成资源的浪费，所以可以将 token 的验证前置到 Ingress 网关。
通过前面的实践可以知道添加 RequestAuthentication 仅对带有 Authorization header 请求做认证，不影响无 header 的请求。具体是否需要分阶段验证，以及在什么位置验证，需要根据业务场景考虑，一般越是顶层条件越靠前如`from.source.requestPrincipals`、`to.operation.hosts`，
而`to.operation.methods/paths`、`when.request.auth.claims[group/scope]`适合服务做详细的授权策略。

另外需要注意的是如果调用链路有多次使用同一个 token，则必须在 RequestAuthentication 的`jwtRules`中开启`forwardOriginalToken: true`以将 Authorization header 向下传递，也可以通过 fromHeaders / fromParams 携带多个不同场景的 token，具体参考 [JWTRule](https://istio.io/docs/reference/config/security/jwt/#JWTRule) 。说到 token 的传递，Authorization header 也可以在服务与服务间调用时添加，所以**最终用户**的定义并不限定为客户端，一个发起调用的服务也是一个**最终用户**。 

### Ingress JWT 认证

1.测试当前无效 token 请求，响应 `401`， 并且在响应的 header 中有`x-envoy-upstream-service-time: 1`，请求被** httpbin 服务**拒绝

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

3.再测试无效 token 请求，同样响应 `401`，但没有了`x-envoy-upstream-service-time: 1`，说明请求是在**网关**被拒绝

```bash
$ curl -I -H "Authorization: Bearer invalidToken" -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 401 Unauthorized
```

### Ingress JWT 授权

1.前面只是 Ingress 的 JWT 认证，结合网关的入口特点可以添加根据 HOST 的不同限定`scope`的授权，如: 访问 `host=authz-jwt.local`要求`scpoe=scope1`

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

2.Authorization 使用 TOKEN_GROUP 请求正常，响应 `200`
 
```bash
$ curl -I -H "Authorization: Bearer $TOKEN_GROUP" -H "Host: authz-jwt.local" http://$INGRESS_IP:$INGRESS_PORT/headers
HTTP/1.1 200 OK
```

3.Authorization 使用 TOKEN 请求被**网关**拒绝，响应 `403`
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

5.请求被** httpbin 服务**拒绝，响应`403`

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

# 小结

本节我们主要实践了在**来源**`from`和**自定义条件**`when`中与**最终用户**相关的属性条件，通过 JWT 标准的 `iss`、`sub`、`aud` 以及合理的 claims 设计可以满足大部分访问控制场景的需求，既可以做签发者这样基础的授权，也可以做**最终用户**到服务**接口/方法**级的访问控制。

# 参考

https://developers.google.com/identity/protocols/oauth2/openid-connect
https://tools.ietf.org/html/rfc7519#section-4
https://istio.io/docs/concepts/security/#request-authentication
https://istio.io/docs/tasks/security/authorization/authz-jwt/
https://istio.io/docs/reference/config/security/authorization-policy/#Source
https://istio.io/docs/reference/config/security/conditions/

