---
owners: ["rootsongjc"]
reviewers: ["tangyong"]
description: "本文介绍了服务网格架构。"
publishDate: 2019-03-10
updateDate: 2019-03-14
tags: ["service mesh"]
category: "evolution"
---

# 服务网格架构

下图是[Conduit](https://condiut.io) Service Mesh（现在已合并到Linkerd2中了）的架构图，这是Service Mesh的一种典型的架构。

![服务网格架构示意图](https://ws2.sinaimg.cn/large/0069RVTdly1fuail4d24jj31080rkgr7.jpg)

服务网格中分为**控制平面**和**数据平面**，当前流行的两款开源的服务网格 Istio 和 Linkerd 实际上都是这种架构，只不过 Istio 的划分更清晰，而且部署更零散，很多组件都被拆分，控制平面中包括 Mixer、Pilot、Citadel，数据平面默认是用 Envoy；而 Linkerd 中只分为 Linkerd 做数据平面，namerd 作为控制平面。

## 控制平面

控制平面的特点：

- 不直接解析数据包
- 与数据平面中的代理通信，下发策略和配置
- 负责网络行为的可视化
- 通常提供 API 或者命令行工具可用于配置版本化管理，便于持续集成和部署

## 数据平面

数据平面的特点：

- 通常是按照无状态目标设计的，但实际上为了提高流量转发性能，需要缓存一些数据，因此无状态也是有争议的
- 直接处理入站和出站数据包，转发、路由、健康检查、负载均衡、认证、鉴权、产生监控数据等
- 对应用来说透明，即可以做到无感知部署

## 参考

- [企业级服务网格架构之路解读](https://jimmysong.io/posts/the-enterprise-path-to-service-mesh-architectures/)

