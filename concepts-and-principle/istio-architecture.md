---
owners: ["rootsongjc"]
reviewers: [""]
description: "本文从高层次上介绍了 Istio 的架构。"
publishDate: 2019-03-10
updateDate: 2019-03-10
tags: ["service mesh"]
category: "evolution"
---

# Istio 架构解析

下面是以漫画的形式说明 Istio 是什么。

![来自 Twitter @daniseyu21](https://ws3.sinaimg.cn/large/006tNbRwly1fujrgeesk7j316c0tz10y.jpg)

该图中描绘了以下内容：

- Istio 可以在虚拟机和容器中运行
- Istio 的组成
  - Pilot：服务发现、流量管理
  - Mixer：访问控制、遥测
  - Citadel：终端用户认证、流量加密
- Service mesh 关注的方面
  - 可观察性
  - 安全性
  - 可运维性
- Istio 是可定制可扩展的，组建是可拔插的
- Istio 作为控制平面，在每个服务中注入一个 Envoy 代理以 Sidecar 形式运行来拦截所有进出服务的流量，同时对流量加以控制
- 应用程序应该关注于业务逻辑（这才能生钱），非功能性需求交给 Service Mesh

## 参考

- [Isito 是什么? - istio.io](https://istio.io/zh/docs/concepts/what-is-istio/)
- [Bookinfo 示例](../action/bookinfo-sample.md)

