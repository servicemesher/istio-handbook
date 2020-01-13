---
owners: ["rootsongjc"]
reviewers: [""]
description: "本文是流量管理的开篇"
publishDate: 2019-03-10
updateDate: 2019-03-10
tags: ["traffic management","index"]
category: "original"
---

# 流量管理

这一章节将带大家了解 Istio 流量管理中的各种概念的含义及表示方法。

流量管理是 Istio 中的最基础功能，使用 Istio 的流量管理模型，本质上是将流量与基础设施扩容解耦，让运维人员可以通过 Pilot 指定流量遵循什么规则，而不是指定哪些 pod/VM 应该接收流量——Pilot 和智能 Envoy 代理会帮我们搞定。

所谓流量管理是指：

- **控制服务之间的路由**：通过在 `VirtualService` 中的规则条件匹配来设置路由，可以在服务间拆分流量。
- **控制路由上流量的行为**：设定好路由之后，就可以在路由上指定超时和重试机制，例如超时时间、重试次数等；做错误注入、设置断路器等。可以由 `VirtualService` 和 `DestinationRule` 共同完成。
- **显式地向网格中注册服务**：显示地引入 Service Mesh 内部或外部的服务，纳入服务网格管理。由 `ServiceEntry` 实现。
- **控制网格边缘的南北向流量**：为了管理进入 Istio service mesh 的南北向入口流量，需要创建 `Gateway` 对象并与 `VirtualService` 绑定。

关于流量管理的详细介绍请参考 [Istio 官方文档](https://istio.io/zh/docs/concepts/traffic-management/)。

## 参考

- [流量管理 - istio.io](https://istio.io/zh/docs/concepts/traffic-management/)
