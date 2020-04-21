---
authors: ["sunny0826"]
reviewers: ["rootsongjc"]
---

# 安装 Istio

本节讲解如何快速在本地安装 Istio 环境。

## 环境准备

本章讲解本地 Kubernetes 环境下的 Istio 安装，这里仅给出本地环境需要的软硬件要求：

**硬件要求**

至少 2 核 CPU 和 4G 的可用内存。

**软件要求**

- [minikube v1.9.2](https://github.com/kubernetes/minikube/releases/tag/v1.9.2)
- `istioctl` v1.5.1
- `kubectl` v1.18
- docker 19.03.8

### 安装 Kubernetes 集群

在本地计算机上部署 Kubernetes，可以帮助开发人员快速高效的测试应用程序，同时也可以在测试时避免对线上正在运行的集群造成影响。目前有许多软件提供了在本地搭建 Kubernetes 集群的功能，本书推荐使用 Minikube 来安装 Kubernetes 集群。

Minikube 是一种可以在本地轻松运行 Kubernetes 的工具，适用于所有主流操作系统。Minikube 可在笔记本电脑上的虚拟机（VM）中运行单节点 Kubernetes 集群，供那些希望尝试 Kubernetes 或进行日常开发的用户使用。下面以 Linux 平台为例，简要说明 Minikube 的安装步骤。

1. 下载 Minikube，你可以在 Minikube 在 GitHub 上的 [releases 页面](https://github.com/kubernetes/minikube/releases)找到 Linux (AMD64) 的包。这里直接下载 Minikube 的二进制文件，并添加执行权限：
    ```bash
    $ curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube
    ```

2. 将 Minikube 可执行文件添加至 path：
    ```bash
    $ sudo mkdir -p /usr/local/bin/
    $ sudo install minikube /usr/local/bin/
    ```

3. 启动 Minikube 并创建一个集群：
    ```bash
    $ minikube start
    ```

创建成功后，用户需要使用 kubectl 工具来管理和操作集群中的各种资源，`minikube start` 命令会创建一个名为 `minikube` 的 [kubectl 上下文](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-set-context-em-)。Minikube 会自动将此上下文设置为默认值，但如果您以后需要切换回它，请运行：

```bash
$ kubectl config use-context minikube
```
或者像这样，每个命令都附带其执行的上下文：
```bash
$ kubectl get pods --context=minikube
```

详细安装步骤以及操作说明请参考 Minikube 官方文档：https://minikube.sigs.k8s.io/docs/

## 安装 Istio

从 Istio v1.15 版本开始，[使用 helm 安装](https://istio.io/zh/docs/setup/install/helm/)的方式已经废弃，需改用 [istioctl 安装](https://istio.io/zh/docs/setup/install/istioctl/)。

在 [Istio release](https://github.com/istio/istio/releases) 页面下载与操作系统匹配的安装包，这里以 Istio 1.5 版本为例：
```bash
$ curl -L https://raw.githubusercontent.com/istio/istio/release-1.5/release/downloadIstioCandidate.sh | sh -
```

安装目录内容：
|目录|包含内容|
|---|---|
|`bin`| 包含 istioctl 的客户端文件 |
|`install`| 包含 Consul、GCP 和 Kubernetes 平台的 Istio 安装脚本和文件 |
|`samples`| 包含示例应用程序 |
|`tools`| 包含用于性能测试和在本地机器上进行测试的脚本 |

将`istioctl`客户端路径加入 $PATH 中：

```bash
export PATH=$PATH:$(pwd)/istio-1.5.1/bin
```

之后就可以使用`istioctl`命令行工具了，该命令行工具具有用户输入校验功能，可以防止错误的安装和自定义选项。

由于本章主要介绍快速在本地安装 Istio 环境，不涉及到性能及可用性相关话题，故使用`demo`配置安装 Istio：

```bash
$ istioctl manifest apply --set profile=demo
```

安装命令运行成功后，检查 Kubernetes 服务是否部署正常，检查除 `jaeger-agent` 服务外的其他服务，是否均有正确的 `CLUSTER-IP`：

```bash
$ kubectl get svc -n istio-system
NAME                        TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                             AGE
grafana                     ClusterIP      10.108.112.31    <none>        3000/TCP                                                                             24s
istio-egressgateway         ClusterIP      10.106.157.7     <none>        80/TCP,443/TCP,15443/TCP                                                                             26s
istio-ingressgateway        LoadBalancer   10.110.57.34     <pending>     15020:31817/TCP,80:30733/TCP,443:31910/TCP,15029:32168/TCP,15030:31733/TCP,15031:31981/TCP,15032:30531/TCP,31400:31169/TCP,15443:31131/TCP   26s
istio-pilot                 ClusterIP      10.110.196.147   <none>        15010/TCP,15011/TCP,15012/TCP,8080/TCP,15014/TCP,443/TCP                                                                             46s
istiod                      ClusterIP      10.104.27.234    <none>        15012/TCP,443/TCP                                                                             46s
jaeger-agent                ClusterIP      None             <none>        5775/UDP,6831/UDP,6832/UDP                                                                             24s
jaeger-collector            ClusterIP      10.103.156.147   <none>        14267/TCP,14268/TCP,14250/TCP                                                                             24s
jaeger-collector-headless   ClusterIP      None             <none>        14250/TCP                                                                             24s
jaeger-query                ClusterIP      10.110.109.206   <none>        16686/TCP                                                                             24s
kiali                       ClusterIP      10.96.182.125    <none>        20001/TCP                                                                             24s
prometheus                  ClusterIP      10.104.167.86    <none>        9090/TCP                                                                             24s
tracing                     ClusterIP      10.102.230.151   <none>        80/TCP                                                                             24s
zipkin                      ClusterIP      10.111.66.10     <none>        9411/TCP                                                                             24s
```

检查相关 pod 是否部署成功：

```bash
$ kubectl get pods -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
grafana-5cc7f86765-jxdcn                1/1     Running   0          4m24s
istio-egressgateway-598d7ffc49-bdmzw    1/1     Running   0          4m24s
istio-ingressgateway-7bd5586b79-gnzqv   1/1     Running   0          4m25s
istio-tracing-8584b4d7f9-tq6nq          1/1     Running   0          4m24s
istiod-646b6fcc6-c27c7                  1/1     Running   0          4m45s
kiali-696bb665-jmts2                    1/1     Running   0          4m24s
prometheus-6c88c4cb8-xchzd              2/2     Running   0          4m24s
```

如果所有组件对应的 Pod 的`STATUS`都变为`Running`，则说明 Istio 已经安装完成了。

## 小结

本节主要介绍如何快速安装 Istio。自 1.5 版本开始，Istio 不但架构回归单体，连安装方式都弃用了之前版本推荐的`Helm`安装，改用`istioctl`来进行安装。整体安装方式更加的简便快捷，在安装时节约了大量的时间，对于新手来说也更加的友好，不再需要像之前那样在安装前就要了解大量的`Helm`参数。提供了多种安装配置方便选择，同时也提供了生成安装清单功能帮助进一步的定制，详细内容会在 Kubernetes 部署章节详解。

## 参考

- [使用 Minikube 安装 Kubernetes - kubernetes.io](https://kubernetes.io/zh/docs/setup/learning-environment/minikube/)
- [开始 - istio.io](https://istio.io/zh/docs/setup/getting-started/)
