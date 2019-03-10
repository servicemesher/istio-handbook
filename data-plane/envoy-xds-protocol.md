---
owners: ["rootsongjc","DavadDi"]
reviewers: [""]
description: "本文是 xDS 协议的解析，译自 Matt Klein 的博客。"
publishDate: 2019-03-10
updateDate: 2019-03-10
tags: ["envoy","concept"]
category: "translation"
---

# xDS 协议解析

> 本文译自 [xDS REST and gRPC protocol](https://github.com/envoyproxy/data-plane-api/blob/master/XDS_PROTOCOL.md)，译者：狄卫华，审校：宋净超

Envoy 通过查询文件或管理服务器来动态发现资源。概括地讲，对应的发现服务及其相应的 API 被称作 _xDS_。Envoy 通过订阅（_subscription_）方式来获取资源，如监控指定路径下的文件、启动 gRPC 流或轮询 REST-JSON URL。后两种方式会发送 [`DiscoveryRequest`](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/discovery.proto#discoveryrequest) 请求消息，发现的对应资源则包含在响应消息 [`DiscoveryResponse`](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/discovery.proto#discoveryresponse) 中。下面，我们将具体讨论每种订阅类型。

## 文件订阅

发现动态资源的最简单方式就是将其保存于文件，并将路径配置在 [ConfigSource](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/core/config_source.proto#core-configsource) 中的 `path` 参数中。Envoy 使用 `inotify`（Mac OS X 上为 `kqueue`）来监控文件的变化，在文件被更新时，Envoy 读取保存的 `DiscoveryResponse` 数据进行解析，数据格式可以为二进制 protobuf、JSON、YAML 和协议文本等。

> 译者注：core.ConfigSource 配置格式如下：

```json
{
  "path": "...",
  "api_config_source": "{...}",
  "ads": "{...}"
}
```

文件订阅方式可提供统计数据和日志信息，但是缺少 ACK/NACK 更新的机制。如果更新的配置被拒绝，xDS API 则继续使用最后一个的有效配置。

## gRPC 流式订阅

### 单资源类型发现

每个 xDS API 可以单独配置 [`ApiConfigSource`](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/core/config_source.proto#core-apiconfigsource)，指向对应的上游管理服务器的集群地址。每个 xDS 资源类型会启动一个独立的双向 gRPC 流，可能对应不同的管理服务器。API 交付方式采用最终一致性。可以参考后续聚合服务发现（ADS） 章节来了解必要的显式控制序列。

> 译者注：core.ApiConfigSource 配置格式如下：

```json
{
  "api_type": "...",
  "cluster_names": [],
  "grpc_services": [],
  "refresh_delay": "{...}",
  "request_timeout": "{...}"
}
```

#### 类型 URL

每个 xDS API 都与给定的资源的类型存在 1:1 对应。关系如下：

- [LDS： `envoy.api.v2.Listener`](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/api/v2/lds.proto)
- [RDS： `envoy.api.v2.RouteConfiguration`](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/api/v2/rds.proto)
- [CDS： `envoy.api.v2.Cluster`](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/api/v2/cds.proto)
- [EDS： `envoy.api.v2.ClusterLoadAssignment`](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/api/v2/eds.proto)
- [SDS：`envoy.api.v2.Auth.Secret`](https://github.com/envoyproxy/data-plane-api/blob/master/envoy/api/v2/auth/cert.proto)

[_类型 URL_](https://developers.google.com/protocol-buffers/docs/proto3#any) 的概念如下所示，其采用 `type.googleapis.com/<resource type>` 的形式，例如 CDS 对应于  `type.googleapis.com/envoy.api.v2.Cluster`。在 Envoy 的请求和管理服务器的响应中，都包括了资源类型 URL。

#### ACK/NACK 和版本

每个 Envoy 流以  `DiscoveryRequest` 开始，包括了列表订阅的资源、订阅资源对应的类型 URL、节点标识符和空的 `version_info`。EDS 请求示例如下：

```yaml
version_info:
node: { id: envoy }
resource_names:
- foo
- bar
type_url: type.googleapis.com/envoy.api.v2.ClusterLoadAssignment
response_nonce:
```

管理服务器可立刻或等待资源就绪时发送 `DiscoveryResponse `作为响应，示例如下：

```yaml
version_info: X
resources:
- foo ClusterLoadAssignment proto encoding
- bar ClusterLoadAssignment proto encoding
type_url: type.googleapis.com/envoy.api.v2.ClusterLoadAssignment
nonce: A
```

Envoy 在处理 `DiscoveryResponse` 响应后，将通过流发送一个新的请求，请求包含应用成功的最后一个版本号和管理服务器提供的 `nonce`。如果本次更新已成功应用，则 `version_info` 的值设置为 __X__，如下序列图所示：

![ACK 后的版本更新](https://ws1.sinaimg.cn/mw690/7e0ee03agy1fvmxs5aod1j20cc06y74c.jpg)

在此序列图及后续中，将统一使用以下缩写格式：

- `DiscoveryRequest`： (V=`version_info`，R=`resource_names`，N=`response_nonce`，T=`type_url`)
- `DiscoveryResponse`： (V=`version_info`，R=`resources`，N=`nonce`，T=`type_url`)

> 译者注：在[信息安全](https://zh.wikipedia.org/wiki/%E8%B3%87%E8%A8%8A%E5%AE%89%E5%85%A8)中，**Nonce**是一个在加密通信只能使用一次的数字。在认证协议中，它往往是一个[随机](https://zh.wikipedia.org/wiki/%E9%9A%8F%E6%9C%BA)或[伪随机](https://zh.wikipedia.org/wiki/%E4%BC%AA%E9%9A%8F%E6%9C%BA)数，以避免[重放攻击](https://zh.wikipedia.org/wiki/%E9%87%8D%E6%94%BE%E6%94%BB%E5%87%BB)。Nonce也用于[流密码](https://zh.wikipedia.org/wiki/%E6%B5%81%E5%AF%86%E7%A0%81)以确保安全。如果需要使用相同的密钥加密一个以上的消息，就需要Nonce来确保不同的消息与该密钥加密的密钥流不同。（引用自[维基百科](https://zh.wikipedia.org/wiki/Nonce)）在本文中`nonce`是每次更新的数据包的唯一标识。

版本为 Envoy 和管理服务器提供了共享当前应用配置的概念和通过 ACK/NACK 来进行配置更新的机制。如果 Envoy 拒绝配置更新 __X__，则回复 [`error_detail`](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/discovery.proto#envoy-api-field-discoveryrequest-error-detail) 及前一个的版本号，在当前情况下为空的初始版本号，`error_detail` 包含了有关错误的更加详细的信息：

![NACK 无版本更新](https://ws1.sinaimg.cn/mw690/7e0ee03agy1fvmxtjqtcsj20cc06y0ss.jpg)

后续，API 更新可能会在新版本 __Y__ 上成功：

![ACK 紧接着 NACK](https://ws1.sinaimg.cn/mw690/7e0ee03agy1fvmxtwzc96j20cc0923yp.jpg)

每个流都有自己的版本概念，但不存在跨资源类型的共享版本。在不使用 ADS 的情况下，每个资源类型可能具有不同的版本，因为 Envoy API 允许指向不同的 EDS/RDS 资源配置并对应不同的 `ConfigSources`。

#### 何时发送更新

管理服务器应该只向 Envoy 客户端发送上次 `DiscoveryResponse` 后更新过的资源。Envoy 则会根据接受或拒绝 `DiscoveryResponse` 的情况，立即回复包含 ACK/NACK 的  `DiscoveryRequest` 请求。如果管理服务器每次发送相同的资源集结果，而不是根据其更新情况，则会导致 Envoy 和管理服务器通讯效率大打折扣。

在同一个流中，新的 `DiscoveryRequests` 将取代此前具有相同的资源类型 `DiscoveryRequest` 请求。**这意味着管理服务器只需要响应给定资源类型最新的 `DiscoveryRequest` 请求即可。**

#### 资源提示

`DiscoveryRequest` 中的 `resource_names` 信息作为资源提示出现。一些资源类型，例如 `Cluster` 和 `Listener` 将使用一个空的 `resource_names`，因为 Envoy 需要获取管理服务器对应于节点标识的所有 `Cluster`（CDS）和  `Listener`（LDS）。对于其他资源类型，如 `RouteConfigurations`（RDS）和 `ClusterLoadAssignments`（EDS），则遵循此前的 CDS/LDS 更新，Envoy 能够明确地枚举这些资源。

LDS/CDS 资源提示信息将始终为空，并且期望管理服务器的每个响应都提供 `LDS/CDS` 资源的完整状态。缺席的 `Listener` 或 `Cluster` 将被删除。

对于 EDS/RDS，管理服务器并不需要为每个请求的资源进行响应，而且还可能提供额外未请求的资源。`resource_names` 只是一个提示。Envoy 将默默地忽略返回的多余资源。如果请求的资源中缺少相应的 RDS 或 EDS 更新，Envoy 将保留对应资源的最后的值。管理服务器可能会依据  `DiscoveryRequest` 中 `node` 标识推断其所需的 EDS/RDS 资源，在这种情况下，提示信息可能会被丢弃。从相应的角度来看，空的 EDS/RDS `DiscoveryResponse` 响应实际上是表明在 Envoy 中为一个空的资源。

当 `Listener` 或 `Cluster` 被删除时，其对应的 EDS 和 RDS 资源也需要在 Envoy 实例中删除。为使 EDS 资源被 Envoy 已知或跟踪，就必须存在应用过的 `Cluster` 定义（如通过 CDS 获取）。RDS 和 `Listeners` 之间存在类似的关系（如通过 LDS 获取）。

对于 EDS/RDS ，Envoy 可以为每个给定类型的资源生成不同的流（如每个 `ConfigSource` 都有自己的上游管理服务器的集群）或当指定资源类型的请求发送到同一个管理服务器的时候，允许将多个资源请求组合在一起发送。虽然可以单个实现，但管理服务器应具备处理每个给定资源类型中对单个或多个 `resource_names`  请求的能力。下面的两个序列图对于获取两个 EDS 资源都是有效的 `{foo，bar}`：

![一个流上多个 EDS 请求](https://ws1.sinaimg.cn/mw690/7e0ee03agy1fvmxuviiqsj20eh06ymx9.jpg)
![不同流上的多个 EDS 请求](https://ws1.sinaimg.cn/mw690/7e0ee03agy1fvmxv7cv21j20j20a4wet.jpg)

#### 资源更新

如上所述，Envoy 可能会更新  `DiscoveryRequest` 中出现的 `resource_names` 列表，其中 `DiscoveryRequest`  是用来 ACK/NACK 管理服务器的特定的 `DiscoveryResponse` 。此外，Envoy 后续可能会发送额外的 `DiscoveryRequests` ，用于在特定 `version_info` 上使用新的资源提示来更新管理服务器。例如，如果 Envoy 在 EDS 版本 __X__ 时仅知道集群 `foo`，但在随后收到的 CDS 更新时额外获取了集群 `bar` ，它可能会为版本 __X__ 发出额外的 `DiscoveryRequest` 请求，并将 `{foo，bar}` 作为请求的 `resource_names` 。

![CDS 响应导致 EDS 资源更新](https://ws1.sinaimg.cn/large/006tNc79ly1fvph0p7u8zj31fm0lq0ve.jpg)

这里可能会出现竞争状况；如果 Envoy 在版本 __X__ 上发布了资源提示更新请求，但在管理服务器处理该请求之前发送了新的版本号为 __Y__  的响应，针对 `version_info` 为 __X__ 的版本，资源提示更新可能会被解释为拒绝  __Y__ 。为避免这种情况，通过使用管理服务器提供的 `nonce`，Envoy 可用来保证每个 `DiscoveryRequest` 对应到相应的 `DiscoveryResponse` ：

![EDS 更新速率激发 nonces](https://ws4.sinaimg.cn/large/006tNc79ly1fvph04ln3fj31kw0rogqc.jpg)

管理服务器不应该为含有过期 `nonce` 的 `DiscoveryRequest` 发送 `DiscoveryResponse` 响应。在向 Envoy 发送的 `DiscoveryResponse`  中包含了的新 `nonce` ，则此前的 `nonce` 将过期。在资源新版本就绪之前，管理服务器不需要向 Envoy 发送更新。同版本的早期请求将会过期。在新版本就绪时，管理服务器可能会处理同一个版本号的多个 `DiscoveryRequests`请求。

![请求变的陈旧](https://ws3.sinaimg.cn/large/006tNc79ly1fvpgy6xewrj31b415ctcy.jpg)

上述资源更新序列表明 Envoy 并不能期待其发出的每个 `DiscoveryRequest` 都得到 `DiscoveryResponse` 响应。

#### 最终一致性考虑

由于 Envoy 的 xDS API 采用最终一致性，因此在更新期间可能导致流量被丢弃。例如，如果通过 CDS/EDS 仅获取到了集群 __X__，而且 `RouteConfiguration` 引用了集群  __X__；在 CDS/EDS 更新集群  __Y__  配置之前，如果将 `RouteConfiguration` 将引用的集群调整为 __Y__ ，那么流量将被吸入黑洞而丢弃，直至集群 __Y__ 被 Envoy 实例获取。

对某些应用程序，可接受临时的流量丢弃，客户端重试或其他 Envoy sidecar 会掩盖流量丢弃。那些对流量丢弃不能容忍的场景，可以通过以下方式避免流量丢失，CDS/EDS 更新同时携带 __X__ 和 __Y__ ，然后发送 RDS 更新从 __X__ 切换到 __Y__ ，此后发送丢弃 __X__ 的 CDS/EDS 更新。

一般来说，为避免流量丢弃，更新的顺序应该遵循 `make before break` 模型，其中

- 必须始终先推送 CDS 更新（如果有）。
- EDS 更新（如果有）必须在相应集群的 CDS 更新后到达。
- LDS 更新必须在相应的 CDS/EDS 更新后到达。
- 与新添加的监听器相关的 RDS 更新必须在最后到达。
- 最后，删除过期的 CDS 集群和相关的 EDS 端点（不再被引用的端点）。

如果没有新的集群/路由/监听器或者允许更新时临时流量丢失的情况下，可以独立推送 xDS 更新。请注意，在 LDS 更新的情况下，监听器须在接收流量之前被预热，例如如其配置了依赖的路由，则先需先从 RDS 进行获取。添加/删除/更新集群信息时，集群也需要进行预热。另一方面，如果管理平面确保路由更新时所引用的集群已经准备就绪，路由可以不用预热。

### 聚合服务发现（ADS）

当管理服务器进行资源分发时，通过上述保证交互顺序的方式来避免流量丢弃是一项很有挑战的工作。ADS 允许单一管理服务器通过单个 gRPC 流，提供所有的 API 更新。配合仔细规划的更新顺序，ADS 可规避更新过程中流量丢失。使用 ADS，在单个流上可通过类型 URL 来进行复用多个独立的 `DiscoveryRequest`/`DiscoveryResponse` 序列。对于任何给定类型的 URL，以上 `DiscoveryRequest` 和 `DiscoveryResponse` 消息序列都适用。 更新序列可能如下所示：

![EDS/CDS 在一个 ADS 流上多路复用](https://ws2.sinaimg.cn/large/006tNc79ly1fvpgxnl947j313q0wgq62.jpg)

每个 Envoy 实例可使用单独的 ADS 流。

最小化 ADS 配置的 `bootstrap.yaml` 片段示例如下：

```yaml
node:
  id: <node identifier>
dynamic_resources:
  cds_config: {ads: {}}
  lds_config: {ads: {}}
  ads_config:
    api_type: GRPC
    grpc_services:
      envoy_grpc:
        cluster_name: ads_cluster
static_resources:
  clusters:
  - name: ads_cluster
    connect_timeout: { seconds: 5 }
    type: STATIC
    hosts:
    - socket_address:
        address: <ADS management server IP address>
        port_value: <ADS management server port>
    lb_policy: ROUND_ROBIN
    http2_protocol_options: {}
admin:
  ...

```

### 增量 xDS

增量 xDS 是可用于允许的 ADS、CDS 和 RDS 单独 xDS 端点：

- xDS 客户端对跟踪资源列表进行增量更新。这支持 Envoy 按需/惰性地请求额外资源。例如，当与未知集群相对应的请求到达时，可能会发生这种情况。
- xDS 服务器可以增量更新客户端上的资源。这支持 xDS 资源可伸缩性的目标。管理服务器只需交付更改的单个集群，而不是在修改单个集群时交付所有上万个集群。

xDS 增量会话始终位于 gRPC 双向流的上下文中。这允许 xDS 服务器能够跟踪到连接的 xDS 客户端的状态。xDS REST 版本不支持增量。

在增量 xDS 中，nonce 字段是必需的，用于匹配  [`IncrementalDiscoveryResponse`](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/discovery.proto#discoveryrequest) 关联的 ACK 或 NACK [`IncrementalDiscoveryRequest`](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/discovery.proto#discoveryrequest)。可选地，存在响应消息级别的 system_version_info，但仅用于调试目的。

`IncrementalDiscoveryRequest` 可在以下 3 种情况下发送：

1. xDS 双向 gRPC 流的初始消息。
2. 作为对先前的 `IncrementalDiscoveryResponse` 的 ACK 或 NACK 响应。在这种情况下，`response_nonce` 被设置为响应中的 nonce 值。ACK 或 NACK 由可由 `error_detail` 字段是否出现来区分。
3. 客户端自发的 `IncrementalDiscoveryRequest`。此场景下可以采用动态添加或删除被跟踪的 `resource_names` 集。这种场景下，必须忽略 `response_nonce`。

在第一个示例中，客户端连接并接收它的第一个更新并 ACK。第二次更新失败，客户端发送 NACK 拒绝更新。xDS客户端后续会自发地请求 “wc” 相关资源。

![增量 session 示例](https://ws4.sinaimg.cn/large/006tNc79ly1fvpgwfbep7j31kw0vldli.jpg)

在重新连接时，支持增量的 xDS 客户端可能会告诉服务器其已知资源从而避免通过网络重新发送它们。

![增量重连示例](https://ws2.sinaimg.cn/large/006tNc79ly1fvpgx05z3kj31kw0phwif.jpg)

## REST-JSON 轮询订阅

单个 xDS  API 可对 REST 端点进行同步（长）轮询。除了无持久流与管理服务器交互外，消息顺序与上述相似。在任何时间点，只存在一个未完成的请求，因此响应消息中的 `nonce` 在 REST-JSON 中是可选的。`DiscoveryRequest` 和 `DiscoveryResponse` 的消息编码遵循 [JSON 变换 proto3](https://developers.google.com/protocol-buffers/docs/proto3#json) 规范。ADS 不支持 REST-JSON 轮询。

当轮询期间设置为较小的值时，则可以等同于长轮询，这时要求避免发送 `DiscoveryResponse`，[除非对请求的资源发生了更改](#何时发送更新)。

## 参考

- [xDS REST and gRPC protocol - github.com](https://github.com/envoyproxy/data-plane-api/blob/master/XDS_PROTOCOL.md)