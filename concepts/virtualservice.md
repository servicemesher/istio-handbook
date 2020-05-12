---
authors: ["ycliu912"]
reviewers: ["ikingye","rootsongjc","tony-Ma-yami"]
---

# VirtualService

VirtualService (虚拟服务) 在增强 Istio 流量管理的灵活性和有效性方面，发挥着至关重要的作用。本小节主要从概念，功能，示例三个方面说明。

## 背景

Istio 一开始确定的抽象模型与对接的底层平台无关，但目前来看基本绑定 Kubernetes，所以为了理解 VirtualService 的概念，先了解 Kubernetes 服务模型是必要的，具体参见 [Istio 中的服务和流量的抽象模型](https://mp.weixin.qq.com/s/-6XiHWj9yE_VkOPsPl4Idw)中的说明。另外，在本文的示例中可以看出 VirtualService 是一套完整的配置模型，您可以通过参考 [Introducing the Istio v1alpha3 routing API ](https://istio.io/blog/2018/v1alpha3-routing/)了解一些动机和设计原则。

## 概念

[VirtualService ](/docs/reference/config/networking/virtual-service/#VirtualService)由一组路由规则组成，用于对服务实体（在 Kubernetes 中对应为 Pod）进行寻址。如果有流量符合其中规则的选择条件，就会发送流量给对应的服务（或者服务的一个版本/子集）。

VirtualService 描述了一个或者多个用户可寻址目标到网格内实际工作负载之间的映射。其中可寻址的目标服务使用 `hosts` 字段来指定，而网格内的实际负载由每个 `route` 配置项中的 `distination` 字段指定，您会在本节的示例中看到详细的配置说明。

## 功能

VirtualService 通过对客户端请求的目标地址与真实响应请求的目标工作负载进行解耦来实现。VirtualService 同时提供了丰富的配置方式，为发送至这些工作负载的流量指定不同的路由规则。

如果没有 VirtualService，Envoy 会以轮询的方式在所有的后端服务实例中分发路由信息。您可以根据具体的工作负载改进这种方式。例如，有些工作负载代表不同的版本。这在 A/B 测试场景中非常有用，您可以依据不同的版本配属对应的比重实现路由分发，或者以您一个内部用户访问特定的实例集合。

使用 VirtualService，您可以为一个或多个主机名指定流量行为。在 VirtualService 中使用路由规则，告诉 Envoy 如何发送 VirtualService 的流量到适当的目标。路由目标可以是相同服务的不同版本，或者是完全不同的服务。

一个典型的应用场景是将流量发送到被指定为服务子集的服务的不同版本。客户端将 VirtualService 视为一个单一实体，将请求发送至 VirtualService 主机，然后 Envoy 根据虚拟服务规则把流量路由到不同的版本中。例如， “20% 的调用转到新的版本” 或者 “这些用户的请求到 `v2` 中”。这样，就可以方便您创建一种金丝雀的发布策略实现新版本流量的平滑比重升级。流量路由完全独立于实例部署，这意味着实现新版本服务的实例可以根据流量的负载来伸缩，完全不影响流量路由。相比之下，类似 Kubernetes 的容器调度平台仅支持基于部署中实例扩缩容比重的流量分发，那样会日趋复杂化。您可以在[用 Istio 实现金丝雀部署](https://istio.io/zh/blog/2017/0.1-canary/)中获取更多的 VirtualService 关于金丝雀部署的帮助信息。

VirtualService 也提供了如下功能。

- 通过单个 VirtualService 处理多个应用程序服务。例如，如果您的服务网格使用是 Kubernetes，您可以配置一个 VirtualService 来处理一个特定命名空间的所有服务。将单一的 VirtualService 映射为多个“真实”的服务特别有用，可以在不需要客户适应转换的情况下，将单体应用转换为微服务构建的复合应用系统。您的路由规则可以指定“请求到  `monolith.com` 的 URLs 跳转至 `microservice A` 中”。

- 和 [`Gateway` ](https://istio.io/zh/docs/concepts/traffic-management/#gateways)(网关) 一起配置流量规则来控制入口和出口流量。

在一些应用场景中，由于指定服务子集，需要配置 DestinationRule (目标规则) 来使用这些功能。在不同的对象中指定服务子集以及其他特定的目标策略可以帮助您在不同的 VirtualService 中清晰地复用这些功能。

## 示例

下面的 VirtualService 根据是否来自于特定用户路由请求到不同的服务版本中。

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

### hosts 字段

使用 `hosts` 字段列举 VirtualService 的主机--即用户指定的目标或者路由规则设定的目标。这是客户端向服务发送请求时使用的一个或者多个地址。

```yaml
hosts:
- reviews
```

VirtualService 主机名可以是一个 IP 地址，一个 DNS 域名，或者依赖于平台的一个简称（例如 Kubernetes 服务的短名称），隐式或者显式地指向一个完全限定域名（FQDN)。您也可以使用通配符（*）前缀，一条路由规则匹配所有的服务。VirtualService 主机不是 Istio 服务注册的一部分，这让您可以为没有路由到网格内部的虚拟主机建模。

### 路由规则

`http` 字段包含了 VirtualService 的路由规则，描述匹配条件和根据所指定的 `hosts` 字段通过 HTTP/1.1， HTTP2， 以及 gRPC 路由目标地址（您也可以使用 `tcp` 和 `tls` 片段为 [TCP ](https://istio.io/zh/docs/reference/config/networking/virtual-service/#TCPRoute)和未终止的 [TLS ](https://istio.io/zh/docs/reference/config/networking/virtual-service/#TLSRoute)流量设置路由规则）。根据您使用的场景，一条路由规则包含零个或者多个匹配条件的目标地址。

#### 匹配条件

示例中第一条路由规则有一个以 `match` 字段开头的条件。在这种场景中，您想应用这条所有从用户 “jason” 的所有请求， 所以您可以使用 `headers`，`end-user`  和 `exec` 字段去匹配符合条件的请求。

```yaml
- match:
   - headers:
       end-user:
         exact: jason
```

#### Destination

路由片段的 `distination` 字段指定符合匹配条件的流量目标地址。这里不像 VirtualService 的 `hosts`，Distination 的 `host` 必须是存在于 Istio 服务注册中心的实际目标地址，否则 Envoy 不知道该将请求发送到哪里。这个目标地址可以是代理的网格服务或者作为服务入口加入的非网格服务。下面的场景中我们运行在 Kubernetes 平台上，主机名是 Kubernetes 的服务名。

```yaml
route:
- destination:
    host: reviews
    subset: v2
```

在这些示例中需要注意的是，为了简化我们使用 Kubernetes 的短名字作为目标主机名字。当这条路由规则生效的时候，Istio 会基于包含路由规则的 VirtualService 命名空间添加域名前缀去获取完全限定域名作为主机名字。在本文的示例中使用短命字也意味着您可以拷贝并在任何命名空间中尝试它。

> 注：像这样使用短命字只有目标主机和 VirtualService 在同一个 Kubernetes 的命名空间中起作用。由于使用 Kubernetes 短名字易导致配置错误，所以推荐您在生产环境中使用完全限定域名。`destination` 片段也指定了 Kubernetes 服务的子集，将匹配此规则条件的请求转入其中。在此示例中子集的名字为 `v2`。在下一节的 DestinationRule 中您会看到怎样定义一个服务子集。

### 路由规则优先级

路由规则按从上到下的顺序选择，在您服务中定义的第一条规则具有最高优先级。在本例中，不满足第一条路由规则的流量均流向一个默认的目标。因此，第二条规则没有匹配条件就直接将流量导向到 `v3` 子集中。

```yaml
- route:
  - destination:
      host: reviews
      subset: v3
```

我们推荐在每个 VirtualService 中配置一条默认“无条件的”或者基于权重的规则以确保 VirtualService 至少有一条匹配的路由。

### 路由规则的更多内容

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

## 小结

本节主要介绍了 VirtualService 的概念、功能和一些典型的配置示例。VirtualServce 和 后面小节介绍的 DestinationRule 都是 Istio 流量路由功能的关键部分。限于篇幅，本节只是做了简要地说明，您可以在小结末尾的参考中查看更多的说明信息和完整的配置示例。

## 参考

- [Istio.io / Concepts / Traffic Management](https://istio.io/docs/concepts/traffic-management/)
- [用 Istio 实现金丝雀部署](https://istio.io/zh/blog/2017/0.1-canary/)
- [HTTPMatchRequest 参考](https://istio.io/docs/reference/config/networking/virtual-service/#HTTPMatchRequest)
- [Introducing the Istio v1alpha3 routing API](https://istio.io/blog/2018/v1alpha3-routing/)
- [Istio中的服务和流量的抽象模型](https://mp.weixin.qq.com/s/-6XiHWj9yE_VkOPsPl4Idw)