# <svg class="icon"><use xlink:href="https://preliminary.istio.io/img/icons.svg#glossary"></use></svg>术语表
# [A](#a) | [B](#b) | [C](#c) | [D](#d) | [E](#e) | [F](#f) | [G](#g) | [H](#h) | [I](#i) | [J](#j) | [K](#k) | [L](#l) | [M](#n) | [N](#n) | [O](#o) | [P](#p) | [Q](#q) | [R](#r) | [S](#s) | [T](#t) | [U](#u) | [V](#v) | [W](#w) | [X](#x) | [Y](#y) | [Z](#z)  
## A
##### ADAPTERS
适配器（adapter）是 Istio 策略和遥测组件 [Mixer](#mixer) 的插件, 可使其与一组开放式基础架构后端交互，这些后端可提供核心功能，例如日志记录、监控、配额、ACL 检查等等。运行时所使用的精确的适配器集合是通过配置确定的，并可以针对新的或定制的基础架构后端轻松扩展。
##### ANNOTATION
注释是指附加到 [Kubernetes annotation](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) 的资源，例如 pod。有关 Istio 特定注释的有效列表，请参见 [Resource Annotations](https://preliminary.istio.io/zh/docs/reference/config/annotations/)。
##### ATTRIBUTE
属性控制着网格中服务运行时的行为，是一堆有名字的、有类型的元数据，它们描述了 ingress 和 egress 流量，以及这些流量所在的环境。
一个 Istio 属性包含了一段特定的信息，例如 API 请求的错误代码、API 请求的延迟或 TCP 请求的原始 IP 地址。例如：

```
request.path: xyz/abc
request.size: 234
request.time: 12:34:56.789 04/17/2017
source.ip: 192.168.0.1
destination.workload.name: example
```

属性被 Istio 的策略和遥测功能所使用。
## C
##### CLUSTER
集群是运行容器化应用程序的一组计算节点。 通常，组成集群的计算节点彼此可以直接连接。 集群通过规则或策略限制外部访问。
##### CONTROL PLANE
控制平面是一组系统服务，这些服务配置网格或者网格的子网来管理工作负载实例之间的通信。 单个网格中控制平面的所有实例共享相同的配置资源。
##### CRDS
[自定义资源定义 (CRD)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
是默认的 Kubernetes API 扩展。Istio 使用 Kubernetes CRD API 来配置，即使是非 Kubernetes 环境下部署的 Istio。
## D
##### DATA PLANE
数据平面是网格的一部分，直接控制工作负载实例之间的通信。 Istio 的数据平面使用智能 Envoy 代理部署成 sidecar 去调节和控制服务网格中发送和接受的流量。
##### DESTINATION
目标服务 (destination) 是 [envoy](#envoy) 代表一个[源服务](#source) [工作负载](#workload)与之打交道的远程上游服务。这些上游服务可以有多个[服务版本](#service)，envoy 根据路由选择对应的版本。
## E
##### ENVOY
Envoy 是在 Istio 里使用的高性能代理，用于为所有[服务网格](#service-mesh)里的[服务](#service)调度进出的流量。
[了解更多关于 Envoy](https://envoyproxy.github.io/envoy/)。
## F
##### FAILURE DOMAIN
故障域是计算环境中物理或者逻辑的一部分，当关键设备或服务遇到问题时，它也会受到负面影响。

对于 Istio 部署而言，故障域可能包含平台中的多个可用性区域。
## I
##### ISTIO
官网翻译为连接、安全加固、控制和观察服务的开放平台。开放平台就是指它本身是开源的，服务对应的是微服务，也可以粗略地理解为单个应用。
##### IDENTITY
身份是基本的安全基础结构概念。Istio 的身份模型是基于第一阶级的工作负载身份。在服务之间的通信开始时，双方使用身份信息交换证书来实现相互认证的目的。

客户端根据其安全的命名信息检查服务器的身份，以便确定服务器是否被授权运行服务。

服务器检查客户端的身份，以确定客户端可以访问的信息。服务器基于客户端的身份，来确定配置的策略。

通过使用身份，服务器可以审核访问信息的时间和特定客户端访问的信息内容。还可以根据客户使用的服务向他们收费，并拒绝任何未付款的客户访问服务。

Istio 身份模型非常灵活，粒度足以代表单个用户、单个服务，或者一组服务。在没有第一阶级服务身份的平台，Istio 可以使用其他的身份为服务实例进行分组，例如服务名称。
Istio 在不同的平台上支持以下服务身份：
- Kubernetes: Kubernetes 服务账户
- GKE/GCE: GCP 服务账户
- GCP: GCP 服务账户
- AWS: AWS IAM 用户/角色 账户
- 本地 （非 Kubernetes）：用户账户、客户服务账户、服务名称、Istio 服务账户，或者 GCP 服务账户。 客户服务账户指现有的服务账户，就像客户身份目录中管理的身份。

通常，[信任域](#trust-domain)指定身份所属的网格。
## M
##### MANAGED CONTROL PLANE
托管控制平面是一个为客户提供管理的[控制平面](#control-plane)。托管控制平面降低了用户部署的复杂性，并通常保证一定水平的性能和可用性。
##### MESH FEDERATION
网格联邦是在网格之间公开服务的一种行为，并且能跨越网格边界进行通信。每一个网格或许会公开其一部分的服务，使一个或多个其他网格使用此公开的服务。您可以使用网格联邦来启用网格之间的通信，可参阅[多个网格部署](#multiple-meshes)。
##### MICRO-SEGMENTATION
Micro-segmentation 是一种安全技术，可在云部署中创建安全区域，使组织能够将工作负载彼此隔离，并分别保证它们的安全。
##### MIXER
Mixer 是 Istio 里的一个组件，它负责增强[服务网格](#service-mesh)里的访问控制和使用策略。它还负责收集来自 [envoy](#envoy) 和其他服务的遥测数据。
##### MIXER HANDLER
Handler 相当于配置完备的 Mixer 适配器。一个适配器二进制文件可以被不同配置使用，这些配置也可以称为 handler。在 Mixer 运行时，Mixer 将 [instances](#mixer-instance) 路由到一个或多个 handlers。
##### MIXER INSTANCE
Mixer Instance 表示通过检查一组请求[属性](#attribute) ，并结合使用者提供的配置而生成的数据块。Mixer Instance 在随请求到达各种基础后端设施的途中，会被传递给各个[处理程序](#mixer-handler)。
##### MULTI-MESH
Multi-mesh 是由两个或多个[服务网格](#service-mesh)组成的部署模型。每个网格都有独立的命名管理和身份管理，但是您可以通过[网格联邦](#mesh-federation)来暴露网格之间的服务, 最终构成一个多网格部署。
##### MULTICLUSTER
Multicluster 是一种部署模型，由具有多个[集群](#cluster)的[网格](#service-mesh)组成。
##### MUTUAL TLS AUTHENTICATION
双向 TLS 通过内置身份和凭证管理，提供强大的服务到服务身份验证。 了解更多关于双向 TLS 身份验证。
## O
##### OPERATOR
Operator 是打包，部署和管理 Kubernetes 应用程序的一种方法。有关更多信息，请参见 [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)。
## P
##### PILOT
Pilot 是 Istio 里的一个组件，它控制 [Envoy](#envoy) 代理，负责服务发现、负载均衡和路由分发。
##### POD
Pod 中包含了一个或多个共享存储和网络的容器 （例如 [Docker](https://www.docker.com/) 容器），
以及如何运行容器的规范。Pod 是 Istio 的 [Kubernetes](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/) 部署中的一个[工作负载实例](#workload-instance)。
## R
##### ROUTING RULES
您在[虚拟服务](#virtual-services)中配置的路由规则，遵循服务网格定义了请求的路径。 使用路由规则，您可以定义将寻址到虚拟服务主机的流量路由到指定目标的工作负载。 路由规则使您可以控制流量，以实现如 A/B 测试、金丝雀发布以及按百分比分配流量的分阶段发布等任务。
## S
##### SERVICE
使用[服务名称](#service-name)标识一组具有关联行为的服务[服务网格](#service-mesh)，并使用这些名称应用 Istio 策略（例如负载均衡和路由）。服务通常由一个或多个[服务 Endpoint](#service-endpoint) 实现，并且或许包含多个[服务版本](#service-version)。
##### SERVICE CONSUMER
服务消费者是使用 [service](#service) 的代理。
##### SERVICE ENDPOINT
Service Endpoint 是一个 [service](#service) 的网络可达表现形式。Service endpoint 由[工作负载实例](#workload-instance)暴露，但并不是所有的服务都有 service endpoint。
##### SERVICE MESH
*服务网格* （简称 *网格* ）是一个可管理、可观测以及支持[工作负载实例](#workload-instance)之间进行安全通信的基础设施层。

在一个网格中，服务名称与命名空间组合具有唯一性。例如，在一个[多集群](#multicluster)的网格中，`cluster-1` 集群的 `foo` 命名空间中的 `bar` 服务和 `cluster-2` 集群的 `foo` 命名空间中的 `bar` 服务被认为是同一个服务。

由于服务网格会共享这种[标识](#identity)，因此同一服务网格内的[工作负载实例](#workload-instance)可以相互认证通信。
##### SERVICE NAME
Service Name 是 [service](#service) 唯一的名字，是 [service](#service) 在 [service mesh](#service-mesh) 里的唯一标识。一个服务不应该被重命名，或者维护它的标识，每一个服务名都是唯一的。一个服务有多个 [versions](#service-version)，但是服务名是与版本独立的。
##### SECURE NAMING
Secure Naming 提供一个 [service name](#service-name) 到 [workload instance principals](#workload-instance-principal) 的映射，这个工作负载实例被授权运行一个 [workload instances](#workload-instance)，实现一个 [service](#service)。
##### SERVICE OPERATOR
Service Operator 是在 [service mesh](#service-mesh) 里管理 [service](#service) 的代理，它们通过操纵配置状态并通过各种仪表板监视服务的运行状况来管理这些服务。
##### SERVICE PRODUCER
创建[服务](#service)的 pilot-agent。
##### SERVICE REGISTRY
Istio 维护了一个内部服务注册表 (service registry)，它包含在服务网格中运行的一组[服务](#service)及其相应的[服务 endpoints](#service-endpoint)。Istio 使用服务注册表生成 [Envoy](#envoy) 配置。

Istio 不提供[服务发现](https://en.wikipedia.org/wiki/Service_discovery)，尽管大多数服务都是通过 [Pilot](#pilot) adapter 自动加入到服务注册表里的，而且这反映了底层平台（Kubernetes、Consul、plain DNS）的已发现的服务。还有就是，可以使用 [`ServiceEntry`](#service-entries) 配置手动进行注册。
##### SERVICE VERSION
区分一系列[服务](#service)，通常通过[工作负载](#workload)二进制文件的不同版本来帮助确定。在一些场景多服务版本是需要的，比如 A/B 测试和金丝雀发布。
##### SOURCE
Source 是 [Envoy](#envoy) 代理的下游客户端。在[服务网格](#service-mesh)里，source 通常是一个[工作负载](#workload)，但是入口流量的 source 有可能包含其他客户端，例如浏览器，或者一个移动应用。
## T
##### TLS ORIGINATION
TLS 源（TLS Origination）发生于一个被配置为接收内部未加密 HTTP 连接的 Istio 代理（sidecar 或 egress gateway）加密请求并使用简单或双向 TLS 将其转发至安全的 HTTPS 服务器时。这与 [TLS 终止](https://en.wikipedia.org/wiki/TLS_termination_proxy)相反，后者发生于一个接受 TLS 连接的 ingress 代理解密 TLS 并将未加密的请求传递到网格内部的服务时。
##### TRUST DOMAIN
[信任域](https://spiffe.io/spiffe/concepts/#trust-domain)对应于系统的信任根，并且是工作负载标识的一部分。

Istio 使用信任域在网格中创建所有[身份](#identity)。每个网格都有一个专用的信任域。

例如在 `spiffe://mytrustdomain.com/ns/default/sa/myname` 中标示网格的子字符串是：`mytrustdomain.com`。此子字符串是此网格的信任域。
##### TRUST DOMAIN MIGRATION
更改 Istio 网格的[信任域](#trust-domain)的过程。
## W
##### WORKLOAD
[operators](#operator) 部署的二进制文件，用于提供服务网格应用的一些功能。工作负载有自己的名称，命名空间，和唯一的 id。这些属性可以通过下面的[属性](#attribute)被策略配置和遥测配置使用：

* `source.workload.name`, `source.workload.namespace`, `source.workload.uid`
* `destination.workload.name`, `destination.workload.namespace`, `destination.workload.uid`

在 Kubernetes 环境中，一个工作负载通常对应一个 Kubernetes deployment，
并且一个[工作负载实例](#workload-instance)对应一个独立的被 deployment 管理的 [pod](#pod)。
##### WORKLOAD INSTANCE
工作负载实例是[工作负载](#workload)的一个二进制实例化对象。
一个工作负载实例可以开放零个或多个[服务 endpoint](#service-endpoint)，
也可以消费零个或多个[服务](#service)。

工作负载实例具有许多属性：

- 名称和命名空间
- 唯一的 ID
- IP 地址
- 标签
- 主体

通过访问 `source.*` 和 `destination.*` 下面的属性，在 Istio 的策略和遥测配置功能中，可以用到这些属性。
##### WORKLOAD INSTANCE PRINCIPAL
工作负载实例主体是[工作负载实例](#workload-instance)的可验证权限。Istio 的服务到服务身份验证用于生成工作负载实例主体。默认情况下，工作负载实例主体与 SPIFFE ID 格式兼容。

在 `policy` 和 `telemetry` 配置中用到了工作负载实例主体，对应的[属性](#attribute)是 `source.principal` 和 `destination.principal`。