---
authors: ["tx19980520"]
reviewers: [""]
---

# 双向 TLS

TLS 在 web 端的使用非常广泛，针对传输的内容进行加密，能够有效的防止中间人攻击。双向TLS（Two way TLS/Mutual TLS，后文均简称 mTLS）的主要使用场景是在 B2B 和 Server-to-Server 的场景中，使得服务和服务之间能够相互鉴权。
![istio-official-authorization-arch](../images/authz.png)

上图源引[Istio官网](https://istio.io/docs/concepts/security/authz.svg)
，图中非常明确的表示 Istio 所希望的是在 Service Mesh 中能够使用 mTLS 进行 authorization，而在 Service Mesh 外使用 JWT+mTLS 进行 authorization。服务间身份认证是使用 mTLS，来源身份验证中则是使用 JWT。

本文将一一阐述 mTLS handshake 的过程，Envoy 中如何实现 handshake，以及 Istio 中在各个方面如何使用，在 Kubernetes 集群中使用有哪些注意事项，和选择 mTLS 的合适使用场景几个方面。

## mTLS 在 Istio 中的使用

Istio 中提供了 AuthorizationPolicy 用于对 trust domain 进行设置，能够对证书做到更加细粒度的验证。具体的一些实践我们也会在后面的章节中进行实践与探讨。

### mTLS 的基本使用

测试环境结构如图，共拥有 full legacy mtls-test 三个 namespace，对 full，mtls-test 设置为自动注入 sidecar。

![istio-mutual-simple-test](../images/istio-mutual-simple-test.jpg)

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: mtls-test
  name: httpbin
  labels:
    app: httpbin
spec:
  ports:
  - name: http
    port: 8000
    targetPort: 80
  selector:
    app: httpbin
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: mtls-test
  name: httpbin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
      version: v1
  template:
    metadata:
      labels:
        app: httpbin
        version: v1
    spec:
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        ports:
        - containerPort: 80
---
```

sleep.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: full
  name: sleep
  labels:
    app: sleep
spec:
  ports:
  - port: 80
    name: http
  selector:
    app: sleep
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: full
  name: sleep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sleep
  template:
    metadata:
      labels:
        app: sleep
    spec:
      containers:
      - name: sleep
        image: governmentpaas/curl-ssl
        command: ["/bin/sleep", "3650d"]
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - mountPath: /etc/sleep/tls
          name: secret-volume
      volumes:
      - name: secret-volume
        secret:
          secretName: sleep-secret
          optional: true
---
```

#### 使用默认的 PERMISSIVE 模式

默认情况下， PERMISSIVE 模式能够支持明文传输，则不管是在 Istio 管理下的 Pod 还是在 Istio 管理外的 Pod，相互之间的通信畅通无阻，相关证书的内容也不需要使用者自行签发。

```bash
for from in "mtls-test" "legacy"; do for to in "mtls-test"; do echo "sleep.${from} to httpbin.${to}";kubectl exec $(kubectl get pod -l app=sleep -n ${from} -o jsonpath={.items..metadata.name}) -c sleep -n ${from} -- curl http://httpbin.${to}:8000/headers  -s  -w "response code: %{http_code}\n" | egrep -o 'URI\=spiffe.*sa/[a-z]*|response.*$';  echo -n "\n"; done; done

sleep.mtls-test to httpbin.mtls-test
URI=spiffe://cluster.local/ns/mtls-test/sa/sleep
response code: 200
sleep.legacy to httpbin.mtls-test
response code: 200
sleep.full to httpbin.mtls-test
URI=spiffe://cluster.local/ns/full/sa/sleep
response code: 200
```

这里需要强调的是从 `sleep.mtls-test` 到 `httpbin.mtls-test` 这里能够找到 SPIFFE 的相关内容，SPIFFE URI 显示来自 X.509 证书的客户端标识，它表明流量是在双向 TLS 中发送的。如果流量为明文，将不会显示客户端证书。

#### 启用 STRICT 模式

```yaml
apiVersion: "networking.istio.io/v1alpha3"
kind: "DestinationRule"
metadata:
  name: "httpbin-mtls-test-mtls"
spec:
  host: httpbin.mtls-test.svc.cluster.local
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
---
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "httpbin"
  namespace: "mtls-test"
spec:
  selector:
    matchLabels:
      app: httpbin
  mtls:
    mode: STRICT
---
```

此时我们已经要求进入到 `httpbin.mtls-test` 的流量必须是 mTLS 模式，则 `httpbin.legacy` 没有办法获取到最终的结果，再次发送请求以验证：

```
sleep.mtls-test to httpbin.mtls-test
URI=spiffe://cluster.local/ns/mtls-test/sa/sleep
response code: 200
sleep.legacy to httpbin.mtls-test
response code: 000
command terminated with exit code 56
sleep.full to httpbin.mtls-test
URI=spiffe://cluster.local/ns/full/sa/sleep
response code: 200
```

上述的实验已经能够表明我们可以在证书的层面进行较为粗粒度的访问管理，进一步的我们对 SPIFFE 的内容开始感兴趣，我们希望针对 SAN 能够获取到更为细粒度的控制。

对此，我们应该对 Istio 实现的证书有一个更进一步的了解。
Istio 在1.5版本前，会在每一个 namespace 下创建一个 `istio.default` 的 secret，存储默认的 CA 文件，并会被 mount 在 sidecar 中以供使用，但是在1.5版本中，不会再存储在 secret 中，只能通过 rpc 调用才能获取到响应的内容，内容最终会被分配在内存中，但我们可以使用 `openssl s_client` 进行访问来获得证书。

```bash
kubectl exec <pod> -c istio-proxy -- openssl s_client -alpn istio -connect <service:port> # 获取到证书
openssl x509 -text -noout -in server.pem # 对上一个cmd返回内容中的server.pem 进行解析
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            # Secret Content
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: O=cluster.local
        Validity
            Not Before: Apr 10 03:38:12 2020 GMT
            Not After : Jul  9 03:38:12 2020 GMT
        Subject: 
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    # Secret Content
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Subject Alternative Name: critical
                URI:spiffe://cluster.local/ns/default/sa/default
    Signature Algorithm: sha256WithRSAEncryption
         # Secret Content
```

我们接下来要重点关注的就是这里所展示的 SAN

### mTLS 与 Trust Domain

Turst Domain 是 Istio 1.4 版本进入 alpha 的一个功能，在我们修改 Trust Domain 时，实质上是修改了 SAN 区域的值，重新签发了新的证书(重新签发证书需要一定的时间，因此在配置之后需要等待一段时间才能生效)。
针对此我们需要重点关注的 CRD 为 `AuthorizationPolicy`。
在我们的实验中，我们将设置 `sleep.full` 不允许访问 `httpbin.mlts-test`。

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: httpbin.mtls-test.svc.cluster.local
  namespace: mtls-test
spec:
  action: DENY
  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/full/sa/sleep
    to:
    - operation:
        methods:
        - GET
  selector:
    matchLabels:
      app: httpbin
---
```

我们再次进行测试，最终的结果如下：

```
sleep.mtls-test to httpbin.mtls-test
URI=spiffe://cluster.local/ns/mtls-test/sa/sleep
response code: 200
sleep.legacy to httpbin.mtls-test
response code: 000
command terminated with exit code 56
sleep.full to httpbin.mtls-test
response code: 403
```

我们看到当 `sleep.full` 请求 `httpbin.mtls-test` 时，此时返回403，说明其存在证书，但是证书的的 SAN 值域并不在 Trust Domain 中(配置文件类似于配置了黑名单)，因此返回403。

### mTLS 与数据库

#### MongoDB 内置 mTLS

数据库作为 cloud 中因其 stateful 的特性以及对于性能和安全的要求，一直存在着许许多多的问题，今天我们就以 MongoDB 为例，通过 mTLS 访问外的 MongoDB 服务。MongoDB 自身能够提供 mTLS 的服务，首先我们在 Kubernetes 首先实现 mTLS，再将 client 放入到 Service Mesh 中。
首先使用 openssl 自行签发证书：

```bash
openssl req -out ca.pem -new -x509 -days 3650 -subj "/C=CN/CN=root/emailAddress=11111@qq.com" # 生成根证书
openssl genrsa -out server.key 2048 # 生成服务器私钥
openssl req -key server.key -new -out server.req -subj "/C=CN/CN=mongo/emailAddress=11111@qq.com" # 生成服务端证书申请文件
openssl x509 -req -in server.req -CAkey privkey.pem -CA ca.pem -days 36500 -CAcreateserial -CAserial serial -out server.crt # 生成服务端证书
cat server.key server.crt > server.pem # 合并证书与私钥
openssl verify -CAfile ca.pem server.pem # 验证证书
openssl genrsa -out client.key 2048 # 生成客户端私钥
openssl req -key client.key -new -out client.req -subj "/C=CN/CN=client/emailAddress=11111@qq.com" # 生成客户端证书申请文件
openssl x509 -req -in client.req -CAkey privkey.pem -CA ca.pem -days 36500 -CAserial serial -out client.crt # 生成客户端证书
cat client.key client.crt > client.pem # 合并证书与私钥
openssl verify -CAfile ca.pem client.pem # 验证证书
```

至此我们已经获得了三个证书，`ca.pem`作为根证书，`server.pem` 和`client.pem`分别作为server和client的证书。
简单使用 ConfigMap 在实验中将证书 mount 进入到 Pod 中。

```bash
kubectl create configmap -n mongo mongo-server-pem --from-file=./ssl/server.pem --from-file=./ssl/ca.pem
kubectl create configmap -n mongo mongo-client-pem --from-file=./ssl/client.pem --from-file=./ssl/ca.pem
```

server:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo
  namespace: mongo
  labels:
    name: mongo
spec:
  ports:
  - port: 27017
    targetPort: 27017
  clusterIP: None
  selector:
    app: mongo
---
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: mongo
  namespace: mongo
spec:
  serviceName: mongo
  replicas: 1
  template:
    metadata:
      labels:
        app: mongo
    spec:
      terminationGracePeriodSeconds: 10
      containers:
        - name: mongo
          image: mongo:4.2
          command:
            - mongod
            - "--bind_ip"
            - 0.0.0.0
            - "--tlsMode"
            - "requireTLS"
            - "--tlsCertificateKeyFile"
            - "/pem/server.pem"
            - "--sslCAFile"
            - "/pem/ca.pem"
          ports:
            - containerPort: 27017
          volumeMounts:
          - name: ssl
            mountPath: /pem
      volumes:
        - name: ssl
          configMap:
            name: mongo-server-pem  
---    
```

client:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: mongo
  name: mongo-client
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo-client
      version: v1
  template:
    metadata:
      labels:
        app: mongo-client
        version: v1
    spec:
      containers:
      - image: mongo:4.2
        imagePullPolicy: IfNotPresent
        name: mongo-client
        volumeMounts:
          - name: ssl
            mountPath: /pem
      volumes:
        - name: ssl
          configMap:
            name: mongo-client-pem
---
```

之后我们在 client 中进行试验：

```bash
root@mongo-client-548f5974f-xbxx6: mongo --tls --tlsCAFile /pem/ca.pem --tlsCertificateKeyFile /pem/client.pem --host mongo
MongoDB shell version v4.2.6
connecting to: MongoDB://mongo:27017/?compressors=disabled&gssapiServiceName=MongoDB
Implicit session: session { "id" : UUID("150e352f-5635-4af2-9ad7-f37e2840ec6b") }
MongoDB server version: 4.2.6
Welcome to the MongoDB shell.


root@mongo-client-548f5974f-xbxx6:/ mongo  --host mongo
MongoDB shell version v4.2.6
connecting to: MongoDB://mongo:27017/?compressors=disabled&gssapiServiceName=MongoDB
2020-05-02T06:19:11.562+0000 I  NETWORK  [js] DBClientConnection failed to receive message from mongo:27017 - HostUnreachable: Connection closed by peer
2020-05-02T06:19:11.562+0000 E  QUERY    [js] Error: network error while attempting to run command 'isMaster' on host 'mongo:27017'  :
connect@src/mongo/shell/mongo.js:341:17
@(connect):2:6
2020-05-02T06:19:11.565+0000 F  -        [main] exception: connect failed
2020-05-02T06:19:11.565+0000 E  -        [main] exiting with code 1


root@mongo-client-548f5974f-xbxx6:/ mongo --tls --tlsCAFile /pem/ca.pem --tlsCertificateKeyFile /pem/ca.pem --host mongo
2020-05-02T06:30:25.551+0000 E  NETWORK  [main] cannot read PEM key file: /pem/ca.pem error:0909006C:PEM routines:get_name:no start line
Failed global initialization: InvalidSSLConfiguration Can not set up PEM key file.


root@mongo-client-548f5974f-xbxx6:/ mongo --tls --tlsCAFile /pem/client.pem --tlsCertificateKeyFile /pem/client.pem --host mongo
MongoDB shell version v4.2.6
connecting to: MongoDB://mongo:27017/?compressors=disabled&gssapiServiceName=MongoDB
2020-05-02T06:30:36.144+0000 E  NETWORK  [js] SSL peer certificate validation failed: self signed certificate in certificate chain
2020-05-02T06:30:36.145+0000 E  QUERY    [js] Error: couldn't connect to server mongo:27017, connection attempt failed: SSLHandshakeFailed: SSL peer certificate validation failed: self signed certificate in certificate chain :
connect@src/mongo/shell/mongo.js:341:17
@(connect):2:6
2020-05-02T06:30:36.147+0000 F  -        [main] exception: connect failed
2020-05-02T06:30:36.148+0000 E  -        [main] exiting with code 1
```

上述的输出可以表明，在没有证书和证书不正确的情况下都无法连入数据库，我们能够正确使用 mTLS 模式访问MongoDB，在访问 MongoDB 上现在已经有了非常安全的保障，但是在 client 端需要开发者自行处理有关证书的事宜，这不仅仅会给开发者带来困扰，也会将证书与私钥对外暴露，如果 client 在 Service Mesh 内部，我们可以让 sidecar 来负责相关的工作。

#### Istio mTLS 结合 MongoDB

我们主要针对 client 端进行改造，将其部署在 Service Mesh 中。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: mtls-test
  name: mongo-client
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo-client
      version: v1
  template:
    metadata:
      labels:
        app: mongo-client
        version: v1
      annotations:
        sidecar.istio.io/userVolume: '[{"name":"client-ssl", "configMap":{"name":"mongo-client-pem"}}]'
        sidecar.istio.io/userVolumeMount: '[{"name":"client-ssl", "mountPath":"/pem"}]'
      
    spec:
      containers:
      - image: mongo:4.2
        imagePullPolicy: IfNotPresent
        name: mongo-client
---
```

其中 `template.annotations` 可参考[Resource Annotations](https://preliminary.istio.io/docs/reference/config/annotations/)，这里主要的用途是为了将我们 client 端的相关证书等内容 mount 进入到 `istio-proxy` 容器中。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  namespace: mtls-test
  name: db-mtls
spec:
  host: mongo.mongo.svc.cluster.local
  trafficPolicy:
    tls:
      mode: MUTUAL
      clientCertificate: /pem/client.pem
      privateKey: /pem/client.key
      caCertificates: /pem/ca.pem
---
```

设置相应的 DestinationRule，应要注意 tls.mode 为 `MUTUAL`，并未使用 `ISTIO_MUTUAL`。相应证书的路径和 Deployment 中的 annotation 相呼应。
在部署后我们进入到 mongo-client 容器中，进行连接的实验：

```bash
root@mongo-client-7557b58674-f2rvc:/ mongo --host mongo.mongo
MongoDB shell version v4.2.6
connecting to: MongoDB://mongo.mongo:27017/?compressors=disabled&gssapiServiceName=MongoDB
Implicit session: session { "id" : UUID("19b8f417-4e6b-4ef6-9fda-15e9f1703ebf") }
MongoDB server version: 4.2.6
Welcome to the MongoDB shell.
```

可以看到这次我们在创建连接时并没有使用设置相应 TLS 的内容，这就是因为将 TLS 层的内容前移到了 istio-proxy 中。

### mTLS 与 Kubernetes 探针

我们常在 Kubernetes 集群中使用探针，用于检测服务的健康状态，而这样的探针，往往是一个 HTTP 请求，那么在我们使用探针时，如果开启 mTLS 严格模式，则会出现探针报错的情况，但实际上我们的服务应该是可以正常运行的。

```bash
kubectl get pods -n mtls-test --watch
NAME                            READY   STATUS             RESTARTS   AGE
httpbin-5446f4d9b4-kxsjx        2/2     Running            0          4h4m
httpbin-probe-79d8c47c6-rmwkl   1/2     CrashLoopBackOff   4          115s
sleep-666475687f-cp252          2/2     Running            0          6h23m
```

因为探针的最终实施者是 kubelet，但是 kubelet 在执行探测时，并不会携带相应合法的证书，因此会被 sidecar 拒绝请求，返回一个非 2xx 的 response code，TCP 同理，因此需要在该方面上得到豁免。

Istio 官方文档也给出了相应的答案，通过添加注解的方式达成豁免。

```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: mtls-test
  name: httpbin-probe
  labels:
    app: httpbin-probe
spec:
  ports:
  - name: http
    port: 8000
    targetPort: 80
  selector:
    app: httpbin-probe
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: mtls-test
  name: httpbin-probe
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin-probe
      version: v1
  template:
    metadata:
      labels:
        app: httpbin-probe
        version: v1
      annotations:
        sidecar.istio.io/rewriteAppHTTPProbers: "true"
    spec:
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /headers
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
---
```

Deployment 的主要修改在 `template.metadata` 中添加了 `sidecar.istio.io/rewriteAppHTTPProbers` 通过 rewrite 的方式保证 probe 能够正常工作。

## mTLS 的使用建议

在当今微服务架构盛行的情况下，服务于服务之间的调用是非常频繁的，这也是 Istio 在 Envoy 中启用 mTLS 的一个原因，但是我们在实际的使用中，就是否需要使用mTLS应当按需使用。

由于我们大多数用户在构建 image 的时候会选择从一些基础的镜像进行搭建，这就给了某些 hacker 可乘之机，进入到集群之中，并执行相应的代码，比如该服务为 Hack，选择了尝试 fetch 另一个内部服务 Cost 的接口 /books/cost 去获取到所有书本的成本价。Cost 此时在 Service Mesh 中，但是 Hack 并不在 Service Mesh 中，如果我们选择使用 PERMISSIVE 模式，则 Hack 的行为合法，如果使用 STRICT 模式，则 Hack 将无法访问到相应资源，返回 Status Code 503。

除此之外，我们需要在 Security 和 Performance 两者间进行一个考虑，虽然我们可以创建更少的 connection 来避免总的握手次数（多路复用），但是对称加密解密也是非常耗时和消耗 CPU 资源的，有关性能消耗的分析，可见一些评测机构对此作出的相关评测。

综上，如果是对性能较为敏感，且数据的敏感性不强，数据库也仅限集群内部访问，可以考虑使用明文传输。但如果数据敏感或业务逻辑需要安全方面的考量，建议使用 mTLS。




## 参考资料

* [Authentication Architecture](https://istio.io/docs/concepts/security/#authentication-architecture)
* [Subject Alternative Name](https://en.wikipedia.org/wiki/Subject_Alternative_Name)
* [Authz-td-Migration](https://istio.io/docs/tasks/security/authorization/authz-td-migration)
* [Istio 1.5 upgrade notes](https://istio.io/news/releases/1.5.x/announcing-1.5/upgrade-notes)
* [MongoDB Configure TLS](https://docs.MongoDB.com/manual/tutorial/configure-ssl)
* [MongoDB TLS Performance](https://www.synopsys.com/content/dam/synopsys/sig-assets/case-studies/tls-performance-overhead-MongoDB.pdf)
