---
authors: ["ycliu912"]
reviewers: [""]
---

# VirtualService

## 虚拟服务 {#virtual-services}

[虚拟服务（Virtual Service)](/docs/reference/config/networking/virtual-service/#VirtualService) 和[目标规则（Distination rules）](#distination-rules)一样，都是 Istio 流量管理的关键功能模块。一个虚拟服务可以使您的配置请求在 Istio 服务网格中怎样路由至一个具体的服务，从而构建基础连接以被您的 Istio 和平台发现。每一个虚拟服务包含一系列预置有序路由集合，使 Istio 在网格中匹配每一个至虚拟服务特定真实目标的给定请求。根据使用的具体场景，您的网格需要配置零个或者多个虚拟服务。

### 为什么使用虚拟服务

虚拟服务在使 Istio 流量管理灵活强大中扮演着关键角色。它们通过在客户端从实际实现它们的目标工作负载发送请求时进行强解耦来实现。虚拟服务也提供了丰富的指向这些工作负载的路由集合。

为什么会如此有用呢？没有虚拟服务，Envoy 以轮询的方式在所有的后端服务实例分发路由信息，就像介绍中描述的一样。您可以根据具体的工作负载改进这种方式。例如，有些工作负载代表不同的版本。这在 A/B 测试场景中非常有用，您可以依据不同的版本配属对应的比重实现路由分发，或者以您一个内部用户访问特定的实例集合。

在一个虚拟服务中，您可以为一个或者多个主机（hostnames）配置特定路由行为。您可以用此虚拟服务的路由规则来告诉 Envoy 怎样发送虚拟服务路由到合适的工作目标中。路由目标可以是相同服务的不同版本，或者是所有不同的服务。

一个典型的应用场景是路由至同一服务的不同版本，作为特定子服务集合。客户端发送请求至虚拟服务主机作为单一入口，此时 Envoy 会依据所配置虚拟服务规则路由到不同的版本中：例如， “20% 的请求至新的版本” 或者 “这些用户的请求到 `v2` 中”。这样，就可以方便您创建一种金丝雀的回滚策略实现路由新版本的平滑比重升级。此平滑比重升级的策略是完全独立于容器实例扩缩容部署过程的，即路由新版本的路由比重与实例部署过程中扩缩容的数量比重是完全解耦的。相比之下，类似 Kubernetes 的容器调度平台仅支持基于部署中实例扩缩容比重的路由策略，那样会日趋复杂化。您可以在[用 Istio 实现金丝雀部署](https://istio.io/zh/blog/2017/0.1-canary/) 中获取更多的虚拟服务关于金丝雀部署的帮助信息。
        
虚拟服务也可以帮助您：

- 通过单一虚拟服务实现多地址的应用服务。例如，如果您的服务网格使用是 Kubernetes，您可以配置一个虚拟服务来处理一个特定命名空间的所有服务。将单一的虚拟服务映射为多个“真实”的服务特别有用在将一个巨大的单体应用重构为一组基于微服务构建的不依赖于服务消费者的复合应用过渡过程中。您的路由规则可以指定“请求到  `monolith.com` 的 URLs 转发至 `microservice A` 中”，诸如此类。您可以参考 我们的下面一个示例 来了解实现过程。
- 和[`Gateway`](https://istio.io/zh/docs/concepts/traffic-management/#gateways)一起配置路由规则来控制入口和出口流量。

在一些应用场景中，由于指定服务子集，需要配置目标规则来使用这些功能。在不同的对象中指定服务子集合与其他特定的目标策略可以使您在不同的虚拟服务中清晰的复用这些功能。您可以在下面的部分中发现更多的目标规则配置。

### 虚拟服务示例

下面的虚拟服务根据是否来自于特定用户路由请求到不同的服务版本中。

```
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

### hosts字段

`hosts` 字段列举虚拟服务主机，也就是说，用户可达的目标地址或者这些路由规则应用的一组地址。这些地址是客户端发送请求服务所使用的地址。

```
hosts:
- reviews
```
虚拟服务主机名可以使一个 IP 地址，一个 DNS 域名，或者依赖于平台的一个简称（例如 Kubernetes 服务的短名称），隐式或者显示地指向一个完全限定域名（FQDN)。您也可以使用通配符（*）前缀，一条路由规则匹配所有的服务。虚拟服务主机不是 Istio 服务注册的一部分，它们仅仅是虚拟目标地址。这可以使没有路由至网格内部服务的虚拟主机模块化。

### 路由规则

`http` 字段包含了虚拟服务的路由规则，描述匹配条件和根据所指定的 `hosts` 字段通过 HTTP/1.1， HTTP2， 以及 gRPC 路由目标地址（您也可以使用 `tcp` 和 `tls` 片段为 [TCP](https://istio.io/zh/docs/reference/config/networking/virtual-service/#TCPRoute) 和 未终止的 [TLS](https://istio.io/zh/docs/reference/config/networking/virtual-service/#TLSRoute) 流量设置路由规则）。根据您使用的场景，一条路由规则包含零个或者多个匹配条件的目标地址。

#### 匹配条件

示例中第一条路由规则有一个以 `match` 字段开头的条件。在这种场景中，您想应用这条所有从用户 “jason” 的所有请求， 所以您可以使用 `headers`，`end-user`  和 `exec` 字段去匹配符合条件的请求。

```
- match:
   - headers:
       end-user:
         exact: jason
```

#### 目标地址

路由片段的 `distination` 字段指定符合匹配条件的流量目标地址。这里不像虚拟服务的主机，目标地址的主机必须是在 Istio 服务注册中真实存在的目标地址或者 Envoy 也不知道往哪里发送流量给它。这个目标地址可以是代理的网格服务或者作为服务入口加入的非网格服务。下面的场景中我们运行在 Kubernetes 平台上，主机名是 Kubernetes 的服务名：

```
route:
- destination:
    host: reviews
    subset: v2
```

在这些示例中需要注意的是，为了简化我们使用 Kubernetes 的短名字作为目标主机名字。当这条路由规则生效的时候，Istio 会基于包含路由规则的虚拟服务命名空间添加域名前缀去获取完全限定域名作为主机名字。在我们的示例中使用短命字也意味着您可以拷贝并在任何命名空间中尝试它。

注：像这样使用短命字只有目标主机和虚拟服务在同一个 Kubernetes 的命名空间中起作用。由于使用 Kubernetes 短名字易导致配置错误，所以我们推荐在生产环境中使用完全限定域名。`destination` 片段也指定了 Kubernetes 服务的子集，将匹配此规则条件的请求转入其中。在此示例中子集的名字为 `v2`。在下面的目标规则中您会看到怎样定义一个服务子集。

#### 路由规则优先级

路由规则按从上到下的顺序选择，在您服务中定义的第一条规则具有最高优先级。在本示例中没有匹配第一条路由规则都会转入由第二条定义的默认目标规则中。因此，第二条规则没有匹配条件就直接将流量导向到 `v3` 子集中。

```
- route:
  - destination:
      host: reviews
      subset: v3
```

我们推荐在每个虚拟服务中配置一条默认“无条件的”或者基于权重的规则（如下文）以确保虚拟服务至少有一条匹配的路由。

### 路由规则的更多内容

如前文所见，路由规则是一个路由特定子集的流量到特定目标地址的强大工具。您可以在流量端口、`header` 字段、 URL 等内容上设置匹配条件。例如，下面的虚拟服务使用户发送流量到两个不同的服务，ratings and reviews， 它们作为更大的虚拟服务 `http://bookinfo.com/` 的一部分。这个虚拟服务匹配基于青请求 URLs 的流量并指向合适的服务中。

```
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

您可以使用 `AND` 向同一个 `match` 块添加多几个匹配条件， 或者使用 `OR` 向同一个规则添加多个 `match` 块。对于任意给定的虚拟服务，您可以配置多条路由规则。这可以使您的路由条件在一个单独的虚拟服务中基于业务场景的复杂度来进行相应的配置。可以在 [HTTPMatchRequest 参考](https://istio.io/docs/reference/config/networking/virtual-service/#HTTPMatchRequest) 查看完整的查看匹配条件字段和他们可能的值。

再者进一步使用匹配条件，您可以使用基于“权重”百分比分发流量。这在 A/B 测试和金丝雀部署非常有用:

```
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

您也可以使用路由规则在流量上执行一些操作，例如：
- 扩展或者删除 `headers`
- 重写 URL
- 为调用这个目标地址设置[重试策略](https://istio.io/docs/concepts/traffic-management/#retries)
