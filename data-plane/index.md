---
owners: ["SataQiu"]
reviewers: ["rootsongjc"]
description: "本文是数据平面的开篇。"
publishDate: 2019-03-10
updateDate: 2019-03-13
tags: ["index","data plane"]
category: "original"
---

# 数据平面介绍

数据平面由一组以 sidecar 方式部署的智能代理组成。
这些代理可以调节和控制微服务之间所有的网络通信。
数据平面真正触及到对网络数据包的相关操作，是上层控制平面策略的具体执行者。

在服务网格中，数据平面 sidecar 代理主要负责执行如下任务：

- 服务发现：探测所有可用的上游或后端服务实例
- 健康检测：探测上游或后端服务实例是否健康，是否准备好接收网络流量
- 流量路由：将网络请求路由到正确的上游或后端服务
- 负载均衡：在对上游或后端服务进行请求时，选择合适的服务实例接收请求，同时负责处理超时、断路、重试等情况
- 身份验证和授权：对网络请求进行身份验证、权限验证，以决定是否响应以及如何响应，使用 mTLS 或其他机制对链路进行加密等
- 链路追踪：对于每个请求，生成详细的统计信息、日志记录和分布式追踪数据，以便操作人员能够理解调用路径并在出现问题时进行调试

简单来说，数据平面就是负责有条件地转换、转发以及观察进出服务实例的每个网络包。

典型的数据平面实现有：[Linkerd](https://linkerd.io/)、[NGINX](https://www.nginx.com/)、[HAProxy](https://www.haproxy.com/)、[Envoy](https://envoyproxy.github.io/)、[Traefik](https://traefik.io/)

## 参考

- [What is Istio?](https://istio.io/docs/concepts/what-is-istio/)
- [Service mesh data plane vs. control plane](https://blog.envoyproxy.io/service-mesh-data-plane-vs-control-plane-2774e720f7fc)