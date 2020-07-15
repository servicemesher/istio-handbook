---
authors: ["wbpcode"]
reviewers: ["rootsongjc"]
---

# Envoy

Istio 通过在业务容器之中注入 Sidecar 来实现对微服务集群中东西向流量的接管。而为了能在对外提供服务的同时，不向客户端暴露整个服务网格，Istio 则需要对接 API 网关来管理微服务集群的南北向流量。

Istio 会将需要对外暴露的功能和接口注册到 API 网关，然后由 API 网关对外部客户端提供访问内部服务网格的统一入口。API 网关会负责提供外部请求的基础路由、负载均衡、认证鉴权、流量审计等等功能。在绝大部分情况下，API 网关都通过 HTTP 协议对外暴露相关服务。

作为 Istio 服务网格所使用的默认数据面，Envoy 自然也能够胜任 API 网关的角色。实际上，**在 Istio 社区提供的部署方案之中，就已经包含了一套以 Envoy 为数据面、Istio 本身为控制面的 API 网关**。可以通过以下命令查看 Istio 中 Envoy 网关相关组件：

```bash
$ kubectl get deploy -n istio-system
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
grafana                1/1     1            1           91m
istio-egressgateway    1/1     1            1           91m    #  <-- 出口网关
istio-ingressgateway   1/1     1            1           91m    #  <-- 入口网关
istio-tracing          1/1     1            1           91m
istiod                 1/1     1            1           92m
kiali                  1/1     1            1           91m
prometheus             1/1     1            1           91m
```

在 istio-system 名称空间下，可以看到名为 istio-egressgateway 和 istio-ingressgateway 的两个额外的负载，前者用于管控服务网格的出口流量，而后者用于管控服务网格的入口流量，两者都是 Envoy 实例。当然，在大部分场景之中，istio-ingressgateway 更符合本节中 API 网关的角色，所以，接下来本节都将以 istio-ingressgateway 为例进行介绍。如果读者目前仍旧没有自己的 Istio 环境，可以按照 [安装与部署](./setup.md) 中教程重新部署一套环境用于亲手实践本节接下来的内容。

## 预备工作

在介绍更多的关于 Envoy 网关对接的内容之前，需要做一些准备工作。比如，需要一个服务网格、一个微服务集群。当然，为了简单起见，本节只准备了一个服务，用来模拟需要对外暴露 API 的后端集群：Httpbin。Httpbin 是一个强大的 HTTP 响应模拟器，可以根据请求特征和要求来模拟出各种 HTTP 响应，比如响应 JSON 数据、响应变长二进制数据、响应特定状态码等等，它被广泛用于各种 HTTP 测试场景之中。 可以使用如下的命令在 Kubernetes 集群之中创建一个新的 Httpbin 服务：

```bash
$ kubectl create -f <https://github.com/istio/istio/raw/master/samples/httpbin/httpbin.yaml>
```

执行上述命令之后，可以通过以下命令查看服务是否成功创建并验证 Httpbin 工作是否正常。可以看到，示例 Httpbin 服务打开了一个 8000 端口，服务名称为 httpbin（注意区分 Httpbin 和 httpbin 含义不同，前者为应用类型，后者为服务具体名称）。Httpbin 提供了一个 `/anything` 接口，可以把客户端请求的详细内容如请求头、请求体以 JSON 格式响应返回。可以通过该接口直接验证 Httpbin 是否正常工作。

```bash
$ kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
httpbin      ClusterIP   10.106.96.163   <none>        8000/TCP   5d23h
$ curl $SERVICE_IP_OF_HTTPBIN:8000/anything
{
  "args": {},
  "data": "",
  "files": {},
  "form": {},
  "headers": {
    "Accept": "*/*",
    "Host": "10.106.96.163:8000",
    "User-Agent": "curl/7.65.3"
  },
  "json": null,
  "method": "GET",
  "origin": "10.244.0.1",
  "url": "<http://10.106.96.163:8000/anything>"
}
```

## 开始监听

在验证 httpbin 服务能够正常工作，完成准备工作之后，现在才能够步入正题。作为 API 网关，自然必须能够监听并处理来自客户端的请求。首先，开发人员需要确认 Envoy 的端口监听状态。Envoy 本身提供了一组管理接口，用于获取 Envoy 内部信息，如当前配置、监听器、集群、日志等级、指标监控数据等等。在 Istio 的默认部署之中，Envoy 会打开 15000 端口来向用户提供相关管理接口。首先通过 kubectl 相关命令进入到容器内部，然后使用如下所示 API 可以获取 Envoy 当前所有监听器（监听器等 Envoy 相关概念请参考本书概念篇）：

注：在具体的实践当中，请将命令中容器名称替换为实际环境中对应容器名称。

```bash
$ kubectl exec $POD_NAME_OF_INGRESS -n istio-system -it -- bash
root@$POD_NAME_OF_INGRESS:/# curl localhost:15000/listeners
dded9b9c-3bdb-4ada-897b-0ab1acc3f960::0.0.0.0:15090  # <-- 用于向 Prometheus 提供指标监控
e63c67b4-8b1f-4949-93a7-69ad68d93c95::0.0.0.0:15021  # <-- 用于健康检查
```

从上述接口响应可以发现，此时 Envoy 仅仅打开了两个 15090 和 15021 两个端口，分别用于对外提供指标监控数据以及健康检查。而通过 Envoy 提供的 `/config_dump` 接口则可以看到关于这两个监听器的更多内容。考虑到 config_dump 响应较大，结构较为复杂，因此不直接在本节中粘贴相关内容，读者可以自己尝试。

```bash
root@$POD_NAME_OF_INGRESS:/# curl localhost:15000/config_dump
```

显然，只开启了上述两个端口的 Envoy 目前仍旧不具备路由外部请求的能力。要让 Envoy 网关能够正常工作，接下来必须为 Envoy 网关创建 Gateway 资源。Gateway 可以看作是 Istio 对 Enovy 中监听器资源的一种抽象，也可以认为它就是一个虚拟网关。开发人员可以动态的创建 Gateway，再由 Istio 将 Gateway 转换为 Envoy 相关配置，通过 xDS 协议下发给 Envoy，最后由 Envoy 根据配置创建对应的监听器。

Gateway 中包含监听端口、协议等配置。下面是一个示例，该示例在 Envoy 网关上打开 80 端口，并代理 HTTP 协议的请求：

```bash
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "httpbin.example.com"
EOF
```

其中 selector 使用 Kubernetes 标签来筛选不同的 Envoy 网关实例。只有符合标签条件的实例才会收到 Istio 下发的配置。在当前的示例当中，自然是 istio-ingressgateway 负载下的 Envoy 实例可以获得更新和配置。对于 Kubernetes 不甚熟悉的读者可以直接执行以下命令来确认不同的负载实例上的标签值：

```bash
kubectl get pod -n istio-system --show-labels
```

而 Gateway 资源中的 hosts 字段则用于指定网关对外暴露的域名，同时该 hosts 字段也可以用于对绑定的路由规则做一些限制。而再次调用对应 Envoy 实例管理端口，可以发现，已经有了一个全新的监听器：

```bash
root@$POD_NAME_OF_INGRESS:/# curl localhost:15000/listeners
dded9b9c-3bdb-4ada-897b-0ab1acc3f960::0.0.0.0:15090
e63c67b4-8b1f-4949-93a7-69ad68d93c95::0.0.0.0:15021
0.0.0.0_8080::0.0.0.0:8080
```

但是此时，如果直接访问对应的监听端口，会发现所有的请求返回都是 404，因为现在还没有任何有效的路由，所以 Istio 会下发一条默认路由，对于所有到达该端口的 HTTP 请求，统一返回 404。

## 一条路由

只有 Gateway，Envoy 仍旧无法完成任何工作。就像是空立的一扇门，打开后也到不了任何地方，必须要开出路来，画上路标，才能四通八达。所以，现在需要路由为到达端口的请求指明方向。

对于路由，Istio 使用了名为 Virtual Service 的资源来抽象和封装。一个 Virtual Service 资源就是一条或者一组路由。下面是一个 Virtual Service 示例。

```bash
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "httpbin.example.com"
  gateways:
  - httpbin-gateway
  http:
  - match:
    - uri:
        prefix: /status
    - uri:
        prefix: /delay
    route:
    - destination:
        port:
          number: 8000
        host: httpbin
EOF
```

在上述示例之中，其中 gateways 字段用于指定当前 Virtual Service 需要绑定生效的 Gateway。也即是说，该 Virtual Service 定义的路由规则，要下发到哪些对应的监听器之上。目前该 Virtual Service 仅仅绑定在名为 httpbin-gateway 的 Gateway 资源之中（Gateway 资源本身如前文所述也会限定配置生效的实例，所以最终该路由仅仅在 istio-ingressgateway 中生效）。而 hosts 字段则有两方面的作用。第一，Virtual Service 的 hosts 中相关的域名应当与其绑定的 Gateway 资源中 hosts 域名相匹配。最终对外暴露域名是以 Gateway 资源中配置为准。通过该规则限制，一定程度上，可以起到 Virtual Service 配置筛选的作用。第二，hosts 字段中域名也是该 Virtual Service 对应路由的请求筛选条件。只有域名与 hosts 相匹配的请求才能够命中对应的路由。在 Virtual Service 中，另外有个关键字段，分别是 match 和 route。其中 match 用于指定更具体的路由匹配条件，如请求路径、请求参数等。而 route 则用于指定路由的目标服务。注意到，目标服务是一个数组类型，可以配置多个服务，读者可以后续自行体验配置多个服务之后，Virtual Service 的最终效果。

在当前的示例之中，该 Virtual Service 的目标服务为 httpbin，对应端口为 8000。在完成路由创建之后，现在开始验证该路由是否生效。

使用如下请求访问 API 网关。其中 IP 地址为 istio-ingressgateway 的服务地址（Kubernetes 中服务、负载以及 Pod 实例等概念的相关内容本书不多做赘述）。注意到，该请求使用了 -H 参数来指定请求域名。读者可以自行试验不指定域名时请求的结果。

```bash
$ curl $SERVICE_IP_OF_INGRESS:80/delay/3  -H"host:httpbin.example.com"
{
  "args": {},
  "data": "",
  "files": {},
  "form": {},
  "headers": {
    "Accept": "*/*",
		# .....
 },
  "origin": "10.244.0.1",
  "url": "<http://httpbin.example.com/delay/3>"
}
```

从请求的结果可以发现，该 Virtual Service 已经生效。通过 Istio 的相关配置，访问 Envoy 的请求被成功转发给了后端 httpbin 服务。

如果从 API 网关的本质来看，现在本节已经将 API 网关与 Isito 对接的最核心内容介绍完毕了。API 网关的基础就是请求转发和路由，其他所有衍生出来的特性如限流、鉴权、黑白名单等等都只是在路由的基础之上扩展并通过更加灵活的配置来控制而已。

Virtual Service 本身提供了更多了配置项来对路由做更进一步的配置，比如重试、比如错误注入，都已经算是功能层面的扩展，无碍于网关整体的设计，算是细枝末节。当读者认为普通的路由不能满足需求之时可以再去阅读[社区文档](https://istio.io/latest/docs/reference/config/networking/virtual-service/#VirtualService)来对各个字段做深入了解。

## 一个服务

前文已经介绍了如何使用 Gateway 和 Virtual Service 来创建网关监听器以及路由，并且成功实现了一个简单的客户端请求转发。但实际上，仍存在最后一个问题，Envoy 要实现完整的路由转发，必须了解真实后端服务。但是 Gateway 仅仅用于控制监听器，Virtual Service 只包含请求匹配条件以及目标服务的名称。Envoy 数据面是从何处获取了后端服务地址等关键信息的呢？答案仍旧是 Istio。作为网关控制面，几乎所有的 Envoy 配置都由 Istio 通过 xDS 协议下发。Istio 默认会发现 Kubernertes 集群下的所有服务（当然，Istio 自然也可以对接其他服务注册中心来做服务发现，但是那已经超出了本节的内容范畴，所以不做赘述）并下发给 Envoy。Envoy 则根据配置将真实后端服务全部抽象为集群。Envoy 提供了 `/stats` 管理接口用于获取内部监控指标数据。可以通过该接口查看 Enovy 中当前活跃的集群数：

```bash
root@$POD_NAME_OF_INGRESS:/# curl localhost:15000/stats -s | grep cluster_manager.active_clusters
cluster_manager.active_clusters: 60
```

可以发现，当前获取集群数为 60 个。现在，为了测试 Istio 的服务发现能力，可以创建一个新的 Kubernetes 服务。简单起见，可以使用如下命令继续基于 Httpbin 负载创建一个新的服务：

```bash
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  labels:
    app: httpbin
  name: httpbin-v2
  namespace: default
spec:
  ports:
  - name: http
    port: 8000
    protocol: TCP
    targetPort: 80
  selector:
    app: httpbin
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
EOF
$ kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
httpbin      ClusterIP   10.106.96.163   <none>        8000/TCP   7d
httpbin-v2   ClusterIP   10.96.83.22     <none>        8000/TCP   24s
```

创建新的服务之后，可以再次通过 `/stats` 接口来查看活跃集群数，如果集群数增加，说明 Istio 成功发现了新的服务并将配置下发到了 Envoy（默认情况下，针对每一个发现的服务，Istio 默认会下发一个 TLS 集群配置和一个普通集群配置，所以服务数和最终 Envoy 创建的集群数不是一一对应）。

```bash
root@$POD_NAME_OF_INGRESS:/# curl localhost:15000/stats -s | grep cluster_manager.active_clusters
cluster_manager.active_clusters: 62
```

借助 Istio 的服务发现，Envoy 网关大部分时候都不需要分心去关注后端服务。但是，假如开发人员确实需要对后端服务做一些调整呢，比如同一个服务在集群中同时存在多个负载版本，希望区分开来；又或者针对特定的服务，希望才需特定的负载均衡策略；又或者希望针对后端服务能够自动进行健康检查等等。为了能够应用上述与服务相关的流量治理策略，Istio 提供了名为 Destination Rule 的资源。下面是一个 Destination Rule 示例：

```bash
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin-dr
spec:
  host: httpbin
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
  subsets:
  - name: test-version
    labels:
      version: v3
    trafficPolicy:
      loadBalancer:
        simple: RANDOM
EOF
```

在示例之中，host 字段自然是服务名称，而 trafficPolicy 字段则包含负载均衡策略、服务熔断策略、连接池等服务流量相关的配置。subsets 则可以将服务下的所有节点（Pod）根据标签值不同划分为多个子组，每个子组都具有独立的流量管理配置。Envoy 最终会为每个子组都创建一个对应的集群。再次调用 `/stats` 接口，由于上述的 Destination Rule 为 httpbin 服务添加了一个 subset，所以 Envoy 为该 subset 创建了新的集群。

```bash
root@$POD_NAME_OF_INGRESS:/# curl localhost:15000/stats -s | grep cluster_manager.active_clusters
cluster_manager.active_clusters: 64
```

现在，假设需要使用新的 subset 来对外暴露服务，应该怎么做呢？很简单，只需要在 Virtual Service 的目标服务处添加 subset 名称即可。

```bash
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin-subset
spec:
  hosts:
  - "httpbin.example.com"
  gateways:
  - httpbin-gateway
  http:
  - match:
    - uri:
        prefix: /anything
    route:
    - destination:
        port:
          number: 8000
        host: httpbin
        subset: test-version
EOF
```

在创建了新的 Virtual Service 之后，再次访问对应的路由：

```bash
$ curl $SERVICE_IP_OF_INGRESS/anything -H"host:httpbin.example.com" -v
*   Trying 10.111.17.154:80...
* TCP_NODELAY set
* Connected to 10.111.17.154 (10.111.17.154) port 80 (#0)
> GET /anything HTTP/1.1
> Host:httpbin.example.com
> User-Agent: curl/7.65.3
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 503 Service Unavailable
< content-length: 19
< content-type: text/plain
< date: Wed, 08 Jul 2020 13:36:00 GMT
< server: istio-envoy
<
* Connection #0 to host 10.111.17.154 left intact
no healthy upstream
```

初次访问新路由的结果并不符合预期，Envoy 网关响应 503 状态码，并告知无可用健康上游服务。回顾本节内容，是否是哪里存在疏忽呢？可以看到，该响应是由 Envoy 生成，说明 Envoy 已经接受到了请求，但是无法将转发给后端服务。再检查 Destination Rule 以及 Virtual Service 的配置，可以发现 Virtual Service 中目标 subset 要求后端节点必须具有  `version: v3` 的标签值。但是现在集群之中并不存在符合要求的节点，自然无法完成请求转发。通过以下命令为一个 Httpbin 负载节点添加相关标签：

```bash
kubectl label pod $POD_NAME_OF_HTTPBIN version=v3 --overwrite=true
```

再次访问同一条路由，此时已经可以得到正确的响应：

```bash
$ curl $SERVICE_IP_OF_INGRESS/anything -H"host:httpbin.example.com"
{
  "args": {},
  "data": "",
  "files": {},
  "form": {},
  "headers": {
    "Accept": "*/*",
		# .....
  },
  "json": null,
  "method": "GET",
  "origin": "10.244.0.1",
  "url": "<http://httpbin.example.com/anything>"
}
```

本小节简单介绍了 Istio 中对服务的抽象以及  Destination Rule 的简单实践。当然，和 Virtual Service 一样，更多丰富的配置项可以直接阅读[社区文档](https://istio.io/latest/docs/reference/config/networking/destination-rule/)。本书限于篇幅，不能一一详述。

## 小结

本节介绍的是 Istio 对接 Envoy API 网关的一些基础内容以及 Istio 中 Gateway、Virtual Service、Destination Rule 三种配置资源与 Envoy 中监听器、路由和集群的映射关系。需要特别说明的是，Envoy 本身并不能算作是完整的 API 网关。Envoy 只是一个 L4/L7 网络代理，作为 Istio 默认网关的数据面，必须要依赖 Istio 才能实现完整的网关功能。在本节的示例当中，网关和服务网格共享了同一套 Istio 控制面。

由于服务网格的无侵入，API 网关可以不感知 Istio 并且与 Istio 完全独立。如 Gloo、Ambassador 等基于 Envoy 的其他 API 网关同样可以和 Istio 有很好的配合。有兴趣的读者可以做更进一步的了解。

## 参考资料：

* [Istio Gateway](https://istio.io/latest/docs/reference/config/networking/gateway/)
* [Istio Virtual Service](https://istio.io/latest/docs/reference/config/networking/virtual-service/)
* [Istio Destination Rule](https://istio.io/latest/docs/reference/config/networking/destination-rule/)