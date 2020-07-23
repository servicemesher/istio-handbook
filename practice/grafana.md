---
authors: ["lianghao"]
reviewers: [""]
---

# Grafana

## Grafana 简介

Grafana 是一款开源的指标数据可视化工具，有着功能齐全的度量仪表盘、图表等时序数据展示面板，支持 Zabbix、InfluentDB、Prometheus、Elasticsearch、MySQL 等数据源的指标展示，详情查看：[Grafana 支持的数据源类型](https://grafana.com/docs/grafana/latest/features/datasources/#supported-data-sources/)。

总而言之，Grafana 是一款提供了将时间序列数据库（TSDB）数据转换为精美的图形和可视化面板的工具。在 Istio 安装完成后，默认已经将 Grafana 服务以 Pod 的方式运行起来，同时也配置了 Istio 中的 Prometheus 作为数据源，定时地从 Prometheus 中采集 Istio 各组件的指标数据，进行可视化展示。

## Grafana 的使用和配置

### 登录设置

我们打开 Grafana 主页，发现没有跳转到登录页面，可以直接访问：

![Grafana 主页](../images/grafana-home.png)

原因是 Istio 在安装 Grafana 时对 Grafana 做了免登录配置，我们查看 Grafana 的 deployment 中 **GF_AUTH_ANONYMOUS_ENABLED** 和 **GF_AUTH_ANONYMOUS_ORG_ROLE** 环境变量：

``` bash
$ kubectl get deployment grafana -n istio-system -o yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
...
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  ...
  template:
    metadata:
      annotations:
    ...
    spec:
	  ...
      containers:
      - env:
        - name: GRAFANA_PORT
          value: "3000"
        - name: GF_AUTH_BASIC_ENABLED
          value: "false"
        - name: GF_AUTH_ANONYMOUS_ENABLED
          value: "true"
        - name: GF_AUTH_ANONYMOUS_ORG_ROLE
          value: Admin
        - name: GF_PATHS_DATA
          value: /data/grafana
      ...
```

当 **GF_AUTH_ANONYMOUS_ENABLED** 环境变量设置为 "true" 时，表示开启匿名免登录访问。而 **GF_AUTH_ANONYMOUS_ORG_ROLE** 环境变量设置为 Admin 则表示匿名免登录时具有 Admin 权限。

Grafana里面的用户有三种权限：Admin、Editor 和 Viewer。Admin 权限为管理员权限，具有最高的执行权限，包括对用户、Data Sources（数据源）、DashBoard（可视化仪表盘）的增删改查操作。拥有 Editor 权限的用户仅对 DashBoard（可视化仪表盘）有增删改查操作。而拥有 Viewer 权限的用户仅可以查看 DashBoard（可视化仪表盘），详情查看：[Grafana 的用户权限角色说明](https://grafana.com/docs/grafana/latest/permissions/organization_roles/)。

### 数据源配置

上文中提到，Grafana 支持多种数据源指标的可视化展示，那么数据源是在哪里配置的呢？回到 Grafana 首页，点击左侧边菜单栏的 Configuration 菜单列表中的 Data Sources 菜单进入数据源配置页面：

![Grafana 首页配置数据源入口](../images/grafana-home-datasources.png)

![数据源配置页面](../images/grafana-datasources-config.png)

发现 Grafana 已经默认配置好了 Istio 中的 Prometheus 数据源，如要添加新的数据源，可以点击绿色的 Add data source 按钮进行添加。我们点击默认配置好的 Prometheus 数据源查看具体配置：

![数据源配置详情](../images/grafana-datasources-config-detail.png)

Grafana 的 Prometheus 数据源配置主要由四部分组成：

- 数据源的名称；
- Prometheus 数据源的 HTTP 地址和访问方式（分为 Grafana 服务器访问和浏览器直接访问两种方式）；
- 抓取数据源时所使用的认证授权信息（包含各种认证授权协议和证书信息），因为 Istio 的 Prometheus 默认没有对访问设置权限且走的是 HTTP 协议，这里默认不填；
- 抓取时间间隔、超时时间和请求数据源的 HTTP 请求方法设置；

### 数据仪表盘配置

我们回到首页，点击左上角的 Home 按钮进入 Dashboard （仪表盘）总览页面，可以看到总览页面中有一组名为 Istio 的仪表盘列表：

![查看 Istio 组下的面板](../images/prometheus-grafana-istio-group.png)

选择其中的 Istio Galley Dashboard 查看具体的仪表盘信息，发现 Galley 相关指标信息都以图表的形式展示了出来：

![查看 galley 组件面板](../images/grafana-galley-dashboard.png)

回到仪表盘总览页面，右侧有仪表盘的新建（New dashboard）和导入（Import dashboard）按钮。

Grafana 的仪表盘可以通过三种方式创建：

- 页面中点击 New dashboard，通过可视化界面进行具体的参数配置；
- 页面中点击 Import dashboard，导入 JSON 配置文件（Grafana 仪表盘可以通过 JSON 数据格式的文件进行配置）；
- 在 Grafana 的配置文件中指定仪表盘的 JSON 配置文件路径，启动后默认加载所配置的仪表盘；

Istio 的仪表盘就是通过第三种方式创建的，Istio 在安装 Grafana 组件时，在 Grafana 的 Pod 中以 ConfigMap 的形式挂载了 Istio 各个组件的仪表盘 JSON 配置文件：

```bash
# 查看 Grafana 相关的 ConfigMap，发现 Istio 各组件的仪表盘配置都已配置在 ConfigMap 中
$ kubectl get cm -n istio-system |grep istio-grafana
istio-grafana                                                        2         20d
istio-grafana-configuration-dashboards-citadel-dashboard             1         20d
istio-grafana-configuration-dashboards-galley-dashboard              1         20d
istio-grafana-configuration-dashboards-istio-mesh-dashboard          1         20d
istio-grafana-configuration-dashboards-istio-performance-dashboard   1         20d
istio-grafana-configuration-dashboards-istio-service-dashboard       1         20d
istio-grafana-configuration-dashboards-istio-workload-dashboard      1         20d
istio-grafana-configuration-dashboards-mixer-dashboard               1         20d
istio-grafana-configuration-dashboards-pilot-dashboard               1         20d
# 查看 Grafana 的 Pod yaml文件，发现 Grafana 以 ConfigMap 的形式将 Istio 各个组件的仪表盘配置文件挂载到了 
# /var/lib/grafana/dashboards/istio/ 目录下
$ kubectl get pod grafana-6565cc4b48-w9dsj -n istio-system -o yaml
apiVersion: v1
kind: Pod
metadata:
...
spec:
  ...
  containers:
  - ...
    volumeMounts:
    - mountPath: /data/grafana
      name: data
    - mountPath: /var/lib/grafana/dashboards/istio/citadel-dashboard.json
      name: dashboards-istio-citadel-dashboard
      readOnly: true
      subPath: citadel-dashboard.json
    - mountPath: /var/lib/grafana/dashboards/istio/galley-dashboard.json
      name: dashboards-istio-galley-dashboard
      readOnly: true
      subPath: galley-dashboard.json
    - mountPath: /var/lib/grafana/dashboards/istio/istio-mesh-dashboard.json
      name: dashboards-istio-istio-mesh-dashboard
      readOnly: true
      subPath: istio-mesh-dashboard.json
    - mountPath: /var/lib/grafana/dashboards/istio/istio-performance-dashboard.json
      name: dashboards-istio-istio-performance-dashboard
      readOnly: true
      subPath: istio-performance-dashboard.json
    - mountPath: /var/lib/grafana/dashboards/istio/istio-service-dashboard.json
      name: dashboards-istio-istio-service-dashboard
      readOnly: true
      subPath: istio-service-dashboard.json
    - mountPath: /var/lib/grafana/dashboards/istio/istio-workload-dashboard.json
      name: dashboards-istio-istio-workload-dashboard
      readOnly: true
      subPath: istio-workload-dashboard.json
    - mountPath: /var/lib/grafana/dashboards/istio/mixer-dashboard.json
      name: dashboards-istio-mixer-dashboard
      readOnly: true
      subPath: mixer-dashboard.json
    - mountPath: /var/lib/grafana/dashboards/istio/pilot-dashboard.json
      name: dashboards-istio-pilot-dashboard
      readOnly: true
      subPath: pilot-dashboard.json
    - mountPath: /etc/grafana/provisioning/datasources/datasources.yaml
      name: config
      subPath: datasources.yaml
    - mountPath: /etc/grafana/provisioning/dashboards/dashboardproviders.yaml
      name: config
      subPath: dashboardproviders.yaml
      ...
    volumes:
  - configMap:
      defaultMode: 420
      name: istio-grafana
    name: config
  - emptyDir: {}
    name: data
  - configMap:
      defaultMode: 420
      name: istio-grafana-configuration-dashboards-citadel-dashboard
    name: dashboards-istio-citadel-dashboard
  - configMap:
      defaultMode: 420
      name: istio-grafana-configuration-dashboards-galley-dashboard
    name: dashboards-istio-galley-dashboard
  - configMap:
      defaultMode: 420
      name: istio-grafana-configuration-dashboards-istio-mesh-dashboard
    name: dashboards-istio-istio-mesh-dashboard
  - configMap:
      defaultMode: 420
      name: istio-grafana-configuration-dashboards-istio-performance-dashboard
    name: dashboards-istio-istio-performance-dashboard
  - configMap:
      defaultMode: 420
      name: istio-grafana-configuration-dashboards-istio-service-dashboard
    name: dashboards-istio-istio-service-dashboard
  - configMap:
      defaultMode: 420
      name: istio-grafana-configuration-dashboards-istio-workload-dashboard
    name: dashboards-istio-istio-workload-dashboard
  - configMap:
      defaultMode: 420
      name: istio-grafana-configuration-dashboards-mixer-dashboard
    name: dashboards-istio-mixer-dashboard
  - configMap:
      defaultMode: 420
      name: istio-grafana-configuration-dashboards-pilot-dashboard
    name: dashboards-istio-pilot-dashboard
  ...

# 进入 Grafana 的 Pod 中查看 Istio 各组件的仪表盘 JSON 配置文件
$ kubectl exec -it grafana-6565cc4b48-w9dsj sh -n istio-system
/usr/share/grafana $ ls /var/lib/grafana/dashboards/istio/
citadel-dashboard.json            istio-performance-dashboard.json  mixer-dashboard.json
galley-dashboard.json             istio-service-dashboard.json      pilot-dashboard.json
istio-mesh-dashboard.json         istio-workload-dashboard.json

```

由于仪表盘的 JSON 配置文件组成较为复杂，一般情况下仅对其做导入和导出操作，不涉及对 JSON 文件的修改，这里不对 JSON 配置文件的具体组成做详细讲解。在日常使用中创建自定义的仪表盘一般使用上面提到的第一种方式：点击 New dashboard 按钮在可视化界面中创建。点击创建按钮后，选择 Add Query 创建可视化面板：

![创建可视化面板](../images/grafana-new-dashboard.png)

进入面板的 Query 页面，在 Query 旁的数据源下拉框中选择 Istio 的 Prometheus 数据源，在 Metrics 输入框中输入 PromQL（一种用于查询 Prometheus 指标数据的特殊查询语句），右上角选择 5 min 即可查询到 Galley 组件近 5 分钟的 CPU 使用情况：

![面板 Query 配置页面](../images/grafana-panel-query.png)

点击左侧第二个圆钮进入 Visualization 配置页面，该页面可以调整图表的样式、选择可视化面板的类型等，还可以为图表中的数据设置跳转链接

![面板 Visualization 配置页面](../images/grafana-panel-visualization.png)

点击左侧第三个圆钮进入 General 配置页面，该页面主要对该面板做常规配置，如：面板标题、描述、链接等

![面板 General 配置页面](../images/grafana-panel-general.png)

点击左侧第四个圆钮进入 Alert 配置页面，该页面用于配置告警规则

![面板 Alert 配置页面](../images/grafana-panel-alert.png)

### 



