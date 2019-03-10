---
owners: ["rootsongjc"]
reviewers: [""]
description: "本文是 xDS 协议中 SDS 的解析，译自 Envoy 官方文档。"
publishDate: 2019-03-10
updateDate: 2019-03-10
tags: ["envoy","sds","xds"]
category: "translation"
---

# SDS（秘钥发现服务）

SDS（秘钥发现服务）是 Envoy 1.8.0 版本起开始引入的服务。可以在 `bootstrap.static_resource` 的 [secrets](https://www.envoyproxy.io/docs/envoy/latest/api-v2/config/bootstrap/v2/bootstrap.proto#envoy-api-field-config-bootstrap-v2-bootstrap-staticresources-secrets) 配置中为 Envoy 指定 TLS 证书（[secret](https://www.envoyproxy.io/docs/envoy/latest/api-v2/config/bootstrap/v2/bootstrap.proto#envoy-api-field-config-bootstrap-v2-bootstrap-staticresources-secrets)）。也可以通过秘钥发现服务（SDS）远程获取。Istio 预计将在 1.1 版本中支持 SDS。

SDS 带来的最大的好处就是简化证书管理。要是没有该功能的话，我们就必须使用 Kubernetes 中的 [secret 资源](https://jimmysong.io/kubernetes-handbook/concepts/secret.html)创建证书，然后把证书挂载到代理容器中。如果证书过期，还需要更新 secret 和需要重新部署代理容器。使用 SDS，中央 SDS 服务器将证书推送到所有 Envoy 实例上。如果证书过期，服务器只需将新证书推送到 Envoy 实例，Envoy 可以立即使用新证书而无需重新部署。

如果 listener server 需要从远程获取证书，则 listener server 不会被标记为 active 状态，在获取证书之前不会打开其端口。如果 Envoy 由于连接失败或错误的响应数据而无法获取证书，则 listener  server 将被标记为 active，并且打开端口，但是将重置与端口的连接。

上游集群的处理方式类似，如果需要通过 SDS 从远程获取集群客户端证书，则不会将其标记为 active 状态，在获得证书之前也它不会被使用。如果 Envoy 由于连接失败或错误的响应数据而无法获取证书，则集群将被标记为 active，可以处理请求，但路由到该集群的请求都将被拒绝。

使用 SDS 的静态集群需定义 SDS 集群（除非使用不需要集群的 Google gRPC），则必须在使用静态集群之前定义 SDS 集群。

Envoy 代理和 SDS 服务器之间的连接必须是安全的。可以在同一主机上运行 SDS 服务器，使用 Unix Domain Socket 进行连接。否则，需要代理和 SDS 服务器之间的 mTLS。在这种情况下，必须静态配置 SDS 连接的客户端证书。

## SDS Server

SDS server 需要实现 [SecretDiscoveryService](https://github.com/envoyproxy/envoy/blob/master/api/envoy/service/discovery/v2/sds.proto) 这个 gRPC 服务。遵循与其他 [xDS](https://github.com/envoyproxy/data-plane-api/blob/master/XDS_PROTOCOL.md) 相同的协议。

## SDS 配置

SDS 支持静态配置也支持动态配置。

**静态配置**

可以在`static_resources`  的 `secrets` 中配置 TLS 证书。

**动态配置**

从远程 SDS server 获取 secret。

- 通过 Unix Domain Socket 访问 gRPC SDS server。
- 通过 UDS 访问 gRPC SDS server。
- 通过 Envoy gRPC 访问 SDS server。

配置详情请参考 [Envoy 官方文档](https://www.envoyproxy.io/docs/envoy/latest/configuration/secret)。

## 参考

- [Secret discovery service (SDS) - envoyproxy.io](https://www.envoyproxy.io/docs/envoy/latest/configuration/secret)