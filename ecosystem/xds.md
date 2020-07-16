---
authors: ["mlycore"]
reviewers: ["malphi","rootsongjc"]
---

# xDS

xDS 协议是 “X Discovery Service” 的简写，这里的 “X” 表示它不是指具体的某个协议，是一组基于不同数据源的服务发现协议的总称，包括 CDS、LDS、EDS、RDS 和 SDS 等。客户端可以通过多种方式获取数据资源，比如监听指定文件、订阅 gRPC stream 以及轮询相应的 REST API 等。

在 Istio 架构中，基于 xDS 协议提供了标准的控制面规范，并以此向数据面传递服务信息和治理规则。在 Envoy 中，xDS 被称为数据平面 API，并且担任控制平面 Pilot 和数据平面 Envoy 的通信协议，同时这些 API 在特定场景里也可以被其他代理所使用。目前 xDS 主要有两个版本 v2 和 v3，其中 v2 版本将于 2020 年底停止使用。

注：对于通用数据平面标准 API 的工作将由 CNCF 下设的 UDPA 工作组开展，后面一节会专门介绍。

## xDS 协议简介

在 Pilot 和 Envoy 通信的场景中，xDS 协议是基于 gRPC 实现的传输协议，即 Envoy 通过 gRPC streaming 订阅 Pilot 的资源配置。Pilot 借助 ADS 对 API 更新推送排序的能力，按照 CDS-EDS-LDS-RDS 的顺序串行分发配置。

![xds-pilot-envoy-arch](../images/xds-pilot-envoy.png)

图中的 ADS 将 xDS 所有的协议都聚合到一起，即上文提到的 CDS、EDS、LDS 和 RDS 等，Envoy 通过这些 API 可以动态地从 Pilot 获取对 Cluster（集群）、Endpoint（集群成员）、Listener（监听器）和 Route（路由）等资源的配置。下表整理了主要的 xDS API：

|服务简写 | 全称 | 描述 |
|:-:|:-:|:-:|
|LDS|Listener Discovery Service | 监听器发现服务 |
|RDS|Route Discovery Service | 路由发现服务 |
|CDS|Cluster Discovery Service | 集群发现服务 |
|EDS|Endpoint Discovery Service | 集群成员发现服务 |
|SDS|Service Discovery Service|v1 时的集群成员发现服务，后改名为 EDS|
|ADS|Aggregated Discovery Service | 聚合发现服务 |
|HDS|Health Discovery Service | 健康度发现服务 |
|SDS|Secret Discovery Service | 密钥发现服务 |
|MS|Metric Service | 指标发现服务 |
|RLS|Rate Limit Service | 限流发现服务 |
|xDS|| 以上各种 API 的统称 |

### CDS 

CDS 是 Cluster Discovery Service 的缩写，Envoy 使用它在进行路由的时候发现上游 Cluster。Envoy 通常会优雅地添加、更新和删除 Cluster。有了 CDS 协议，Envoy 在初次启动的时候不一定要感知拓扑里所有的上游 Cluster。在做路由 HTTP 请求的时候通过在 HTTP 请求头里添加 Cluster 信息实现请求转发。

尽管可以在不使用 EDS（Endpoint Discovery Service）的情况下通过指定静态集群的方式使用 CDS（Cluster Discovery Service），但仍然推荐通过 EDS API 实现。因为从内部实现来说，Cluster 定义会被优雅地更新，也就是说所有已建立的连接池都必须排空然后重连。使用 EDS 就可以避免这个问题，当通过 EDS 协议添加或移除 hosts 时，Cluster 里现有的 hosts 不会受此影响。

### EDS

EDS 即 Endpoint Discovery Service 的缩写。在 Envoy 术语中，Endpoint 即 Cluster 的成员。Envoy 通过 EDS API 可以更加智能地动态获取上游 Endpoint。使用 EDS 作为首选服务发现的原因有二：
* EDS 可以突破 DNS 解析的最大记录数限制，同时可以使用负载均衡和路由中的很多信息，因而可以做出更加智能的负载均衡策略。
* Endpoint 配置包含灰度状态、负载权重和可用域等 hosts 信息，可用于服务网格负载均衡和实现信息统计等。

### LDS

LDS 即 Listener Discovery Service 的缩写。基于此，Envoy 可以在运行时发现所有的 Listener，包括 L3 和 L4 filter 等所有的 filter 栈，并由此执行各种代理工作，如认证、TCP 代理和 HTTP 代理等。添加 LDS 使得 Envoy 的任何配置都可以动态执行，只有发生一些非常罕见的变更（管理员、追踪驱动等）、证书轮转或二进制更新时才会使用热更新。

### RDS

RDS 即 Router Discovery Service 的缩写，用于 Envoy 在运行时为 HTTP 连接管理 filter 获取完整的路由配置，比如 HTTP 头部修改等。并且路由配置会被优雅地写入而无需影响已有的请求。当 RDS 和 EDS、CDS 共同使用时，可以帮助构建一个复杂的路由拓扑蓝绿发布等。

### ADS 

EDS，CDS 等每个独立的服务都对应了不同的 gRPC 服务名称。对于需要控制不同类型资源抵达 Envoy 顺序的需求，可以使用聚合发现服务，即 Aggregated xDS，它可以通过单一的 gRPC 服务流支持所有的资源类型，借助于有序的配置分发，从而解决资源更新顺序的问题。

## xDS 协议的基本流程

作为 Pilot 和 Envoy 之间通信协议的 xDS，可以通过两种方式实现：gRPC 和 REST，无论哪种方法都是通过 xDS API 发送 DiscoveryRequest 请求，然后解析响应 DiscoveryResponse 中包含的配置信息并动态加载。

![xds-envoy-pilot-flow](../images/xds-envoy-pilot-flow.png)

### DiscoveryRequest

DiscoveryRequest 是结构化的请求，它为某个 Envoy 请求包含了某些 xDS API 的一组版本化配置资源。相关字段展示如下表：

| 属性名 | 类型 | 作用 |
|:-:|:-:|:-:|
|VersionInfo|string | 成功加载的资源版本号，首次为空 |
|Node|`*core.Node` | 发起请求的节点信息，如位置信息等元数据 |
|ResourceNames|[]string | 请求的资源名称列表，为空表示订阅所有的资源 |
|TypeUrl|string | 资源类型 |
|ResponseNonce|string|ACK/NACK 特定的 response|
|ErrorDetail|`*rpc.Status` | 代理加载配置失败，ACK 为空 |

### DiscoveryResponse

类似于 DiscoveryRequest，DiscoveryResponse 的相关字段如下表：

| 属性名 | 类型 | 作用 |
|:-:|:-:|:-:|
|VersionInfo|string|Pilot 响应版本号 |
|Resources|[]types.Any | 序列化资源，可表示任意类型的资源 |
|TypeUrl|string | 资源类型 |
|Nonce|string | 基于 gRPC 的订阅使用，nonce 提供了一种在随后的 DiscoveryRequest 中明确 ACK 特定 DiscoveryResponse 的方法 |

### ACK/NACK

当 Envoy 使用 DiscoveryRequest 和 DiscoveryResponse 进行通信的时候，除了可以在类型级别指定版本，还有一种资源实例版本，它不属于 API 的属性。例如如下的 EDS 请求：

```yaml
version_info:
node: {id: envoy}
resource_names:
- foo
- bar
type_url: type.googleapis.com/envoy.api.v2.ClusterLoadAssignment
response_nonce:
```

管理服务端可能会立即返回响应，也可能在请求资源可用时通过 DiscoveryResponse 返回，示例如下：

```yaml
version_info: X
resources:
- foo ClusterLoadAssignment proto encoding
- bar ClusterLoadAssignment proto encoding
type_url: type.googleapis.com/envoy.api.v2.ClusterLoadAssignment
nonce: A
```

当 Envoy 解析完 DiscoveryResponse 以后，将通过流发送一个新的请求，指明最近成功应用的版本以及服务器提供的 Nonce（注：Nonce 是加密通信中用于一次一密的随机数，以免重放攻击）。借助于这个版本给 Envoy 和管理服务端同时指明当前所使用的配置版本。这种 ACK/NACK 的机制分别实现对应用新 API 配置版本或先前的 API 配置版本进行标识。

#### ACK

如果更新被成功应用，version_info 将如图所示置为 X：

![xDS ACK](../images/xds-ack.png)

#### NACK

如果 Envoy 拒绝了配置更新 X，那么会返回具体的 error_detail 以及之前的版本号，下图中为空。

![xDS NACK](../images/xds-nack.png)

对于 xDS 客户端来说，每当收到 DiscoveryResponse 时都应该进行 ACK 或 NACK。ACK 标识成功的配置更新，并且包含来自 DiscoveryResponse 的 version_info，而 NACK 标识失败的配置更新，并且包含之前的 version_info。只有 NACK 应该有 error_detail 字段。

### 基于 xDS 的推和拉 

Envoy 在启动时会和 Pilot 建立全双工的长链接，这就为实现双向配置分发提供了条件。具体来说在 Pilot 与 Envoy 进行通信的时候有主动和被动两种方式，它们分别对应推和拉两个动作。在主动分发模式里，由 Pilot 监听到事件变化以后分发给 Envoy 。在被动分发模式里，由 Envoy 订阅特定资源事件，当资源更新时生成配置并下发。

## xDS 协议的特点

对于通过 gRPC streaming 传输的 xDS 协议有四个变种，它们覆盖了两个维度。

第一个维度是全量（State of the World：SotW）传输对比增量（Incremental）传输。早期的 xDS 使用了全量传输，客户端必须在每个请求里指定所有的资源名，服务端返回所有资源。这种方式的扩展性受限。所以后来引入了增量传输，在这种方式里允许客户端和服务端指定相对之前状态变化的部分，这样服务端就只需返回那些发生了变化的资源。同时增量传输还提供了对于资源的 “慢加载”。

第二个维度是每种资源独立的 gRPC stream 对比所有资源聚合 gRPC stream。同样前者是早期 xDS 早期使用的方式，它提供了最终一致性模型。后者对应于那些需要显式控制传输流的场景。

所以这四个变种分别为：
1. State of the World（Basic xDS）：全量传输独立 gRPC stream；
2. Incremental xDS：增量传输独立 gRPC stream；
3. Aggregated Discovery Service（ADS）：全量传输聚合 gRPC stream；
4. Incremental ADS：增量传输聚合 gRPC stream （暂未实现）；

对于所有的全量方法，请求和响应类型分别为 DiscoveryRequest 和 DiscoverResponse；对于所有的增量方法，请求和响应类型分别为 DeltaDiscoveryRequest 和 DeltaDiscoveryResposne。

### 增量 xDS

每个 xDS 协议都拥有两种 Gprc 服务，一种是 Stream，另一种是 Delta。在 Envoy 设计早期采用了全量更新策略，即以 Stream 的方式来提供强一致的配置同步。如此一来，任何配置的变更都会触发全量配置下发，显然这种全量更新的方式会为整个网格带来很高的负担。所以 Envoy 社区提出了 Delta xDS 方案，当配置发生变化时，仅下发和更新发生变化的配置部分。

增量 xDS 利用 gRPC 全双工流，支持 xDS 服务器追踪 xDS 客户端的状态。在增量 xDS 协议中，nonce 域用来指明 DeltaDiscoveryResponse 和 DeltaDiscoveryRequest ACK 或 NACK。

对于 DeltaDiscoveryRequest 可以在如下场景里发送：
* xDS 全双工 gRPC stream 中的初始化消息；
* 作为对前序 DeltaDiscoveryResponse 的 ACK 或 NACK；
* 在动态添加或移除资源时客户端自动发来的 DeltaDiscoveryRequest，此场景中必须忽略 response_nonce 字段；

在下面第一个例子中，客户端收到第一个更新并且返回 ACK，而第二次更新失败返回了 NACK，之后 xDS 客户端自发请求 'wc' 资源：

![增量 xDS](../images/xds-incremental.png)

在网络重连以后，因为并没有对之前的状态进行保存，增量 xDS 客户端需要向服务器告知它已拥有的资源从而避免重复发送：

![xDS 增量重连](../images/xds-incremental-reconnect.png)

### 最终一致性

对于分布式系统而言，在设计之初选择强一致性还是最终一致性是很关键的一步，它直接关系到未来的应用场景。比如 ZooKeeper 就是强一致性服务发现的代表。但是对于服务网格的场景来说，可能同时存在成百上千个节点，这些节点间进行如此庞大的数据复制是相当困难的，并且很有可能会耗尽资源。也就是说对于分布式系统来说，为了提供强一致性需要付出巨大的代价。Envoy 在设计之初就选择了最终一致性，并且从底层线程模型到上层配置发现都进行了相应的实现。这样一来不仅简化了系统，提供了更好的性能，也更方便运维。

因为 Envoy xDS API 是满足最终一致性，部分流量可能在更新时被丢弃。比如只有集群 X 可以通过 CDS/EDS 发现，那么当引用集群 X 的路由配置更新时，并且在 CDS/EDS 更新前将配置指向集群 Y，那么在 Envoy 实例获取配置前的部分流量会被丢弃。

对于一些应用来说可以接受暂时的流量丢弃，在客户端或者其他 Envoy Sidecar 的重试会掩盖这次丢弃。对于其它无法忍受数据丢弃的场景来说，流量丢弃可以通过更新对集群 X 和 Y 的 CDS/EDS 来避免，然后 RDS 更新里将 X 指向 Y，并且 CDS/EDS 更新中丢弃集群 X。

通常为了避免丢弃，更新的顺序应该遵循 make before break 规则，即：
* CDS 更新应该被最先推送；
* 对相应集群的 EDS 更新必须在 CDS 更新后到达；
* LDS 更新必须在对应的 CDS/EDS 更新后到达；
* 对新增的相关监听器的 RDS 更新必须在 CDS/EDS/LDS 更新后到达；
* 对任何新增路由配置相关的 VHDS 更新必须在 RDS 更新后到达；
* 过期的 CDS 集群和相关的 EDS 端点此刻被移除；

如果没有新的集群、路由或监听器添加，或者应用可以接受短期的流量丢弃，那么 xDS 更新可以被独立推送。在 LDS 更新的场景里，监听器要在收到流量前被预热。当添加、移除或更新集群时要对集群进行预热。另一方面，路由不需要被预热。

## xDS 协议生态

按照 Envoy 的设想，社区中无论是实现控制面的团队，还是实现数据面的团队，都希望能参与和使用 [github.com/envoyproxy/data-plane-api](https://github.com/envoyproxy/data-plane-api) 上规定的这套控制面和数据面之间的数据平面 API 接口。

## 参考资料：
* [Envoy 官方文档：xDS 协议](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol)
* [Embracing eventual consistency in soa networking](https://blog.envoyproxy.io/embracing-eventual-consistency-in-soa-networking-32a5ee5d443d)
