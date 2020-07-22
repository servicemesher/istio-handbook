---
authors: ["lianghao"]
reviewers: [""]
---

# Grafana

## Grafana 简介

Grafana 是一款开源的指标数据可视化工具，有着功能齐全的度量仪表盘、图表等时序数据展示面板，支持 Zabbix、InfluentDB、Prometheus、Elasticsearch、MySQL 等数据源的指标展示，详情查看：[Grafana 支持的数据源类型](https://grafana.com/docs/grafana/latest/features/datasources/#supported-data-sources/)。

总而言之，Grafana 是一款提供了将时间序列数据库（TSDB）数据转换为精美的图形和可视化面板的工具。在 Istio 安装完成后，默认已经将 Grafana 服务以 Pod 的方式运行起来，同时也配置了 Istio 中的 Prometheus 作为数据源，定时地从 Prometheus 中采集 Istio 各组件的指标数据，进行可视化展示。

## Grafana 的使用和配置

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

我们点击左上角的 Home 按钮查看名为 Istio 分组下的面板：

![查看 Istio 组下的面板](../images/prometheus-grafana-istio-group.png)

选择其中的 Istio Galley Dashboard 查看具体的面板信息，发现 Galley 相关指标信息都以图表的形式展示了出来：

![查看 galley 组件面板](../images/grafana-galley-dashboard.png)

