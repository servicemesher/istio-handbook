---
authors: ["sunzhaochang","ycliu912"]
reviewers: ["rootsongjc","GuangmingLuo","ikingye","tony-Ma-yami","malphi",“wangfakang”]
---

# 路由

Istio 的流量路由规则可以让您很容易的控制服务之间的流量和 API 调用。Istio 在服务层面提供了断路器，超时，重试等功能，通过这些功能可以简单地实现 A/B 测试，金丝雀发布，基于百分比的流量分割等，此外还提供了开箱即用的故障恢复功能，用于增加应用的健壮性，以应对服务故障或网络故障。这些功能都可以通过 Istio 的流量管理 API 添加流量配置来实现。

跟其他 Istio 配置一样，流量管理 API 也使用 CRD 指定。本小节主要介绍下面几个典型的流量管理 API 资源，以及这些 API 的功能和使用示例。

## VirtualService

VirtualService (虚拟服务) 在增强 Istio 流量管理的灵活性和有效性方面，发挥着至关重要的作用。[VirtualService](/docs/reference/config/networking/virtual-service/#VirtualService) 由一组路由规则组成，用于对服务实体（在 Kubernetes 中对应为 Pod）进行寻址。如果有流量命中了某条路由规则，就会将其发送到对应的服务或者服务的一个版本/子集。

VirtualService 描述了用户可寻址目标到网格内实际工作负载之间的映射。可寻址的目标服务使用 `hosts` 字段来指定，而网格内的实际负载由每个 `route` 配置项中的 `destination` 字段指定，您将在本节的示例中看到详细的配置说明。

### 功能

VirtualService 通过对客户端请求的目标地址与真实响应请求的目标工作负载进行解耦来实现。VirtualService 同时提供了丰富的配置方式，为发送至这些工作负载的流量指定不同的路由规则。

如果没有 VirtualService，Envoy 会以轮询的方式在所有的服务实例中分发请求。您可以根据具体的工作负载改进这种行为。例如，有些工作负载代表不同的版本。这在 A/B 测试场景中可能有用，您可以根据不同版本的比重来配置路由，或者将来自内部用户的流量定向到一组特定的实例。

使用 VirtualService，您可以为一个或多个主机名指定流量行为。在 VirtualService 中使用路由规则，告诉 Envoy 如何发送 VirtualService 的流量到适当的目标。路由目标可以是相同服务的不同版本，或者是完全不同的服务。

一个典型的应用场景是将流量发送到被指定为服务子集的服务的不同版本。客户端将 VirtualService 视为一个单一实体，将请求发送至 VirtualService 主机，然后 Envoy 根据 VirtualService 规则把流量路由到不同的版本中。例如， “20% 的调用转到新的版本” 或者 “这些用户的请求转到 `v2` 中”。这样，就可以方便您创建一种金丝雀的发布策略实现新版本流量的平滑比重升级。流量路由完全独立于实例部署，这意味着实现新版本服务的实例可以根据流量的负载来伸缩，完全不影响流量路由。相比之下，类似 Kubernetes 的容器调度平台仅支持基于部署中实例扩缩容比重的流量分发，那样会日趋复杂化。您可以在[用 Istio 实现金丝雀部署](https://istio.io/zh/blog/2017/0.1-canary/)中获取更多的 VirtualService 关于金丝雀部署的帮助信息。

VirtualService 也提供了如下功能。

- 通过单个 VirtualService 处理多个应用程序服务。例如，如果您的服务网格使用是 Kubernetes，您可以配置一个 VirtualService 来处理一个特定命名空间的所有服务。将单一的 VirtualService 映射为多个“真实”的服务特别有用，可以在不需要客户适应转换的情况下，将单体应用转换为微服务构建的复合应用系统。您的路由规则可以指定“请求到  `monolith.com` 的 URLs 跳转至 `microservice A` 中”。

- 和 [`Gateway`](https://istio.io/zh/docs/concepts/traffic-management/#gateways) (网关) 一起配置流量规则来控制入口和出口流量。

在一些应用场景中，由于指定服务子集，需要配置 DestinationRule (目标规则) 来使用这些功能。在不同的对象中指定服务子集以及其他特定的目标策略可以帮助您在不同的 VirtualService 中清晰地复用这些功能。

VirtualService 通过解耦客户端请求的目标地址和真实响应请求的目标工作负载为服务提供了合适的统一抽象层，而由此演化设计的配置模型为管理这方面提供了一致的环境。您将在下面示例中看到一些典型的配置场景。

### 示例

下面的 VirtualService 根据是否来自于特定用户路由请求到不同的服务版本中（如果请求来自用户 `jason` ，则访问 `v2` 版本的 `reviews`，否则访问 `v3` 版本）。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v3
```

#### hosts 字段

使用 `hosts` 字段列举 VirtualService 的主机--即用户指定的目标或者路由规则设定的目标。这是客户端向服务发送请求时使用的一个或者多个地址。

```yaml
hosts:
- reviews
```

VirtualService 主机名可以是 IP 地址、 DNS 域名，或者依赖于平台的一个简称（例如 Kubernetes 服务的短名称），隐式或者显式地指向一个完全限定域名（FQDN)。您也可以使用通配符（”*“）前缀，让您创建一组匹配所有服务的路由规则。VirtualService 的 `hosts` 实际上不必是 Istio 服务注册的一部分，它只是虚拟的目标地址。这让您可以为没有路由到网格内部的虚拟主机建模。

#### 路由规则

`http` 字段包含了 VirtualService 的路由规则，描述匹配条件和根据所指定的 `hosts` 字段通过 HTTP/1.1， HTTP2， 以及 gRPC 路由目标地址（您也可以使用 `tcp` 和 `tls` 片段为 [TCP](https://istio.io/zh/docs/reference/config/networking/virtual-service/#TCPRoute) 和未终止的 [TLS](https://istio.io/zh/docs/reference/config/networking/virtual-service/#TLSRoute) 流量设置路由规则）。根据您使用的场景，一条路由规则包含零个或者多个匹配条件的目标地址。

##### 匹配条件

示例中第一条路由规则有一个以 `match` 字段开头的条件。在这种场景中，您想应用这条所有从用户 “jason” 的所有请求， 所以您可以使用 `headers`，`end-user`  和 `exec` 字段去匹配符合条件的请求。

```yaml
- match:
   - headers:
       end-user:
         exact: jason
```

##### Destination

路由片段的 `destination` 字段指定符合匹配条件的流量目标地址。这里不像 VirtualService 的 `hosts`，Destination 的 `host` 必须是存在于 Istio 服务注册中心的实际目标地址，否则 Envoy 不知道该将请求发送到哪里。这个目标地址可以是代理的网格服务或者作为服务入口加入的非网格服务。下面的场景中我们运行在 Kubernetes 平台上，主机名是 Kubernetes 的服务名。

```yaml
route:
- destination:
    host: reviews
    subset: v2
```

在这些示例中需要注意的是，为了简化我们使用 Kubernetes 的短名字作为目标主机名字。当这条路由规则生效的时候，Istio 会基于包含路由规则的 VirtualService 命名空间添加域名前缀去获取完全限定域名作为主机名字。在本文的示例中使用短命字也意味着您可以拷贝并在任何命名空间中尝试它。

> 注：像这样使用短命字只有目标主机和 VirtualService 在同一个 Kubernetes 的命名空间中起作用。由于使用 Kubernetes 短名字易导致配置错误，所以推荐您在生产环境中使用完全限定域名。`destination` 片段也指定了 Kubernetes 服务的子集，将匹配此规则条件的请求转入其中。在此示例中子集的名字为 `v2`。在下一节的 DestinationRule 中您会看到怎样定义一个服务子集。

#### 路由规则优先级

路由规则按从上到下的顺序选择，在您服务中定义的第一条规则具有最高优先级。在本例中，不满足第一条路由规则的流量均流向一个默认的目标。因此，第二条规则没有匹配条件就直接将流量导向到 `v3` 子集中。

```yaml
- route:
  - destination:
      host: reviews
      subset: v3
```

我们推荐在每个 VirtualService 中配置一条默认“无条件的”或者基于权重的规则以确保 VirtualService 至少有一条匹配的路由。

#### 路由规则的更多内容

路由规则是将特定流量子集路由到特定目标地址的强大工具。您可以在流量端口、`header` 字段、 URL 等内容上设置匹配条件。例如，下面的VirtualService 使用户发送流量到两个独立的服务，ratings and reviews， 就好像它们是 `http://bookinfo.com/` 这个更大的 VirtualService 的一部分。VirtualService 规则根据请求的 URL 和指向适当服务的请求匹配流量。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
    - bookinfo.com
  http:
  - match:
    - uri:
        prefix: /reviews
    route:
    - destination:
        host: reviews
  - match:
    - uri:
        prefix: /ratings
    route:
    - destination:
        host: ratings
```

对于匹配条件，您可以使用确定的值，一条前缀、或者一条正则表达式。

您可以使用 `AND` 向同一个 `match` 块添加多个匹配条件， 或者使用 `OR` 向同一个规则添加多个 `match` 块。对于任意给定的 VirtualService ，您可以配置多条路由规则。这可以使您的路由条件在一个单独的 VirtualService 中基于业务场景的复杂度来进行相应的配置。可以在 [HTTPMatchRequest 参考](https://istio.io/docs/reference/config/networking/virtual-service/#HTTPMatchRequest)中查看匹配条件字段和他们可能的值。

再者进一步使用匹配条件，您可以使用基于“权重”百分比分发流量。这在 A/B 测试和金丝雀部署中非常有用。

```yaml
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 75
    - destination:
        host: reviews
        subset: v2
      weight: 25
```

您也可以使用路由规则在流量上执行一些操作，例如

- 扩展或者删除 `headers`
- 重写 URL
- 为调用这个目标地址设置[重试策略](https://istio.io/docs/concepts/traffic-management/#retries)

## DestinationRule

DestinationRule 是 Istio 流量路由功能的重要组成部分。一个 VirtualService 可以看作是如何将流量分发到给定的目标地址，然后调用 DestinationRule 来配置分发到该目标地址的流量。DestinationRule 在 VirtualService 的路由规则之后起作用（即在 VirtualService 的 `match` -> `route` -> `destination` 之后起作用，此时流量已经分发到真实的 service 上），应用于真实的目标地址。

特别地，可以使用 DestinationRule 来指定命名的服务子集，例如根据版本对服务的实例进行分组，然后通过 VirtualService 的路由规则中的服务子集将控制流量分发到不同服务的实例中。

DestinationRule 允许在调用完整的目标服务或特定的服务子集，如倾向使用的负载均衡模型，TLS 安全模型或断路器时自定义 Envoy 流量策略。

Istio 默认会使用轮询策略，此外 Istio 也支持如下负载均衡模型，可以在 DestinationRule 中使用这些模型，将请求分发到特定的服务或服务子集。

- Random：将请求转发到一个随机的实例上
- Weighted：按照指定的百分比将请求转发到实例上
- Least requests：将请求转发到具有最少请求数目的实例上

# DestinationRule 示例

下面的 DestinationRule 使用不同的负载均衡策略为 my-svc 目的服务配置了3个不同的子集（subset）。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-destination-rule
spec:
  host: my-svc
  trafficPolicy:     #默认的负载均衡策略模型为随机
    loadBalancer:
      simple: RANDOM
  subsets:
  - name: v1  #subset1，将流量转发到具有标签 version:v1 的 deployment 对应的服务上
    labels:
      version: v1
  - name: v2  #subset2，将流量转发到具有标签 version:v2 的 deployment 对应的服务上，指定负载均衡为轮询
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
  - name: v3   #subset3，将流量转发到具有标签 version:v3 的 deployment 对应的服务上
    labels:
      version: v3
```

每个子集由一个或多个 `labels` 定义，对应 Kubernetes 中的对象（如 pod）的 key/value 对。这些标签定义在 Kubernetes 服务的 deployment 的 metadata 中，用于标识不同的版本。

除了定义子集外，DestinationRule 还定义了该目的地中所有子集的默认流量策略，以及仅覆盖该子集的特定策略。默认的策略定义在 `subset` 字段之上，为 `v1` 和 `v3` 子集设置了随机负载均衡策略，在 `v2` 策略中使用了轮询负载均衡。

## Gateway

Gateway 用于管理进出网格的流量，指定可以进入或离开网格的流量。Gateway 配置应用于网格边缘的独立的 Envoy 代理上，而不是服务负载的 Envoy 代理上。

与其他控制进入系统的流量的机制(如 Kubernetes ingress API)不同，Istio gateway 允许利用 Istio 的流量路由的强大功能和灵活性。Istio 的 gateway 资源仅允许配置 4-6 层的负载属性，如暴露的端口，TLS 配置等等，但结合 Istio 的 Virtual service，就可以像管理 Istio 网格中的其他数据面流量一样管理 Gateway 的流量。

Gateway 主要用于管理 Ingress 流量，但也可以配置 egress gateway。通过 egress gateway 可以配置流量离开网格的特定节点，限制哪些服务可以访问外部网络，或通过 egress 安全控制来提高网格的安全性。gateway 可以用于配置为一个纯粹的内部代理。

istio (通过 `istio-ingressgateway` 和 `istio-egressgateway` 参数)提供了一些预配置的 Gateway 代理，`default` profile 下仅会部署 ingress gateway。Gateway 可以通过部署文件进行部署，也可以单独部署。

下面是 `default` profile 默认安装的 ingress

```bash
$ kubectl get gateways.networking.istio.io -n istio-system
NAME                   AGE
istio-ingressgateway   4d20h
```

可以看到该 ingress 就是一个普通的 pod，该 pod 仅包含一个 istio-proxy 容器

```bash
$ kubectl get pod -n istio-system |grep ingress
istio-ingressgateway-64f6f9d5c6-qrnw2 1/1 Running 0 4d20h
```

下面是一个 Gateway 的例子，用于配置外部 HTTPS 的 ingress 流量：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: ext-host-gwy
spec:
  selector:              #指定 gateway 配置下发的代理，如具有标签 app: my-gateway-controller 的 pod
    app: my-gateway-controller
  servers:
  - port:                #gateway pod 暴露的端口信息
      number: 443
      name: https
      protocol: HTTPS
    hosts:                #外部流量
    - ext-host.example.com
    tls:
      mode: SIMPLE
      serverCertificate: /tmp/tls.crt
      privateKey: /tmp/tls.key
```
上述 Gateway 配置允许来自 `ext-host.example.com` 流量进入网格的 443 端口，但没有指定该流量的路由。(此时流量只能进入网格，但没有指定处理该流量的服务，因此需要与 Virtual service 进行绑定)

为了为 Gateway 指定路由，需要通过 Virtual service 的 `gateway` 字段，将 Gateway 绑定到一个 Virtual service 上，将来自 `ext-host.example.com` 流量引入一个 VirtualService，`hosts` 可以是通配符，表示引入匹配到的流量。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: virtual-svc
spec:
  hosts:
  - ext-host.example.com
  gateways:        #将 gateway "ext-host-gwy" 绑定到 virtual service "virtual-svc"上
  - ext-host-gwy
```

## ServiceEntry

Istio 支持对接 Kubernetes、Consul 等多种不同的注册中心，控制平面 Pilot 启动时，会从指定的注册中心获取 Service Mesh 集群的服务信息和实例列表，并将这些信息进行处理和转换，然后通过 xDS 下发给对应的数据平面，保证服务之间可以互相发现并正常访问。

同时，由于这些服务和实例信息都来源于服务网格内部，Istio 无法从注册中心直接获取网格外的服务，导致不利于网格内部与外部服务之间的通信和流量管理。为此，Istio 引入 ServiceEntry 实现对外通信和管理。

使用 ServiceEntry 可以将外部的服务条目添加到 Istio 内部的服务注册表中，以便让网格中的服务能够访问并路由到这些手动指定的服务。ServiceEntry 描述了服务的属性（DNS 名称、VIP、端口、协议、端点）。这些服务可能是位于网格外部（如，web APIs），也可能是处于网格内部但不属于平台服务注册表中的条目（如，需要和 Kubernetes 服务交互的一组虚拟机服务）。

### ServiceEntry 示例和属性介绍

对于网格外部的服务，下面的 ServiceEntry 示例表示网格内部的应用通过 https 访问外部的 API。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: google
spec:
  hosts:
  - www.google.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL
```

对于在网格内部但不属于平台服务注册表的服务，使用下面的示例可以将一组在非托管 VM 上运行的 MongoDB 实例添加到 Istio 的注册中心，以便可以将这些服务视为网格中的任何其他服务。
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc-mongocluster
spec:
  hosts:
  - mymongodb.somedomain
  addresses:
  - 192.192.192.192/24 # VIPs
  ports:
  - number: 27018
    name: mongodb
    protocol: MONGO
  location: MESH_INTERNAL
  resolution: STATIC
  endpoints:
  - address: 2.2.2.2
  - address: 3.3.3.3
```

结合上面给出的示例，这里对 ServiceEntry 涉及的关键属性解释如下：

* `hosts`: 表示与该 ServiceEntry 相关的主机名，可以是带有通配符前缀的 DNS 名称。
* `addresses`: 与服务相关的虚拟 IP 地址，可以是 CIDR 前缀的形式。
* `ports`: 和外部服务相关的端口，如果外部服务的 endpoints 是 Unix socket 地址，这里必须只有一个端口。
* `location`: 用于指定该服务属于网格内部（MESH_INTERNAL）还是外部（MESH_EXTERNAL）。
* `resolution`: 主机的服务发现模式，可以是 NONE、STATIC、DNS。
* `endpoints`: 与服务相关的一个或多个端点。
* `exportTo`: 用于控制 ServiceEntry 跨命名空间的可见性，这样就可以控制在一个命名空间下定义的资源对象是否可以被其他命名空间下的 Sidecar、Gateway 和 VirtualService 使用。目前支持两种选项，"." 表示仅应用到当前命名空间，"*" 表示应用到所有命名空间。

### 使用 ServiceEntry 访问外部服务

Istio 提供了三种访问外部服务的方法：

1. 允许 sidecar 将请求传递到未在网格内配置过的任何外部服务。使用这种方法时，无法监控对外部服务的访问，也不能利用 Istio 的流量控制功能。
2. 配置 ServiceEntry 以提供对外部服务的受控访问。这是 Istio 官方推荐使用的方法。
3. 对于特定范围的 IP，完全绕过 sidecar。仅当出于性能或其他原因无法使用 sidecar 配置外部访问时，才建议使用该配置方法。

这里，我们重点讨论第 2 种方式，也就是使用 ServiceEntry 完成对网格外部服务的受控访问。

对于 sidecar 对外部服务的处理方式，istio 提供了两种选项:

* `ALLOW_ANY`：默认值，表示 Istio 代理允许调用未知的外部服务。上面的第一种方法就使用了该配置项。
* `REGISTRY_ONLY`：Istio 代理会阻止任何没有在网格中定义的 HTTP 服务或 ServiceEntry 的主机。

可以使用下面的命令查看当前所使用的模式:
```bash
$ kubectl get configmap istio -n istio-system -o yaml | grep -o "mode: ALLOW_ANY"
mode: ALLOW_ANY
```
如果当前使用的是 `ALLOW_ANY` 模式，可以使用下面的命令切换为 `REGISTRY_ONLY` 模式:
```bash
$ kubectl get configmap istio -n istio-system -o yaml | sed 's/mode: ALLOW_ANY/mode: REGISTRY_ONLY/g' | kubectl replace -n istio-system -f -
configmap "istio" replaced
```

在 `REGISTRY_ONLY` 模式下，需要使用 ServiceEntry 才能完成对外部服务的访问。当创建如下的 ServiceEntry 时，服务网格内部的应用就可以正常访问 httpbin.org 服务了。
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: httpbin-ext
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
  location: MESH_EXTERNAL
```

### 管理外部流量

使用 ServiceEntry 可以使网格内部服务发现并访问外部服务，除此之外，还可以对这些到外部服务的流量进行管理。结合 VirtualService 为对应的 ServiceEntry 配置外部服务访问规则，如请求超时、故障注入等，实现对指定服务的受控访问。

下面的示例就是为外部服务 httpbin.org 设置了超时时间，当请求时间超过 3s 时，请求就会直接中断，避免因外部服务访问时延过高而影响内部服务的正常运行。由于外部服务的稳定性通常无法管控和监测，这种超时机制对内部服务的正常运行具有重要意义。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin-ext
spec:
  hosts:
  - httpbin.org
  http:
  - timeout: 3s
    route:
      - destination:
          host: httpbin.org
        weight: 100
```

同样的，我们也可以为 ServiceEntry 设置故障注入规则，为系统测试提供基础。下面的示例表示为所有访问 `httpbin.org` 服务的请求注入一个403错误。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
 name: httpbin-service
spec:
 hosts:
 - httpbin.org
 http:
 - route:
   - destination:
       host: httpbin.org
   fault:
     abort:
       percent: 100
       httpStatus: 403
```

### 小结

本小节主要介绍了 VirtualService、DestinationRule、Gateway、 ServiceEntry 的概念、功能和一些典型的配置示例。VirtualServce 和 DestinationRule 都是 Istio 流量路由功能的关键部分。Gateway 是用于控制南北流量的网关，将 VirtualService 绑定到 Gateway 上，可以控制进入的 HTTP/TCP 流量。通过使用 ServiceEntry，可以使网格内部的服务正常发现和路由到外部服务，并在此基础上，结合 VirtualService 实现请求超时、故障注入等功能。限于篇幅，本小节只是做了简要地说明，您可以在小结末尾的参考中查看更多的说明信息和完整的配置示例。


### 参考

- [istio.io / Concepts / Traffic Management](https://istio.io/docs/concepts/traffic-management/)
- [用 Istio 实现金丝雀部署](https://istio.io/zh/blog/2017/0.1-canary/)
- [HTTPMatchRequest 参考](https://istio.io/docs/reference/config/networking/virtual-service/#HTTPMatchRequest)
