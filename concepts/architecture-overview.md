---
authors: ["linda01232003，AsCat"]
reviewers: ["rootsongjc "]
---

# 架构解析

前文中讲述了 Service Mesh 的基本概念以及 Service Mesh 与 Kubernetes 的关联关系，并且对其进行了对比，同时也提到了 Istio 是一个功能丰富的 Service Mesh 的解决方案。

本节重点说明以下内容：
- 介绍 Istio 跟 Service Mesh 的关系
- 介绍 Istio 的发展历程
- 介绍 Istio 核心功能以及包含的组件

##  Istio 跟 Service Mesh 的关系

对 Service Mesh 有初步了解的人们往往不能避免提到 Istio 社区以及它的架构及实现，实际上 Istio 是 Service Mesh 的其中一个解决方案。实际上不少公司也同时在开展 Service Mesh 的解决方案，其中第一代 Service Mesh 以 Linkerd 为代表。

Linkerd 作为业界第一个开源 Service Mesh 的解决方案，其作者为云原生软件公司 Buoyant 公司 的 CEO William Morgan。同时他也是 service mesh 的布道师和践行者。Linkerd 是用 JVM 语言编写的 ，因此内存开销比较大，并且也不代理 TCP 请求，也不支持 websockets，当然它也有自己的优势： 应付大规模访问时，Linkerd 拥有绝对惊人的流量控制 。

第二代 Service Mesh 以 Istio 为代表，主要改进集中在更为强大的控制平面功能。Istio 是目前最主流的 Service Mesh 方案，也是国内外认可度最高的 Service Mesh 标准。它集成了 Envoy 作为 sidecar 的实现方案，而其他的控制平面组件都是用 Go 语言编写。它的设计理念非常先进，并且有大厂背书，因此作为最主流的 Service Mesh 解决方案当之无愧。

##  Istio 发展历程

Istio 是由 Google、IBM 和 Lyft 联合开发的。其中它的数据平面直接使用了 Lyft 的 Envoy 作为 sidecar。控制平面是由 Google、IBM 联合开发。其第一个版本 0.1 release 发布于2017年5月24日。截止到2020年4月份已经发布了1.5.1版本。其中1.4之前 Istio 的架构还是基本沿用一致的架构设计，仅仅是在功能上做增强以及增加一些辅助的组件。1.5 版本后将原有的 Istio 架构设计重新推倒，将之前的组件合并为一个单体架构 Istiod。然后其组件虽然精简成单体架构，但是之前支持的功能却一个都没有少。因此1.5版本作为 Istio 社区的一个全新形态，堪称是一次突破性的设计革新。

1.4 版本之前的Istio控制平面组件架构图如下所示：

![Istio 控制平面组件架构图](../images/istiofeatures2-1.4.png)

1.5 版本回归单体后的组件架构图：

![Istio 控制平面单体架构图](../images/istiofeatures3-istio1.5.png)

由于回归单体后功能全部都集中于 istiod 组件，对于初次接触 istio 的开发人员反而不利于了解内部的设计演进和开发思维，因此下面的核心功能和组件介绍还是以1.4版本之前的组件架构来介绍。

##  Istio核心功能和组件介绍

Istio 的架构可以以 1.5 版本作为分水岭。

### Istio 1.5 版本之前的架构

上小节提到了，Istio 分为控制平面和数据平面，控制平面在1.4版本之前的组件架构分为以下几个核心组件：
- Pilot ：Istio 的核心组件，主要负责核心治理、流量控制、将配置转换为数据平面可识别的 xDS 协议分发配置到数据平面。
- Galley ：主要负责配置校验、以及配置/服务发现。这两项功能在 galley 的架构设计中是相对独立的，对应的是galley的两种运行模式 server 和 enable-validation，两种模式可以单独开启也可以共同开启。
- Citadel ：可选开启或关闭，负责安全相关的证书和密钥管理。
- Injector ：负责数据平面的初始化相关的动作，例如自动注入 sidecar 就是使用该组件完成的。
- Mixer  ：默认关闭，负责提供策略控制和遥测收集的组件，内部包含 Policy 和 Telemetry 2个子模块，其中 Policy 负责在服务相互调用过程中对请求进行策略检查，例如鉴权、限流，而 Telemetry 负责监控相关的采集数据的信息聚合以用于对接各种后端。
- Istio数据平面  ：Istio 的数据平面主流选用 Lyft 的 Envoy，当然也可以选择其他的数据平面，例如 MOSN。

其中 Pilot 和 Galley 2个模块联合运作，可以完成从用户提交配置到校验、下发的一系列流程，例如可见以下流程图：

![Istio pilot galley流程图](../images/istiofeatures4-istio-pilotgalley.png)

1. 当用户向 Kubernetes 提交一份新的配置，首先会触发 galley 注册在 kubernetes  中的 webhook，webhook 会检查配置是否合法，如图中的步骤1。
1. 若配置无法通过校验，则 kubernetes 将拒绝用户提交的配置，并给出相应的错误信息。如图步骤2。
1. 当配置通过校验后，通过 kubernetes  的通知机制，galley 得到配置变更信息，如图中的步骤3。
1. Galley 将变更的配置/服务信息转换为 MCP 的格式通过 MCP 协议推送给 pilot，如图步骤4。
1. 最后一步，pilot 通过 xDS 协议向数据平面推送变更的配置。 

### Istio 自 1.5 版本以后的架构

Istio 1.5 版本后的一体化设计虽然说是 Istio 社区的一个全新形态，并不是完全推翻之前的设计，简单地说，只是将原有的多进程设计模式优化成了单进程的形态，之前各个组件被设计成了 istiod 的内部子模块而已，因此下面的职责，都将由 istiod 来承担；
- 监听配置，接手了原来 galley 的工作，负责监听来自多种支持配置源的数据，比如 Kubernetes  API server，本地文件或者基于 MCP 协议的配置，包括原来 galley 需要处理的 API 校验和配置的转发也需要设计在内；
- 监听 endpoint，监听来自本地或者远程集群的 endpoint 信息，在将来还会计划允许 endpoint 复制在各个集群内，包括非 Kubernetes  的注册中心例如 consul；
- CA 根的生成，生成私钥和证书，目前 Citadel 的职责之一；
- 控制平面身份标识，目前在内部控制平面之间是通过 CA 根来生成一个 SPIFFE ID 用于识别身份，也同时用于 injector 组件的证书生成；
- 证书生成，主要是为各个控制平面之间的通讯生成私钥和证书来保证安全通讯；
- 自动注入，在 Kubernetes  里需要为 Mutating Admission Controller 提供接口支持，从而可以在 pod 创建阶段修改资源文件来实现 sidecar 的自动注入；
- CNI/CRI  注入，通过 CNI/CRI 作为 hook 来实现自动注入的另一种方式，目前还没使用；
- Envoy 启动配置生成，上文目标中提到的，Envoy 的启动配置将由 istiod 来提供；
- 本地的 SDS Server，提供密钥发现服务的本地服务端；
- 中央 SDS Server，同上，是一个中央化的密钥发现服务端，一般用以对接第三方的密钥系统；
- xDS 服务提供，之前 pilot 的核心能力，为所有的 Envoy 提供 xDS 下发的服务端；

##  小结

以上分别介绍了 Istio 跟 Service Mesh 的关系、介绍了 Istio 的发展历程以及介绍 了 Istio 核心功能以及包含的组件。并且结合版本演进说明了 Istio 组件的变化。后续会通过对 Istio 内部核心组件分别进行分析，以便让读者们对 Service Mesh 以及 Istio 有深入的认识。

##  参考

- [Service Mesh 化繁为简 —— 基于 Istiod 回归单体设计](https://xw.qq.com/cmsid/20200322A06WDH00)
