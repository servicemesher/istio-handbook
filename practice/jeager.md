---
authors: ["lyzhang1999"]
reviewers: ["rootsongjc","GuangmingLuo","malphi"]
---

# Jeager
Jeager 的简介

## Envoy-Jeager 架构

## 环境准备
部署 Bookinfo 示例；
1. 安装
    使用 Demo 配置安装，会自动安装 tracing 组件。也可以手动安装：
    ```
    $ istioctl manifest apply --set values.tracing.enabled=true
    ✔ Istio core installed
    ✔ Istiod installed
    ✔ Ingress gateways installed
    ✔ Addons installed
    ✔ Installation complete
    ```
    此命令会启用一个“开箱即用”的 Jaeger 演示环境。
    > 注意：默认采样率为 1%，你可以通过 `--set values.pilot.traceSampling=<Value>` 来配置采样率。Value 范围在 0.0 到 100.0 之间，精度为 0.01 。例如，Value 配置 0.01 意味着 10000 请求中跟踪 1 个请求。

2. 现有实例
    ```
    istioctl manifest apply --set values.global.tracer.zipkin.address=<jaeger-collector-service>.<jaeger-collector-namespace>:9411
    ```

## 访问 Jeager Dashboard


## 参考
