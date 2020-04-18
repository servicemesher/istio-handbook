---
authors: ["wangfakang"]
reviewers: [""]
---

## MOSN 简介

MOSN（Modular Open Smart Network） 是一款使用 Go 语言开发的网络代理软件，作为云原生的网络数据平面，旨在为服务提供多协议，模块化，智能化，安全的代理能力。MOSN 可以与任何支持 xDS API 的 Service Mesh 集成，另外可以作为独立的四、七层负载均衡器，API Gateway，云原生 Ingress 等场景使用。

## 框架介绍

对 MOSN 有了初步了解后，接下来我们将从其功能特性、架构分层、内存及连接池这几方面来深入剖析一下 MOSN 。

### 功能特性

下图展示的是组成 MOSN 的各个模块：

![concepts-mosn-feature](../images/concepts-mosn-feature.png)

其中：

* Starter, Server, Listener, Config, xDS 为 MOSN 启动模块，用于完成 MOSN 的运行
* 最左侧的 Hardware, NET/IO, Protocol, Stream, Proxy 为 MOSN 架构 中介绍的 MOSN 核心模块，
用来完成 Service MESH 的核心功能
* Istio 集成
  * 支持 Istio Pilot V2 API，可基于全动态资源配置运行
* Router 为 MOSN 的核心路由模块，支持的功能包括：
  * VirtualHost 形式的路由功能
  * 基于 subset 的子集群路由匹配
  * 路由重试以及重定向功能
* Upstream 为后端管理模块，支持的功能包括：
  * Cluster 动态更新
  * Host 动态更新
  * 对 Cluster 的 主动/被动 健康检查
  * 熔断保护机制
  * CDS/EDS 对接能力
* Protocol 为 MOSN 提供多协议框架，可方便的扩展自定义协议，目前支持的协议包括：
  * HTTP/HTTP2 作为标准的HTTP系协议
  * SOFA/Dubbo 作为服务之间常用的RPC协议
* Metrics 模块可对协议层的数据做记录和追踪
  * Metrics当前统计了网络读写流量、请求状态、连接数等元数据
  * Trace 框架集成 SkyWalking 组件，可方便的观察请求的链路 
* 进程管理
  * 支持平滑 reload
  * 支持平滑升级，连接无损迁移
* LoadBalance 当前支持 RR, Random, Subset LB, Original_Dst 等负载均衡算法

### 架构解析

MOSN 延续 OSI(Open System Interconnection) 的分层思想，将其系统分为 NET/IO 、Protocol 、 Stream 、 Proxy 四层，如下图所示：

![concepts-mosn-arch](../images/concepts-mosn-arch.png)

其中：

* NET/IO 作为网络层，监测连接和数据包的到来，同时作为listener filter 和 network filter的挂载点
* Protocol 作为多协议引擎层，对数据包进行检测，并使用对应协议做 decode/encode 处理
* Stream 对 decode 的数据包做二次封装为 stream ，作为 stream filter的挂载点
* Proxy 作为 MOSN 的转发框架，对封装的 stream 做 proxy 处理

MOSN 整体框架采用分而治之的架构思想，其中每一层通过工厂设计模式向外暴露其接口，方便用户灵活的注册自身的需求，采用协程池的方式使得用户以同步的编码风格达到异步的异曲同工之妙。通过协程的类型 MOSN 中分为 read 和 proxy worker 两大类协程，read 协程主要是处理网络的读取及协议解析，proxy worker 协程用来完成读取后数据的加工、路由、转发等，其架构如下图所示：

![concepts-mosn-goroutine](../images/concepts-mosn-goroutine.png)

### 内存及连接池

MOSN 为了规避 Runtime GC 带来的卡顿自身做了内存池的封装方便多种对象高效的复用内存，另外为了提升服务网格之间的建连性能还设计了多种协议的连接池从而方便的实现连接复用及管理。

MOSN 的 Proxy 模块在 Downstream 收到 Request 的时候，在经过路由、负载均衡等模块处理获取到 Upstream Host 以及对应的转发协议时，会去 Cluster Manager 获取对应协议的连接池 ，如果连接池不存在则创建并加入缓存中，之后在长连接上创建 Stream，并发送数据。

如下图所示为连接池工作的示意图：

![concepts-mosn-connpool](../images/concepts-mosn-connpool.png)

MOSN 基于 sync.Pool 之上封装了一层自己的注册管理逻辑，可以方便的扩展各种类型的对象进行复用和管理。其中 bpool 是用来存储各类对象的构建方法，vpool 用来存放 bpool 中各个实例对象具体的值。运行时通过bpool里保存的构建方法来创建对应的对象通过index关联记录到vpool中，使用完后通过 sync.Pool 进行空闲对象的管理达到复用，如下图所示：

![concepts-mosn-mempool](../images/concepts-mosn-mempool.png)

## 应用场景

MOSN 不仅可以作为独立的四、七层负载均衡软件，也可以被集成到 Istio 里面的 Sidecar 或 Kubernetes 下的 Ingress Gateway 等使用，下面介绍两种主流的使用场景。

### sidecar

MOSN 作为 Istio 的数据面时，通过 xDS 协议和 istiod 组件进行通信，运行时动态获取服务的路由、监听、后端集群等配置信息，从而使得 MOSN 的服务治理能力可完美的和 Istio 结合。

![concepts-mosn-istio.png](../images/concepts-mosn-istio.png)

### Gateway

MOSN 提供负载均衡、路由管理、可观测性、多协议、配置热生效等通用Gateway的能力，使用者可以将其作为南北向的七层 Gateway 为服务提供流量管理和监控能力。

## 小结

本节中我们主要围绕 MOSN 是什么、可以做什么以及架构原理这几个方面做了介绍。在下一节中我们将会详细介绍下 MOSN 在 Service Mesh 中是如何配合 Istio 发挥数据面的能力。

## 参考资料

* [MOSN 官网博客](https://mosn.io/zh/blog/code/)
