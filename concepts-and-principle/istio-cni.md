---
owners: ["imroc"]
reviewers: ["rootsongjc"]
description: "本文介绍了 Istio CNI Plugin 的概念与原理"
publishDate: 2019-04-10
updateDate: 2019-04-10
tags: ["service mesh"]
category: "original"
---

# Istio CNI Plugin
## 设计目标
当前实现将用户 pod 流量转发到 proxy 的默认方式是使用 privileged 权限的 istio-init 这个 init container 来做的（运行脚本写入 iptables），Istio CNI 插件的主要设计目标是消除这个 privileged 权限的 init container，换成利用 k8s CNI 机制来实现相同功能的替代方案


## 原理
- Istio CNI Plugin 不是 istio 提出类似 k8s CNI 的插件扩展机制，而是 k8s CNI 的一个具体实现
- k8s CNI 插件是一条链，在创建和销毁pod的时候会调用链上所有插件来安装和卸载容器的网络，istio CNI Plugin 即为 CNI 插件的一个实现，相当于在创建销毁pod这些hook点来针对istio的pod做网络配置：写入iptables，让该 pod 所在的 network namespace 的网络流量转发到 proxy 进程
- 当然也就要求集群启用 CNI，kubelet 启动参数: `--network-plugin=cni` （该参数只有两个可选项：`kubenet`, `cni`）

## 实现方式
- 运行一个名为 istio-cni-node 的 daemonset 运行在每个节点，用于安装 istio CNI 插件
- 该 CNI 插件负责写入 iptables 规则，让用户 pod 所在 netns 的流量都转发到这个 pod 中 proxy 的进程
- 当启用 istio cni 后，sidecar 的自动注入或`istioctl kube-inject`将不再注入 initContainers (istio-init)

## istio-cni-node 工作流程
- 复制 Istio CNI 插件二进制程序到CNI的bin目录（即kubelet启动参数`--cni-bin-dir`指定的路径，默认是`/opt/cni/bin`）
- 使用istio-cni-node自己的ServiceAccount信息为CNI插件生成kubeconfig，让插件能与apiserver通信(ServiceAccount信息会被自动挂载到`/var/run/secrets/kubernetes.io/serviceaccount`)
- 生成CNI插件的配置并将其插入CNI配置插件链末尾（CNI的配置文件路径是kubelet启动参数`--cni-conf-dir`所指定的目录，默认是`/etc/cni/net.d`）
- watch CNI 配置(`cni-conf-dir`)，如果检测到被修改就重新改回来
- watch istio-cni-node 自身的配置(configmap)，检测到有修改就重新执行CNI配置生成与下发流程（当前写这篇文章的时候是istio 1.1.1，还没实现此功能）

## 设计提案
- Istio CNI Plugin 提案创建时间：2018-09-28
- Istio CNI Plugin 提案文档存放在：Istio 的 Google Team Drive
  - Istio TeamDrive 地址：https://drive.google.com/corp/drive/u/0/folders/0AIS5p3eW9BCtUk9PVA
  - Istio CNI Plugin 提案文档路径：`Working Groups/Networking/Istio CNI Plugin`
  - 查看文件需要申请权限，申请方法：加入istio-team-drive-access这个google网上论坛group
  - istio-team-drive-access group 地址: https://groups.google.com/forum/#!forum/istio-team-drive-access

## 参考资料
- Install Istio with the Istio CNI plugin: https://istio.io/docs/setup/kubernetes/additional-setup/cni/
- istio-cni 项目地址：https://github.com/istio/cni