---
authors: ["sunny0826"]
reviewers: ["rootsongjc"]
---

# 升级

自 Istio 1.5 版本开始，就弃用了之前使用 `Helm` 安装的方式，所以本节只讲解使用 `istioctl` 进行升级。并且虽然 Istio 提供了降级方式，但是经过测试降级的体验并不好，并且出现了由于不支持的 CRD `apiVersion` 导致无法降级的情况，所以请谨慎升级。

>Istio 不支持 跨版本升级。仅支持从 1.5 版本升级到 1.6 版本。如果您使用的是旧版本，请先升级到 1.5 版本。

## 金丝雀升级

Istio 1.6 推出了渐进式的 Istio 升级方式：金丝雀升级，可以让新老版本的 `istiod` 同时存在，并允许将所有流量在由新版 `istiod` 控制之前，先将一小部分工作负载交由新版本 `istiod` 控制，并进行监控，渐进式的完成升级。该方式比原地升级安全的多，也是推荐的升级方式。

### 控制平面升级

首先需要 [下载新版本 Istio](https://github.com/istio/istio/releases) 并切换目录为新版本目录。

安装 `canary` 版本，将 `revision` 字段设置为 `canary`：

```bash
$ istioctl install --set revision=canary
```

这里会部署新的`istiod-canary`，并不会对原有的控制平面造成影响，部署成功后会看到两个并行的`istiod`：

```bash
$ kubectl get pods -n istio-system
NAME                                        READY   STATUS    RESTARTS   AGE
pod/istiod-85745c747b-knlwb                 1/1     Running   0          33m
pod/istiod-canary-865f754fdd-gx7dh          1/1     Running   0          3m25s
```

这里还可以看到新版的`sidecar injector`配置：

```bash
$ kubectl get mutatingwebhookconfigurations
NAME                            CREATED AT
istio-sidecar-injector          2020-07-07T08:39:37Z
istio-sidecar-injector-canary   2020-07-07T09:06:24Z
```

### 数据平面升级

只安装 `canary` 版本的控制平面并不会对现有的代理造成影响，要升级数据平面，将他们指向新的控制平面，需要在 namespace 中插入 `istio.io/rev` 标签。

例如，想要升级 `default` namespace 的数据平面，需要添加 `istio.io/rev` 标签以指向 `canary` 版本的控制平面，并删除 `istio-injection` 标签：

```bash
$ kubectl label namespace default istio-injection- istio.io/rev=canary
```

注意：`istio-injection`标签必须删除，因为该标签的优先级高于`istio.io/rev` 标签，保留该标签将导致无法升级数据平面。

在 namespace 更新成功后，需要重启 Pod 来重新注入 Sidecar：

```bash
$ kubectl rollout restart deployment -n default
```

当重启成功后，该 namespace 的 pod 将被配置指向新的`istiod-canary`控制平面，使用如下命令查看启用新代理的 Pod：

```bash
$ kubectl get pods -n default -l istio.io/rev=canary
```

同时可以使用如下命令验证新 Pod 的控制平面为`istiod-canary`：

```bash
$ istioctl proxy-config endpoints ${pod_name}.default --cluster xds-grpc -ojson | grep hostname
    "hostname": "istiod-canary.istio-system.svc"
```

## 原地升级

> 目前原地升级有很大的概率通不过升级检测，导致无法升级，不推荐这种升级方式。

升级过程中可能发生流量中断。为了缩短流量中断时间，请确保每个组件（Citadel 除外）至少运行有两个副本。同时，确保 `PodDisruptionBudgets` 配置最小可用性为 1。

`istioctl` 通过查看支持的版本列表，验证是否支持从当前 Istio 版本升级：

```bash
$ istioctl manifest versions
```

使用`istioctl upgrade`命令进行升级：

```bash
$ istioctl upgrade -f '<your-custom-configuration-file>'
````

`<your-custom-configuration-file>` 是 [IstioControlPlane API Custom Resource Definition](https://istio.io/latest/zh/docs/setup/install/istioctl/#configure-the-feature-or-component-settings) 文件，该文件用于自定义安装当前运行版本的 Istio。

遗憾的是 `istioctl upgrade` 命令不支持 `--set` 选项。因此，如果 Istio 是使用 `--set` 选项安装的，请创建一个与配置项等效的配置文件，并将其传递给 `istioctl upgrade` 命令。使用 `-f` 选项来加载配置文件。

同样的，在完成升级后，还需要重启 Pod 来重新注入 Sidecar：

```bash
$ kubectl rollout restart deployment
```

## 总结

总体来说，金丝雀升级的出现很好的解决了控制平面渐进式升级的需求，但是由于 `istioctl upgrade` 命令支持的场景和版本太少以及 Istio 整体架构的更改，目前的升级体验很不好，急需提升。

## 参考

- [Upgrade Istio - istio.io](https://istio.io/latest/docs/setup/upgrade/)
