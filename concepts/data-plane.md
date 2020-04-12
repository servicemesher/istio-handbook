---
owners: ["Liu-HongYe"]
reviewers: [""]
description: "讲述数据平面中envoy实现的架构"
publishDate: 2019-03-29
updateDate: 2019-03-29
tags: ["envoy","data plane","concepts"]
category: "original"
---

# 数据平面介绍

Istio 数据平面核心是以边车(sidecar)模式运行的职能代理.
这些代理可以调节和控制微服务之间所有的网络通信. 每个服务Pod启动时会伴随启动Istio-init和proxy容器.
其中Istio-init容器主要功能是初始化Pod网络和对Pod设置Iptable规则, 设置完成后自动结束.
Proxy容器启动两个服务: Pilot-agent以及网络代理组件. 
Pilot-agent的作用是同步管理数据, 启动并管理网络代理服务进程, 上报遥测数据.
网络代理组件则根据管理策略完成流量管控、生成遥测数据
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