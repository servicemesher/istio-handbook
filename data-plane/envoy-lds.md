---
owners: ["rootsongjc"]
reviewers: [""]
description: "本文是 xDS 协议中 LDS 的解析，译自 Envoy 官方文档。"
publishDate: 2019-03-10
updateDate: 2019-03-10
tags: ["envoy","lds","xds"]
category: "translation"
---

# LDS（监听器发现服务）

Listener 发现服务（LDS）是一个可选的 API，Envoy 将调用它来动态获取 Listener。Envoy 将协调 API 响应，并根据需要添加、修改或删除已知的 Listener。

Listener 更新的语义如下：

- 每个 Listener 必须有一个独特的[名字](https://www.envoyproxy.io/docs/envoy/latest/api-v1/listeners/listeners.md#config-listeners-name)。如果没有提供名称，Envoy 将创建一个 UUID。要动态更新的 Listener ，管理服务必须提供 Listener 的唯一名称。
- 当一个 Listener 被添加，在参与连接处理之前，会先进入“预热”阶段。例如，如果 Listener 引用 [RDS](http://www.servicemesher.com/envoy/configuration/http_conn_man/rds.html#config-http-conn-man-rds) 配置，那么在 Listener 迁移到 “active” 之前，将会解析并提取该配置。
- Listener 一旦创建，实际上就会保持不变。因此，更新 Listener 时，会创建一个全新的 Listener （使用相同的侦听套接字）。新增加的监听者都会通过上面所描述的相同“预热”过程。
- 当更新或删除 Listener 时，旧的 Listener 将被置于 “draining（逐出）” 状态，就像整个服务重新启动时一样。Listener 移除之后，该 Listener 所拥有的连接，经过一段时间优雅地关闭（如果可能的话）剩余的连接。逐出时间通过 [`--drain-time-s`](http://www.servicemesher.com/envoy/operations/cli.html#cmdoption-drain-time-s) 选项设置。

**注意**

- Envoy 从 1.9 版本开始已不再支持 v1 API。
- [v2 LDS API](http://www.servicemesher.com/envoy/configuration/overview/v2_overview.html#v2-grpc-streaming-endpoints)

## 统计

LDS 的统计树是以 `listener_manager.lds` 为根，统计如下：

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

- [Listener discovery service(LDS) - envoyproxy.io](https://www.envoyproxy.io/docs/envoy/latest/configuration/listeners/lds)
- [监听器发现服务（LDS）- servicemesher.com](http://www.servicemesher.com/envoy/configuration/listeners/lds.html)