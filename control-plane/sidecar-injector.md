---
owners: ["zhongfox"]
reviewers: ["zhongfox"]
description: "对 istio sidecar injector 进行实现剖析。"
publishDate: 2019-07-22
updateDate: 2019-07-22
tags: ["sidecar injector","control plane"]
category: "original"
---

# Sidecar Injector

用户空间的 Pod 要想加入 mesh, 首先需要注入 sidecar 容器, istio 提供了 2 种方式实现注入:

- 自动注入: 利用 [Kubernetes Dynamic Admission Webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/) 对 新建的 pod 进行注入: initContainer + sidecar
- 手动注入: 使用命令`istioctl kube-inject`

「注入」本质上就是修改 Pod 的资源定义, 添加相应的 sidecar 容器定义, 内容包括 2 个新容器:

- 名为`istio-init`的 initContainer: 通过配置 iptables 来劫持 Pod 中的流量
- 名为`istio-proxy`的 sidecar 容器: 两个进程 pilot-agent 和 envoy, pilot-agent 进行初始化并启动 envoy

<img src="https://ws4.sinaimg.cn/large/006tKfTcgy1g187flw0dmj30wq0grn0b.jpg" referrerpolicy="no-referrer"/>


## Dynamic Admission Control

kubernetes 的准入控制(Admission Control)有 2 种:

- Built in Admission Control: 这些 Admission 模块可以选择性地编译进 api server, 因此需要修改和重启 kube-apiserver
- Dynamic Admission Control: 可以部署在 kube-apiserver 之外, 同时无需修改或重启 kube-apiserver。

其中, Dynamic Admission Control 包含 2 种形式:

- Admission Webhooks: 该 controller 提供 http server, 被动接受 kube-apiserver 分发的准入请求。
- Initializers: 该 controller 主动 list and watch 关注的资源对象, 对 watch 到的未初始化对象进行相应的改造。

 其中, Admission Webhooks 又包含 2 种准入控制:

- ValidatingAdmissionWebhook
- MutatingAdmissionWebhook

istio 使用了 MutatingAdmissionWebhook 来实现对用户 Pod 的注入,  首先需要保证以下条件满足:

- 确保 kube-apiserver 启动参数 开启了 MutatingAdmissionWebhook
- 给 namespace 增加 label: `kubectl label namespace default istio-injection=enabled`
- 同时还要保证 kube-apiserver 的 aggregator layer 开启: `--enable-aggregator-routing=true` 且证书和 api server 连通性正确设置。

另外还需要一个配置对象, 来告诉 kube-apiserver istio 关心的资源对象类型, 以及 webhook 的服务地址。 如果使用 helm 安装 istio, 配置对象已经添加好了, 查阅 MutatingWebhookConfiguration:

```yaml
% kubectl get mutatingWebhookConfiguration -oyaml
- apiVersion: admissionregistration.k8s.io/v1beta1
  kind: MutatingWebhookConfiguration
  metadata:
    name: istio-sidecar-injector
  webhooks:
    - clientConfig:
      service:
        name: istio-sidecar-injector
        namespace: istio-system
        path: /inject
    name: sidecar-injector.istio.io
    namespaceSelector:
      matchLabels:
        istio-injection: enabled
    rules:
    - apiGroups:
      - ""
      apiVersions:
      - v1
      operations:
      - CREATE
      resources:
      - pods
```

该配置告诉 kube-apiserver: 命名空间 istio-system 中的服务 `istio-sidecar-injector`(默认 443 端口), 通过路由`/inject`, 处理`v1/pods`的 CREATE, 同时 pod 需要满足命名空间`istio-injection: enabled`, 当有符合条件的 pod 被创建时, kube-apiserver 就会对该服务发起调用, 服务返回的内容正是添加了 sidecar 注入的 pod 定义。


## Sidecar 注入内容分析

查看 Pod `istio-sidecar-injector`的 yaml 定义:

```yaml
% kubectl -n istio-system get pod istio-sidecar-injector-5f7894f54f-w7f9v -oyaml
......
    volumeMounts:
    - mountPath: /etc/istio/inject
      name: inject-config
      readOnly: true

  volumes:
  - configMap:
      items:
      - key: config
        path: config
      name: istio-sidecar-injector
    name: inject-config
```

可以看到该 Pod 利用[projected volume](https://kubernetes.io/docs/concepts/storage/volumes/#projected)将`istio-sidecar-injector`这个 config map 的 config 挂到了自己容器路径`/etc/istio/inject/config`, 该 config map 内容正是注入用户空间 pod 所需的模板。

如果使用 helm 安装 istio, 该 configMap 模板源码位于: <https://github.com/istio/istio/blob/master/install/kubernetes/helm/istio/templates/sidecar-injector-configmap.yaml>。

该 config map 是在安装 istio 时添加的, kubernetes 会自动维护 projected volume 的更新, 因此 容器 `sidecar-injector`只需要从本地文件直接读取所需配置。

高级用户可以按需修改这个模板内容。

```plain
kubectl -n istio-system get configmap istio-sidecar-injector -o=jsonpath='{.data.config}'
```

查看该 configMap, `data.config`包含以下内容(简化):

```yaml
policy: enabled # 是否开启自动注入
template: |-    # 使用go template 定义的pod patch
  initContainers:
  [[ if ne (annotation .ObjectMeta `sidecar.istio.io/interceptionMode` .ProxyConfig.InterceptionMode) "NONE" ]]
  - name: istio-init
    image: "docker.io/istio/proxy_init:1.1.0"
    ......
    securityContext:
      capabilities:
        add:
        - NET_ADMIN
    ......
  containers:
  - name: istio-proxy
    args:
    - proxy
    - sidecar
    ......
    image: [[ annotation .ObjectMeta `sidecar.istio.io/proxyImage`  "docker.io/istio/proxyv2:1.1.0"  ]]
    ......
    readinessProbe:
      httpGet:
        path: /healthz/ready
        port: [[ annotation .ObjectMeta `status.sidecar.istio.io/port`  0  ]]
    ......
    securityContext:
      capabilities:
        add:
        - NET_ADMIN
      runAsGroup: 1337
  ......
    volumeMounts:
    ......
    - mountPath: /etc/istio/proxy
      name: istio-envoy
    - mountPath: /etc/certs/
      name: istio-certs
      readOnly: true
      ......
  volumes:
  ......
  - emptyDir:
      medium: Memory
    name: istio-envoy
  - name: istio-certs
    secret:
      optional: true
      [[ if eq .Spec.ServiceAccountName "" -]]
      secretName: istio.default
      [[ else -]]
      secretName: [[ printf "istio.%s" .Spec.ServiceAccountName ]]
      ......

```

对 istio-init 生成的部分参数分析:

- `-u 1337` 排除用户 ID 为 1337，即 Envoy 自身的流量
- 解析用户容器`.Spec.Containers`, 获得容器的端口列表, 传入`-b`参数(入站端口控制)
- 指定要从重定向到 Envoy 中排除（可选）的入站端口列表, 默认写入`-d 15020`, 此端口是 sidecar 的 status server
- 赋予该容器`NET_ADMIN` 能力, 允许容器 istio-init 进行网络管理操作

对 istio-proxy 生成的部分参数分析:

- 启动参数`proxy sidecar xxx` 用以定义该节点的代理类型(NodeType)
- 默认的 status server 端口`--statusPort=15020`
- 解析用户容器`.Spec.Containers`, 获取用户容器的 application Ports, 然后设置到 sidecar 的启动参数`--applicationPorts`中, 该参数会最终传递给 envoy, 用以确定哪些端口流量属于该业务容器。
- 设置`/healthz/ready` 作为该代理的 readinessProbe
- 同样赋予该容器`NET_ADMIN`能力

另外`istio-sidecar-injector`还给容器`istio-proxy`挂了 2 个 volumes:

- 名为`istio-envoy`的 emptydir volume, 挂载到容器目录`/etc/istio/proxy`, 作为 envoy 的配置文件目录

- 名为`istio-certs`的 secret volume, 默认 secret 名为`istio.default`,  挂载到容器目录`/etc/certs/`, 存放相关的证书, 包括服务端证书, 和可能的 mtls 客户端证书

  ```plain
  % kubectl exec productpage-v1-6597cb5df9-xlndw -c istio-proxy -- ls /etc/certs/
  cert-chain.pem
  key.pem
  root-cert.pem
  ```
