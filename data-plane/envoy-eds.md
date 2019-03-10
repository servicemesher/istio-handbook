---
owners: ["rootsongjc"]
reviewers: [""]
description: "本文是 xDS 协议中 EDS 的解析，译自 Envoy 官方文档。"
publishDate: 2019-03-10
updateDate: 2019-03-10
tags: ["envoy","eds","xds"]
category: "translation"
---

# EDS（端点发现服务）

EDS 只是 Envoy 中众多的[服务发现](http://www.servicemesher.com/envoy/intro/arch_overview/service_discovery.html)方式的一种。要想了解 EDS 首先我们需要先知道什么是 Endpoint。

**Endpoint**

Endpoint 即上游主机标识。它的数据结构如下：

```json
{
  "address": "{...}",
  "health_check_config": "{...}"
}
```

其中包括端点的地址和健康检查配置。详情请参考 [endpoint.Endpoint](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/endpoint/endpoint.proto)。

终端发现服务（EDS）是一个[基于 gRPC 或 REST-JSON API 服务器的 xDS 管理服务](https://www.envoyproxy.io/docs/envoy/latest/configuration/overview/v2_overview#config-overview-v2-management-server)，在 Envoy 中用来获取集群成员。集群成员在 Envoy 的术语中被称为“终端”。对于每个集群，Envoy 都会通过发现服务来获取成员的终端。由于以下几个原因，EDS 是首选的服务发现机制：

- Envoy 对每个上游主机都有明确的了解（与通过 DNS 解析的负载均衡进行路由相比而言），并可以做出更智能的负载均衡决策。
- 在每个主机的发现 API 响应中携带的额外属性通知 Envoy 负载均衡权重、金丝雀状态、区域等。这些附加属性在负载均衡、统计信息收集等过程中会被 Envoy 网格全局使用。

Envoy 提供了 [Java](https://github.com/envoyproxy/java-control-plane) 和 [Go](https://github.com/envoyproxy/go-control-plane) 语言版本的 EDS 和[其他发现服务](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/dynamic_configuration#arch-overview-dynamic-config)的参考 gRPC 实现。

通常，主动健康检查与最终一致的服务发现服务数据结合使用，以进行负载均衡和路由决策。

## 参考

- [Service Mesh深度学习系列part3—istio源码分析之pilot-discovery模块分析（续）- servicemesher.com](http://www.servicemesher.com/blog/istio-service-mesh-source-code-pilot-discovery-module-deepin-part2)
- [EDS - envoyproxy.io](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/eds.proto)
- [endpoint.Endpoint - envoyproxy.io](https://www.envoyproxy.io/docs/envoy/latest/api-v2/api/v2/endpoint/endpoint.proto)