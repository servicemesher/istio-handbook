---
owners: ["rootsongjc"]
reviewers: [""]
description: "本文是 xDS 协议中 ADS 的解析，译自 Envoy 官方文档。"
publishDate: 2019-03-10
updateDate: 2019-03-10
tags: ["envoy","ads","xds"]
category: "translation"
---

# ADS（聚合发现服务）

虽然 Envoy 本质上采用了最终一致性模型，但 ADS 提供了**对 API 更新推送进行排序**的机会，并确保单个管理服务器对 Envoy 节点的 API 更新具有亲和力。ADS 允许管理服务器在单个双向 gRPC 流上传递一个或多个 API 及其资源。否则，一些 API（如 RDS 和 EDS）可能需要管理多个流并连接到不同的管理服务器。

**ADS** 通过适当得排序 xDS 可以无中断的更新 Enovy 的配置。例如，假设 foo.com 已映射到集群 X。我们希望将路由表中将该映射更改为在集群 Y。为此，必须首先提供 X、Y 这两个集群的 CDS/EDS 更新。

如果没有 ADS，CDS/EDS/RDS 流可能指向不同的管理服务器，或者位于需要协调的不同 gRPC流连接的同一管理服务器上。EDS 资源请求可以跨两个不同的流分开，一个用于 X，一个用于 Y。ADS 将这些流合并到单个流和单个管理服务器，从而无需分布式同步就可以正确地对更新进行排序。使用 ADS，管理服务器将在单个流上提供 CDS、EDS 和 RDS 更新。

**ADS** 仅适用于 gRPC 流（非REST），[本文档](https://github.com/envoyproxy/data-plane-api/blob/master/xds_protocol.rst#aggregated-discovery-service-ads)对此进行了更全面的描述。

## 参考

- [Aggregated Discovery Service - envoyproxy.io](https://www.envoyproxy.io/docs/envoy/latest/configuration/overview/v2_overview#aggregated-discovery-service)
