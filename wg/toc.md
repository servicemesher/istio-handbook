# 目录

> Draft: v0.2
> Date: 2020-03-14
> MindNode 脑图：[mind-map.mindnode](mind-map.mindnode)

![mind map](mind-map.png)

## 概念篇

### Istio Service Mesh 概述

#### Service Mesh 基本概念

#### 解决的问题（Service Mesh 在云原生中的定位和使用场景）

#### Istio 是什么，能做什么，与其他 Service Mesh 产品对比

### 架构解析

#### 控制平面

##### Pilot

##### Citadel

##### Galley

#### 数据平面

##### Envoy

##### MOSN

### 核心功能

#### 流量控制

##### 路由

###### VirtualService

###### DestinationRule

###### ServiceEntry

###### Gateway

##### 弹性能力与测试

###### 超时

###### 重试

###### 熔断

###### 故障注入

###### 流量镜像

#### 安全

##### 授权

##### 架构

##### 策略

##### 认证

##### 架构

##### 策略

#### 可观察性

##### 指标

##### 日志

##### 分布式追踪

## 实践篇

### 安装与部署

#### 安装Istio

#### 升级

#### 各环境部署

##### Kubernetes

##### VM

#### Bookinfo

#### KataCoda

### 入门实践

#### 流量控制

##### 路由

##### 熔断

##### 故障注入

##### Ingress/Egress

##### 超时

##### 重试

##### 流量镜像

#### 可观察性

##### 监控与可视化

###### Prometheus

###### Grafana

###### Kiali

##### 日志

###### ELK

##### 分布式追踪

###### Jeager

###### Zipkin

###### Skywalking

#### 安全

##### 认证

###### 认证策略

###### 网格间服务认证

###### 终端用户认证

###### 双向 TLS

##### 授权

###### HTTP 流量授权

###### TCP 流量授权

###### JWT 授权

###### Ingress 授权

###### 拒绝授权

##### Istio DNS 证书管理

### 进阶实践

#### 集成传统微服务框架

##### Spring Cloud

##### Dubbo

#### 集成注册中心配置中心

##### Consul

##### Nacos

##### Apollo

#### 对接 API 网关

##### Envoy

##### Kong

##### Nginx

#### 部署模型

## 故障排查

### 常见问题（根据情况动态添加）

### 工具（Follow official）

## 生态扩展

### 标准

#### SMI

#### UDPA

### 扩展

#### WebAssembly

#### Solo

#### Ambassador

#### Contour

## 附录

## 术语表