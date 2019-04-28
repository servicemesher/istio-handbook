---
owners: ["rootsongjc"]
reviewers: [""]
description: "本文是 xDS 协议中 CDS 的解析，译自 Envoy 官方文档。"
publishDate: 2019-03-10
updateDate: 2019-03-10
tags: ["envoy","cds","xds"]
category: "translation"
---

# CDS（集群发现服务）

集群发现服务（CDS）是一个可选的 API，Envoy 将调用该 API 来动态获取 cluster manager 的成员。Envoy 还将根据 API 响应协调集群管理，根据需要完成添加、修改或删除已知的集群。

关于 Envoy 是如何通过 CDS 从 `pilot-discovery` 服务中获取的 cluster 配置，请参考 [Service Mesh深度学习系列part3—istio源码分析之pilot-discovery模块分析（续）](http://www.servicemesher.com/blog/istio-service-mesh-source-code-pilot-discovery-module-deepin-part2)一文中的 CDS 服务部分。

**注意**

- 在 Envoy 配置中静态定义的 cluster 不能通过 CDS API 进行修改或删除。

- Envoy 从 1.9 版本开始已不再支持 v1 API。

- [v2 CDS API](https://www.envoyproxy.io/docs/envoy/latest/configuration/overview/v2_overview#v2-grpc-streaming-endpoints)

## 统计

CDS 的统计树以 `cluster_manager.cds.` 为根，统计如下：

| 名字                          | 类型    | 描述                                                         |
| ----------------------------- | ------- | ------------------------------------------------------------ |
| config_reload                 | Counter | 因配置不同而导致配置重新加载的总次数                         |
| update_attempt                | Counter | 尝试调用配置加载 API 的总次数                                |
| update_success                | Counter | 调用配置加载 API 成功的总次数                                |
| update_failure                | Counter | 调用配置加载 API 因网络错误的失败总数                        |
| update_rejected               | Counter | 调用配置加载 API 因 schema/验证错误的失败总次数              |
| version                       | Gauge   | 来自上次成功调用配置加载API的内容哈希                        |
| control_plane.connected_state | Gauge   | 布尔值，用来表示与管理服务器的连接状态，1表示已连接，0表示断开连接 |

## 参考

- [Service Mesh深度学习系列part3—istio源码分析之pilot-discovery模块分析（续）- servicemesher.com](http://www.servicemesher.com/blog/istio-service-mesh-source-code-pilot-discovery-module-deepin-part2)
- [Cluster discovery service - envoyproxy.io](https://www.envoyproxy.io/docs/envoy/latest/configuration/cluster_manager/cds)
