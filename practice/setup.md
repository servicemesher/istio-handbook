---
authors: ["sunny0826"]
reviewers: [""]
---

# 安装与部署

Istio 自 1.5 版本开始发生了巨大的变化，不但架构回归单体，连安装方式都弃用了之前版本推荐的`Helm`安装，改用`istioctl`来进行安装。用户可以使用`istioctl`在本地或公有云上搭建 Istio 环境，也可直接使用公有云平台上集成的 Istio 的托管服务。本章为实践篇的开篇，将从 Istio 的安装开始，逐步介绍升级和不同环境下的部署，同时还将介绍`Bookinfo`示例，带领读者快速体验 istio 的安装方式以及基本应用，利用 [Katacoda 平台](https://katacoda.com)，手把手教读者快速上手 Istio，体验 Istio 的各种功能。
