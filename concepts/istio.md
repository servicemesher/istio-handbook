---
authors: ["linda01232003"]
reviewers: [malphi","rootsongjc"]
---

#  什么是 Istio
简单来说，Istio 是一种 Service Mesh 的解决方案，它针对现有的服务网络，提供了一种简单的方式将连接、安全、控制和观测的模块，与应用程序或服务隔离开来，从而使得开发人员可以将更多的精力放在核心的业务逻辑上。

//todo

##   Istio 的平台支持
Istio 独立于平台，被设计为可以在各种环境中运行，包括跨云、内部环境、Kubernetes 等等。您可以在 Kubernetes 或是装有 Consul 的 Nomad 环境上部署 Istio。Istio 目前支持：
1. Kubernetes 上的服务部署
1. 基于 Consul 的服务注册
1. 服务运行在独立的虚拟机上
