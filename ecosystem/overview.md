---
authors: ["wuti1609"]
reviewers: ["rootsongjc"]
---

# Service Mesh 生态

在过去几年中，微服务成为了业界技术热点，大量的互联网公司都在使用微服务架构，也有很多传统企业开始实践互联网技术转型，基本上也是以微服务和容器为核心。伴随着这股微服务大潮，新一代的微服务开发技术 Service Mesh 也开始快速兴起。随着 Willian Morgan（Buoyant 公司 CEO）在 2017 年首次正式提出 Service Mesh 概念，不论是开源软件还是商业软件，在该领域都进行了大量投入，目前已形成不容忽视的庞大生态环境。  

## 核心开源项目

### Linkerd

Linkerd 技术最早由 Twitter 公司贡献，并由创业公司 Buoyant 主导开发和推广。作为最早期的 Service Mesh 技术提倡者，Linkerd 的发展可谓一波三折。Linkerd 经历了两个重要阶段，即早期的 Linkerd 和后期的 Linkerd2。

早期，Linkerd 是一个实现了服务网格设计的开源项目。其核心是一个透明代理，可以用它来实现专用的基础设施层以提供服务间的通信，进而为软件应用提供服务发现、路由、错误处理以及服务可见性等功能，而无需侵入应用内部本身的实现。

2017 年 Linkerd 加入 CNCF，对于 Service Mesh 技术是一个非常重要的历史事件，这代表社区对 Service Mesh 理念的认同和赞赏，Service Mesh也因此得到社区更大范围的关注。但是，随着 Istio 的出现和伴随名企光环的快速推进，Linkerd 的风光瞬间被盖过。2017 年 7 月 11 日，Linkerd 发布版本 1.1.1，宣布和 Istio 项目集成，希望借助 Istio 的势头重拾信心。但与 Istio 体系下的 Envoy 相比，Linkerd 基于 Scala 为主的技术栈，作为数据面代理并无优势。
	
![linkerd2](images/linkerd2.jpg)
	
2018 年 9 月，Buoyant 启动了 Linkerd 2.0，对自家的控制面项目 Conduit 进行整合，并基于 Golang 对 Linkerd 进行了重写，这让项目焕然一新。Linkerd2 采用了数据平面+控制平面的设计，也摆脱了长期困扰 Linkerd 的数据面性能问题，成为了 Istio 的有力竞争者。当然，项目方向的巨大调整也带来多方面阵痛，原本占得先机的 Linkerd 在转向 Linkerd2 后，稳定性和成熟度都有待验证。

### Envoy

Envoy 最初由 Lyft 创建，是一款开源、高性能的边缘、中间与服务代理。Envoy 的进程外架构适用于任何应用程序、语言和运行时，非常适合用于 Service Mesh 数据面；因此其也成为了 Istio 中 Sidecar 的官方标配。
	
![envoy](images/envoy.png)

在功能方面，Envoy 旨在实现服务与边缘代理功能，通过管理微服务之间的交互以确保应用程序性能。该项目提供的超时、速率限制、断路、负载均衡、重试、统计、日志记录以及分布式追踪等高级功能，可以帮助用户以高容错性和高可靠性的方式处理各类网络故障问题。Envoy 支持的协议与功能包括 HTTP/2、gRPC、MongoDB、Redis、Thrift、外部授权、全局速率限制以及一个富配置 API 等等。
	
在性能方面，Envoy 基于 C++ 实现，并采用了类Nginx的架构：多线程 + 非阻塞 + 异步IO（Libevent），虽然 Envoy 的设计者没有把极致性能作为目标，但作为数据面代理的 Envoy 仍然能够保证较低的延迟和高并发处理能力，并不断在此方向做出优化，这也是 Envoy 在社区中保持较高呼声的重要原因之一。
	
Envoy 近几年的发展非常稳健，一方面继续收获独立客户，一方面伴随 Istio 一起成长。作为业界仅有的两个生产级 Service Mesh 实现之一，Envoy 于 2017 年 9 月加入 CNCF，成为 CNCF 的第二个 Service Mesh 项目，并于 2018 年 8 月毕业，成为 CNCF 第三个毕业项目。
	
### Istio

Istio 作为本书的主角，2017 年 5 月 24 日由 IBM、Google 和 Lyft 联合宣布启动，项目发起方将 Istio 定位为一个连接、管理、监视和保护微服务的开放平台。

Istio 整体架构分为数据面和控制面。数据平面默认由 Envoy 提供，被部署为 sidecar，传输和控制网格内所有服务之间的网络通信。但随着 xDS 协议向标准化方向的推进，越来越多的数据面代理技术开始接入 Istio。控制面负责管理和配置代理，以及运行时执行的策略。最新的 1.5 版本中对控制面进行了较大的调整，将多个控制模块合而为一，旨为降低 Istio 架构长期被人诟病的复杂度和维护成本。

![istio](images/istio.png)

另一方面，根植于 Kubernetes 生态之上的 Istio 也在不断吸引优秀的项目融入其生态，比如使用 Kiali 或 Weave Scope 进行网格的可视化展示，使用 Prometheus 和 Grafana 为网格提供监控，使用 Zipkin 和 Jaeger 执行微服务之间的分布式跟踪，等等。

Istio 项目目前是 Service Mesh 领域最受瞩目的明星项目，有众多企业和开发者参与其中，项目本身仍处于高速发展阶段，建议企业用户在大规模实施前谨慎评估项目迭代可能带来的影响。
	
### 其他

除了以上较为广泛的项目，还有更多的开源项目参与到 Service Mesh 生态中，并向更多的细分领域进行深化。

**NginMesh**

2017 年 9 月，在美国波特兰举行的 nginx.conf 大会上，Nginx 宣布了 Nginmesh，并随即在 github 发布了 0.1.6 版本。

NginMesh 定位于 Istio 服务网格平台中的数据面代理。它旨在提供七层负载均衡和服务路由功能，与 Istio 集成作为 sidecar 部署，并将以“标准，可靠和安全的方式”使得服务间通信更容易。Nginmesh 提供了服务发现，请求转发，路由规则，性能指标收集等功能。但 NginMesh 的发展一直比较缓慢，目前它还没有应用到生产环境中。

**Consul connect**

2018 年 6 月 26 日，HashiCorp 发布了 HashiCorp Consul 1.2。这个版本主要新增了一个新的功能叫做 Connect, 它能够将现有的 Consul 集群自动转变为 Service Mesh 的解决方案。
	
Connect 通过自动 TLS 加密和基于鉴权的授权机制支持服务和服务之间的安全通信。它在设计开发时就贯注了易于使用的想法，可以仅仅通过一个配置参数打开，在服务注册时额外添加一行就可以使得任何现存的应用接受基于 Connect 的连接。证书更新是自动的，因此不会导致服务停机。对于所有必须的子系统，Connect 仅仅需要一个二进制文件就可以支持。凭借 Consul 的庞大用户群体，Consul connect 在发布后一直得到业界的广泛关注。

**MOSN**

2018 年 7 月，蚂蚁金服科技发布 MOSN。MOSN 是一个使用 Go 语言开发的数据面代理，为服务提供多协议、模块化、智能化、安全的代理能力。MOSN 可以与任何支持 xDS API 的 Service Mesh 集成，亦可以作为独立的四、七层负载均衡，API Gateway，云原生 Ingress 等使用。

MOSN 项目已于 2019 年 8 月加入 CNCF 基金会，由于有着蚂蚁和阿里的实力加成，目前该项目社区比较活跃。

**Kong kuma**

2019年9月10日，Kong正式宣布开源旗下 Service Mesh 项目 Kuma。Kuma 定位是一个通用的服务网格，不同于市场上的大多数服务网格项目，它的设计初衷是在 Kubernetes 生态系统内部和外部都能工作，这包括虚拟机、容器、遗留环境以及 Kubernetes。除此之外，Kuma 引入 Envoy 作为数据面代理，并引入自家的明星产品 Kong 作为 API 网关，该网关管理组织与部署现代微服务的各种方法之间的信息流。

## 商业化项目


除了蓬勃发展的开源生态，很多云厂商也在积极布局 Service Mesh。

### AWS

AWS APP Mesh 是 AWS 在 re:Invent 2018 大会上发布的一款新服务，并在 2019 年 4 月发布了正式版。App Mesh 作为 AWS 原生服务网格，与 AWS 现有产品簇完全集成，包括：

* 网络（AWS cloud map）
* 计算（Amazon EC2 和 AWS Fargate）
* 编排工具（AWS EKS，Amazon ECS 和 EC2 上客户管理的 Kubernetes）

AWS APP Mesh 旨在解决在 AWS 上运行的微服务的监控和控制问题，该服务将微服务之间的通信流程标准化，为用户提供了端到端的可视化界面，并且帮助用户应用实现高可用。App Mesh 使用开源的 Envoy 作为网络代理，这也使得它可以兼容一些开源的微服务监控工具。用户可以在 AWS ECS、EC2、EKS 或者 Fargate 上使用 App Mesh。鉴于 AWS 在公有云领域建立的先发优势，App Mesh 在商业化方面的市场占有量也保持领先。

### Google

Google 作为 Istio 的主要发起方之一，在 Service Mesh 方面投入了大量资源。在 Google Cloud 上就有 GKE mesh、Google Cloud Service mesh 和 Google Traffic Director 等多种产品。

* 2018 年底首先推出了 Istio on GKE，即“一键集成 Istio”，并提供遥测、日志、负载均衡、路由和 mTLS 安全能力。
* 接着 Google 又推出 Google Cloud Service Mesh，这是 Istio 的完全托管版本，不仅仅提供 Istio 开源版本的完整特性，还集成了 Google Cloud 上的重要产品 Stackdriver 。
* 2019 年 5 月，Google 推出 Traffic Director 测试版本，Traffic Director 是完全托管的服务网格流量控制平面，支持全局负载均衡，并且与 AWS Mesh 类似，适用于虚拟机和容器，提供混合云和多云支持、集中式健康检查和流量控制。

Google 在云原生领域进行了大量投入，尤其在开源生态，Google 始终主导着 Istio 发展，其中各种缘由使得该项目始终未进入 CNCF 基金会。但正是处于对 Kubernetes 以及相关生态的领先地位，使得 Google Cloud 在以容器为核心的云产品方面增速超过 AWS，而 Service Mesh 也正是其中的重要一环。

### Microsoft

微软早在 2018 年 10 月发布 Service Fabric Mesh 预览版，但时至今日，仍然没有看到正式版本的推出，相关资料也比较缺乏。反倒是在 2019 年 5 月，微软在 KubeConf 上高调推出 SMI（Service Mesh Interface）。

SMI 是一个在 Kubernetes 上运行服务网格的规范，用于定义其通用标准并由各种供应商实现。作为一个通用的行业规范/标准，如果能让各家 Service Mesh 提供商都遵循这个标准，则有机会在具体的 Service Mesh 产品之上，抽象出一组通用的、可移植的 Service Mesh API，屏蔽掉上层应用/工具/生态系统对具体 Service Mesh 产品的实现细节，这使得 Kubernetes 用户可以以供应商无关的方式使用这些 API。通过这种方式，用户可以定义使用 Service Mesh 技术的应用程序，而无需紧密绑定到任何特定实现。

SMI 是一个开放项目，由微软、Linkerd、HashiCorp、Solo、Kinvolk 和 Weaveworks 联合启动，并得到了Aspen Mesh、Canonical、Docker、Pivotal、Rancher、Red Hat 和 VMware 的支持。

### Red Hat

Red Hat 在云原生方面深耕多年，作为 Istio 项目的早期采用者和贡献者，希望将 Istio 正式成为 OpenShift 平台的一部分。

Red Hat 于 2018 年推出了 OpenShift Service Mesh 技术预览版，正是基于 Istio 实现，可以为 OCP 客户提供在其 OpenShift 集群上部署和使用 Istio 的能力。此外，Red Hat 还创建了一个名为 Maistra 的社区项目，为 Istio 集成了诸多开源项目，以降低 Istio 的使用成本。Red Hat 凭借在私有化交付市场中的高占有率，实施落地了众多 Service Mesh 案例。

### Aspen Mesh

Aspen Mesh 来自大名鼎鼎的 F5 Networks 公司，基于 Istio 构建，定位企业级服务网格，口号是”Service Mesh Made Easy”。

Aspen Mesh 项目据说启动非常之早，2017 年 5 月 Istio 发布 0.1 版本不久之后就开始组建团队进行开发，但是一直以来都非常低调，外界了解到的信息不多。在 2018 年 9 月，Aspen Mesh 1.0 发布，基于 Istio 1.0 实现。用户可以在 Aspen Mesh 的官方网站上申请免费试用。

### 国内

除以上几个国际化云厂家之外，国内云厂商也在积极布局 Service Mesh。以蚂蚁金服、华为、阿里、腾讯、网易、新浪微博等公司为代表的国内互联网公司，以多种方式给出了符合自身特点的 Service Mesh 产品，思路和打法各有不同。
	
* 蚂蚁金服：较早推出了 SOFAMesh，该服务继承了 Istio 强大的功能和丰富特性，并基于 MOSN 提供数据面服务，在兼容 xDS 标准的基础上，对 Istio 进行了部分优化和增强。
* 腾讯云 TSF：基于 Istio 实现了商业化 Service Mesh 服务，并通过一套框架将 Spring Cloud 和 Service Mesh 服务进行了融合，以便于用户平滑过渡到 Service Mesh 方案。
* 华为云 CSE：基于 Golang 自研 Service Mesh 服务框架，并以私有项目 Mesher 作为数据面代理，为用户提供了开箱即用的服务体验。
* 网易轻舟微服务：基于 Istio 实现了商业化 Service Mesh 服务，通过自研 API-plane 组件对 Istio API 进行了封装，从而实现轻舟商业化版本与 Istio 开源版本的融合，目前已在网易严选、网易传媒等场景落地使用。
* 新浪微博 WeiboMesh：微博内部跨语言服务化解决方案，目前已经在微博多条业务线上得到广泛使用，包括热搜、话题等核心项目。

## 小结

Service Mesh 作为一种新兴技术，仍处于生命周期的初级阶段。目前已经有很多优秀的项目和产品加入了 Service Mesh 生态，同时有更多的企业和个人都正在实践自己的服务网格。这表明 Service Mesh 是有价值的，并且应该探索不同的实现方式，才可以不断完善壮大。同时，伴随着更多实际的案例落地，Service Mesh 相关技术终会趋于成熟，走入大众的视野。

## 参考

* [The Top 3 Service Mesh Developments in 2020](https://thenewstack.io/the-top-3-service-mesh-developments-in-2020/)
* [The enterprise service mesh ecosystem comes into focus](https://www.computerworld.com/article/3428077/the-enterprise-service-mesh-ecosystem-comes-into-focus.html)
* [Service Mesh - the Microservices in Post Kubernetes Era](https://jimmysong.io/en/blog/service-mesh-the-microservices-in-post-kubernetes-era/)
* [Kubernetes Service Mesh: A Comparison of Istio, Linkerd and Consul](https://platform9.com/blog/kubernetes-service-mesh-a-comparison-of-istio-linkerd-and-consul/)
* [HashiCorp Consul 1.2: Service Mesh](https://www.hashicorp.com/blog/consul-1-2-service-mesh/)
* [Istio 1.5 introduces simpler architecture, improving operational experience](https://developer.ibm.com/blogs/istio-15-release/)
* [Kong’s Kuma Service Mesh Climbs the Kubernetes Wall](https://www.sdxcentral.com/articles/news/kongs-kuma-service-mesh-climbs-the-kubernetes-wall/2019/09/)
* [Service Mesh年度总结：群雄逐鹿烽烟起](https://skyao.io/publication/201801-service-mesh-2017-summary/)
* [Service Mesh2018年度总结Service Mesh](https://skyao.io/publication/201902-service-mesh-2018-summary/)
* [Service Mesh 发展趋势：云原生中流砥柱](https://mp.weixin.qq.com/s/N_z14Ej_TUCEvo3Onzausw)
* [史上最全的高性能代理服务器 Envoy 中文实战教程](https://blog.csdn.net/Ki8Qzvka6Gz4n450m/article/details/103519150)