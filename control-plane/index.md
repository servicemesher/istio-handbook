---
owners: ["zhongfox"]
reviewers: ["zhongfox"]
description: "本文是展示了istio组件进程级别的拓扑图。"
publishDate: 2019-07-15
updateDate: 2019-05-13
tags: ["index","control plane"]
category: "original"
---

# 组件概览

## istio 组件构成

以下是istio 1.1 官方架构图:

![Istio 架构图](https://gw.alipayobjects.com/mdn/rms_91f3e6/afts/img/A*_GPdTIDlhD4AAAAAAAAAAABkARQnAQ)

虽然 Istio 支持多个平台，但将其与 Kubernetes 结合使用，其优势会更大，Istio 对 Kubernetes 平台支持也是最完善的，本文将基于 Istio + Kubernetes 进行展开。

在安装了grafana, prometheus, kiali, jaeger等后端组件的情况下, 一个完整的控制面组件应该包括以下pod：

```bash
% kubectl -n istio-system get pod
NAME                                          READY     STATUS
grafana-5f54556df5-s4xr4                      1/1       Running
istio-citadel-775c6cfd6b-8h5gt                1/1       Running
istio-galley-675d75c954-kjcsg                 1/1       Running
istio-ingressgateway-6f7b477cdd-d8zpv         1/1       Running
istio-pilot-7dfdb48fd8-92xgt                  2/2       Running
istio-policy-544967d75b-p6qkk                 2/2       Running
istio-sidecar-injector-5f7894f54f-w7f9v       1/1       Running
istio-telemetry-777876dc5d-msclx              2/2       Running
istio-tracing-5fbc94c494-558fp                1/1       Running
kiali-7c6f4c9874-vzb4t                        1/1       Running
prometheus-66b7689b97-w9glt                   1/1       Running
```

将istio系统组件细化到进程级别, 大概是这个样子：

![Istio 组件图](https://gw.alipayobjects.com/mdn/rms_91f3e6/afts/img/A*cbxAQYM5DrgAAAAAAAAAAABkARQnAQ)

Service Mesh 的 Sidecar 模式要求对数据面的用户 Pod 进行代理的注入, 注入的代理容器会去处理服务治理领域的各种「脏活累活」, 使得用户容器可以专心处理业务逻辑。

从上图可以看出, Istio 控制面本身就是一个复杂的微服务系统, 该系统包含多个组件 Pod, 每个组件各司其职, 既有单容器 Pod, 也有多容器 Pod, 既有单进程容器, 也有多进程容器, 每个组件会调用不同的命令, 各组件之间会通过 RPC 进行协作, 共同完成对数据面用户服务的管控。

## Istio 源码, 镜像和命令

Istio 项目代码主要由以下两个 git 仓库组成:

| 仓库地址                       | 语言 | 模块                                                                            |
|--------------------------------|------|---------------------------------------------------------------------------------|
| https://github.com/istio/istio | Go   | 包含 istio 控制面的大部分组件: pilot, mixer, citadel, galley, sidecar-injector 等  |
| https://github.com/istio/proxy | C++  | 包含 istio 使用的边车代理, 这个边车代理包含 envoy 和 mixer client 两块功能          |
| https://github.com/istio/api   | Go   | 包含 istio 组件之间的 API 以及资源配置定义, 使用 protobuf 进行定义                 |

### istio/istio

https://github.com/istio/istio 包含的主要的镜像和命令:

| 容器名                   | 镜像名                 | 启动命令          | 源码入口                          |
| ------------------------ | ---------------------- | ----------------- | --------------------------------- |
| Istio_init               | istio/proxy_init       | istio-iptables.sh | istio/tools/deb/istio-iptables.sh |
| istio-proxy              | istio/proxyv2          | pilot-agent       | istio/pilot/cmd/pilot-agent       |
| sidecar-injector-webhook | istio/sidecar_injector | sidecar-injector  | istio/pilot/cmd/sidecar-injector  |
| discovery                | istio/pilot            | pilot-discovery   | istio/pilot/cmd/pilot-discovery   |
| galley                   | istio/galley           | galley            | istio/galley/cmd/galley           |
| mixer                    | istio/mixer            | mixs              | istio/mixer/cmd/mixs              |
| citadel                  | istio/citadel          | istio_ca          | istio/security/cmd/istio_ca       |

另外还有2个命令不在上图中使用:

| 命令       | 源码入口                      | 作用                                                                   |
|------------|-------------------------------|------------------------------------------------------------------------|
| mixc       | istio/mixer/cmd/mixc          | 用于和 Mixer server 交互的客户端                                        |
| node_agent | istio/security/cmd/node_agent | 用于 node 上安装安全代理, 这在 Mesh Expansion 特性中会用到, 即 k8s 和 vm 打通  |

### istio/proxy

https://github.com/istio/proxy 该项目本身不会产出镜像, 它可以编译出一个 `name = "Envoy"` 的二进制程序, 该二进制程序会被加入到 istio 的边车容器镜像 `istio/proxyv2` 中。

istio proxy 项目使用的编译方式是 Google 出品的 bazel , bazel 可以直接在编译中引入第三方库，加载第三方源码。

这个项目包含了对 Envoy 源码的引用，还在此基础上进行了扩展，这些扩展是通过 Envoy filter（过滤器）的形式来提供，这样做的目的是让边车代理将策略执行决策委托给 Mixer，因此可以理解为 istio proxy 这个项目有两大功能模块:

1. Envoy: 使用到 Envoy 的全部功能。
2. mixer client: 策略和遥测相关的客户端实现, 基于 Envoy 做扩展，通过 RPC 与 Mixer server 进行交互, 实现策略管控和遥测。

### istio/api

https://github.com/istio/api 使用 [protobuf](https://github.com/protocolbuffers/protobuf) 对 Istio API 和资源进行定义, 包括:

* 组件之间的 API, 如 Mesh Configuration Protocol (MCP) 等。
* 所有的 Istio CRD, 如 VirtualService、DestinationRule 等。

该项目会作为依赖包被 istio 主项目引用。

### 更多

后续将对以上各个模块、命令以及它们之间的协作进行分析。
