---
authors: ["silvasong"]
reviewers: [""]
---

# Kubernetes

这一节将介绍如何使用 istioctl 在 Kubernetes 集群中部署 Istio 。如需了解通过 [Helm](https://istio.io/docs/setup/install/helm/) 和 [Operator](https://istio.io/docs/setup/install/standalone-operator/) 部署，请参考官方文档介绍 。

### Kubernetes 版本要求

Istio 1.5 建议部署在 Kubernetes 1.14 及以上。

### Kubernetes Pod 和 Service 要求

作为 Istio 服务网格中的一部分,集群中的 Pod 和 Service 必须满足以下要求：

- 命名的服务端口： Service 的端口必须命名，端口名键值对必须按以下格式：name: &lt;protocol&gt;[-&lt;suffix&gt;]。
- Service 关联: 无论 Pod 是否对外暴露端口，每个 Pod 必须至少属于一个 Kubernetes Service 。
- 带有 app 和 version 标签（label）的 Deployment : 建议给 Deployment 加上 app 和 version 标签。 通过标签可以给 Istio 收集的指标和遥测信息增加上下文信息。
- 应用 UID: 确保你的 Pod 不会以用户 ID（UID）为 1337 的用户运行应用。
- NET_ADMIN 功能: 如果你的集群执行 Pod 安全策略，必须给 Pod 配置 NET_ADMIN 功能。使用 Istio CNI 插件可以不配置。

### 下载 Istio

可以从 GitHub 的 [release](https://github.com/istio/istio/releases) 页面下载，或者直接通过下面的命令下载：

`curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.5.0 sh -`

下载后进入 istio-1.5 ，可以看到目录结构：
```bash
.
├── bin
├── install
├── LICENSE
├── manifest.yaml
├── README.md
├── samples
└── tools
```
- `install/kubernetes` : 针对 Kubernetes 平台的安装文件
- `samples` : 示例应用
- `bin` :  istioctl 二进制文件，可以用来手动注入 sidecar proxy

### 通过配置文件安装

Istio 提供多种配置文件，通过 `istioctl profile list `命令查看。

```bash
Istio configuration profiles:
    separate
    default
    demo
    empty
    minimal
    remote
```

不同类型配置文件差异如下：

|                       |default| demo | minimal| remote |
| ----------------------|-------| -----|------- |------  |
| 核心组件              |       |      |        |        |
| istio-egressgateway   |       | X    |        |        |
| istio-ingressgateway  | X     | X    |       |        |
| istio-pilot           | X     | X    | X     |        |
| 附加组件               |       |      |      |       |
| Grafana               |       | X     |       |        |
| istio-tracing         |       | X     |       |        |
| kiali                 |       | X     |       |        |
| prometheus            | X     | X     |       |X       |

其中标记 X 表示安装该组件。

**使用默认配置文件安装**

如下命令，将使用 default 配置文件安装。

`istioctl manifest apply`

如果要在 default 配置文件基础上启用其他功能，可以通过 —set 选项，例如下面的命令将启用 Grafana Dashboard 。

`istioctl manifest apply --set addonComponents.grafana.enabled=true`

**使用外部 Chart **

默认情况下 istioctl 使用 install/kubernetes/operator/charts 目录下内置的 Chart 生成安装清单，通过配置 installPackagePath 参数，允许使用外部 Chart 。

`istioctl manifest apply --set installPackagePath=/istio-1.5.0/install/kubernetes/operator/charts`

生产环境建议使用内置 Chart 而不是外部提供，以确保 istioctl 二进制文件与 Chart 的兼容性。

**安装前生成清单**

除了使用 manifest apply 外，还可以使用 manifest generate 命令生成清单，例如，通过如下命令为 default 配置文件生成清单：

`istioctl manifest generate > $HOME/generated-manifest.yaml`

根据需要调整清单，然后通过 kubectl 命令应用清单。

`kubectl apply -f $HOME/generated-manifest.yaml`

### 验证安装是否成功

istioctl 提供 verify-install 命令检查安装是否成功，它将集群上的安装与您指定的清单进行比较。

`istioctl verify-install -f $HOME/generated-manifest.yaml`

### 定制化配置

除了使用 Istio 内置的配置文件，istioctl manifest 提供用于自定义配置的 IstioOperator API。
使用 istioctl 命令的 --set 选项可以设置此 API 中的配置参数。 例如，通过如下命令可以在default配置基础上启用控制平面安全特性：

`istioctl manifest apply --set values.global.controlPlaneSecurityEnabled=true`

或者，可以在 YAML 文件中指定 IstioOperator 配置，通过 -f 选项将其传给 istioctl :

`istioctl manifest apply -f samples/operator/pilot-k8s.yaml`

IstioOperator API 定义的核心组件如下：

- base
- pilot
- proxy
- sidecarInjector
- telemetry
- policy
- citadel
- nodeagent
- galley
- ingressGateways
- egressGateways
- cni

除核心组件外，还提供了第三方附加功能和组件，通过配置 IstioOperator API 的 addonComponents spec 启用。

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  addonComponents:
    grafana:
      enabled: true
```

识别组件的名称后，可以通过 --set 标志或者 --filename 标志，设置组件配置，启用或禁用组件功能。

例如，通过下面的命令可以禁用遥测功能。

`istioctl manifest apply --set components.telemetry.enabled=false`

或者，您可以使用 yaml 文件禁用遥测功能。
1、创建 telemetry_off.yaml 文件。
```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  components:
    telemetry:
      enabled: false
```
2、使用 telemetry_off.yaml 禁用遥测功能。

`istioctl manifest apply -f telemetry_off.yaml`

### 自定义Kubernetes设置

IstioOperator API 允许以一致的方式自定义每个组件的 Kubernetes 设置。使用如下列表来标识要自定义的设置：

- Resources
- Readiness probes
- Replica count
- HorizontalPodAutoscaler
- PodDisruptionBudget
- Pod annotations
- Service annotations
- ImagePullPolicy
- Priority class name
- Node selector
- Affinity and anti-affinity
- Service
- Toleration
- Strategy
- Env

例如，通过下面的示例可以调整 Pilot 的 resource 和 pod 水平自动伸缩的设置。

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  components:
    pilot:
      k8s:
        resources:
          requests:
            cpu: 1000m # override from default 500m
            memory: 4096Mi # ... default 2048Mi
        hpaSpec:
          maxReplicas: 10 # ... default 5
          minReplicas: 2  # ... default 1
        nodeSelector:
          master: "true"
        tolerations:
        - key: dedicated
          operator: Exists
          effect: NoSchedule
        - key: CriticalAddonsOnly
          operator: Exists
```
使用 manifest apply 将修改后的设置应用于集群：

`istioctl manifest apply -f samples/operator/pilot-k8s.yaml`

如需了解更多设置，可以参考 Kubernetes API 定义。