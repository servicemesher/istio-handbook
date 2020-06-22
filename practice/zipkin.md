---
authors: ["lyzhang1999"]
reviewers: ["rootsongjc","GuangmingLuo","malphi"]
---

# Zipkin
Zipkin是一个分布式跟踪系统，它有助于收集服务的调用关系和服务的调用时序数据，它的功能包括收集数据和结果展示。
## 追踪原理
在分布式系统架构中，用户的一次请求，往往需要经过不同的服务和模块，才能够最终返回请求结果。不同的“微服务”之间，每次调用都会形成一个完整的调用链，就像一个树状结构。
![微服务间调用的树状结构](../images/zipkin-user-invoke-link.png)

这一颗完整的树，我们称为 `Trace`， 树里的每一个节点，我们称之为 `Span` 。当服务被调用的时候，都会为 `Trace` 和 `Span` 生成唯一的 `ID` 标识，通过记录调用发起方 `parentId` ，我们就可以很清晰的将每一个 `Span` 串联起来，形成一条完整的调用链，这是分布式调用链追踪的基本原理。

当然，Zipkin 兼容 `OpenTracing` 协议，所以还会记录其他的信息如：时间戳、span 上下文、以及额外的 K/V tag 等。

## Envoy-Zipkin 架构
在 `Istio` 中，`Envoy` 原生支持分布式追踪系统 Zipkin，当请求通过 `Envoy` Sidecar 流量拦截时，`Envoy` 会自动为 HTTP Header 添加 `x-b3` 开头的 Header 和 `x-request-id`，此 Header 在不同的服务之间传递，最终上报给 `Zipkin` 解析出完整的调用链。

详细的过程大致为：
* 如果 incoming 请求没有 trace 相关的 headers，则会在流量进入 POD 之前创建一个 root span。
* 如果 incoming 请求包含 trace 相关的 headers，Envoy 将会解析 Span 上下文信息，然后在流量进入 POD 之前创建一个新的 Span 继承自旧的 Span 上下文。

![Zipkin 追踪过程(根据 Zipkin 官方重绘)](../images/zipkin-principle.png)

在 Istio 服务网格内，调用链的 `Span` 信息由 `Envoy` 通过 proxy 直接上报给 `zipkinServer`，不同服务和 `Zipkin` 的调用流程大致如下：

![Envoy-Zipkin 追踪过程(根据 Zipkin 官方重绘)](../images/zipkin-architecture.png)

* 传输层：默认为 HTTP 的方式，当然还可以使用 Kafka、Scribe 的方式。
* 收集器（Collector）：收集过来的 span 数据，并进行数据验证、存储以及创建必要的索引。
* 存储（storage）：默认是 in-memory 的方式，用于测试，请注意此方式追踪数据并不会被持久化，其他可选方式有 JDBC（Mysql）、Cassandra、Elasticsearch 。
* API：提供 API 外部调用查询，主要提供 Web UI 调用。
* User Interfaces（Web UI）：提供追踪数据查询的 Web UI 界面展示。

## 环境准备

## 部署 Zipkin
这里解析源码的行为，需要传递，istio 总览里面的内容；
## 参考
* [Envoy - 追踪](https://www.servicemesher.com/envoy/intro/arch_overview/tracing.html)
* [Zipkin - Architecture](https://zipkin.io/pages/architecture.html)
* [Istio - Distributed-tracing](https://istio.io/latest/docs/tasks/observability/distributed-tracing/overview/)
* [Istio - Zipkin](https://istio.io/latest/docs/tasks/observability/distributed-tracing/zipkin/)
* [分布式集群环境下调用链路追踪](https://www.ibm.com/developerworks/cn/web/wa-distributed-systems-request-tracing/index.html)
* [Distributed Tracing, Istio and Your Applications](https://thenewstack.io/distributed-tracing-Istio-and-your-applications/)