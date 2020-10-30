---
authors: ["violetgo"]
reviewers: [""]
---

# 扩展

一直以来，可扩展需求都是 Istio 的项目的基本规则。但是随着 Istio 的不断落地，更多的需求也在不断的涌现。Istio 社区自身和其他的重要生态公司也在不断的完善自身产品的架构和可扩展性。其中最具有代表性的就是 Istio 新发布的 WebAssembly 机制及为了解决南北向流量而诞生的各类边缘服务。


## Mixer 与 WebAssembly

在 Istio 项目的1.5版本发布之前， Istio 对于可扩展性的做法是启用一个通用的进程外扩展模型： Mixer ，以此带来轻量级的开发者体验。在 Mixer 的模型中，在每个请求执行前提条件检查之前，以及在每个请求报告遥测之后，Envoy sidecar 都会在逻辑上调用 Mixer。

Mixer 处理不同基础架构后端的灵活性来自其通用插件模型。 每个插件都被称为适配器 ，它们允许 Mixer 与提供基础功能（例如日志记录、监视、配额、ACL 检查等）的不同基础架构后端进行交互。运行时使用的适配器的确切集合是通过配置确定的，可以轻松扩展以针对新的或定制的基础架构后端。

自 2016 年使用 Envoy 以后，Istio 项目一直想提供一个平台，在此平台上可以构建丰富的扩展，以满足用户多样化的需求。同样的，Envoy 也一直把可扩展性作为一个基本原则。但是 Envoy 采取了不同的实现方式，他更注重代理内的扩展。

Envoy 提供了一个特殊的 Http 七层 filter，名为 WASM ，用于载入和执行 WASM 字节码。对于每一个 WASM 扩展插件都可以被编译为一个 `*.WASM` 文件，而 Envoy 七层提供的 wasm Filter 可以通过动态下发相关配置（指定文件路径）使其载入对应的文件并执行。

由于 Mixer 的模型会导致明显的资源效率低下，从而影响性能和资源利用率，所以该模型在根本上来说是有局限性的。于是在 Istio 1.5 的版本中，Mixer 被废弃了。取而代之的是基于 Wasm sandbox API 的代理内扩展。他使用 类似Envoy 的WASM机制来扩展插件：兼顾性能、多语言支持、动态下发动态载入、以及安全性。

新版本中，HTTP 遥测默认基于 in-proxy Stats filter，这节省了 50% 的 CPU 使用量，可以说影响老版本性能的罪魁祸首Mixer最被社区废弃。而 Mixer V2 其最终目标就是将现有的 out-of-process 的插件模型最终用基于 WASM 的 in-proxy 扩展模型来替代。




## Ambassador && Contour

在解决了 Istio 的内部扩展问题后，以 Istio 为代表的 Service Mesh 实际上就可以有效的解决“服务异构化”、“动态化”、“多协议”场景所带来的“东西向”流量的管控问题。但是对于南北向流量的控制，仅仅提供了 ingress/egress 做流量入口，出口的管理。

为了解决云原生环境下的南北向流量控制问题，Contour(https://projectcontour.io/) 和 Ambassador（https://www.getambassador.io) 开始走入大家的视线。

本质上，Contour 和 Ambassador 都是基于 Envoy 的 Ingress 的控制器，但是 Contour 使用了官方的 Envoy 镜像，而 Ambassador 选择了自定义镜像。所以 Ambassador 在性能和支持的协议类型上，要略优于 Contour 。

类似Contour 和 Ambassador 的扩展可以作为网关，部署在网络边缘，将传入网络的流量路由到相应的内部服务。在这种部署方式下，来自集群外部的入站流量首先会经过网关，再由网关将流量路由到 Istio。在网关处，可以主要处理认证、边缘路由、TLS 解密，等功能。

这种部署方式能让运维人员得到一个高性能、现代化的边缘服务与最先进的服务网格>（Istio）相结合的网络。




## 小结

在 Istio 社区内部，已经意识到了 Mixer 的 out-process 的处理方式的不足，虽然在1.5的release中，已经扩展了 WASM 模型，但是目前依然有很长的一段路要走，毕竟即使 Istio 社区本身的插件，也未能完全在 WASM 沙箱中落地。

而对于 Istio 不擅长的南北流量部分，目前 Ingress 功能不足，我们可能需要的不仅仅是一个引流功能，而是希望能够支持灰度，影子流量，更多的协议，熔断，限流等服务治理功能。

总而言之，Service Mesh 还在积极落地中，这里还没有一个明显的赢家，社区和各家公司都也在积极发展各自的技术，一切皆未尘埃落定。


## 参考
- [重新定义代理的扩展性](https://istio.io/latest/zh/blog/2020/wasm-announce/)
- [拥抱变化 —— Istio 1.5 新特性解读](https://zhuanlan.zhihu.com/p/112050345)
- [Deprecating Mixer](https://docs.google.com/document/d/1x5XeKWRdpFPAy7JYxiTz5u-Ux2eoBQ80lXT6XYjvUuQ/edit#)
- [Istio1.5 & Envoy 数据面 WASM 实践](https://blog.csdn.net/sD7O95O/article/details/105851877)
- [Ambassador和Istio：边缘代理和服务网格](https://kuaibao.qq.com/s/20180122G0N4PC00?refer=cp_1026)
- [8款开源的Kubernetes Ingress Controller/API Gateway推荐](https://www.servicemesher.com/blog/nginx-ingress-vs-kong-vs-traefik-vs-haproxy-vs-voyager-vs-contour-vs-ambassador/)
- [在Kubernetes上对Ambassador，Contour和Nginx性能进行基准测试](https://segmentfault.com/a/1190000022339033)
