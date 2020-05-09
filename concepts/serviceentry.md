---
authors: ["sunzhaochang"]
reviewers: ["rootsongjc","GuangmingLuo"]
---

# ServiceEntry

Istio 支持对接 Kubernetes、Consul 等多种不同的注册中心，控制平面 Pilot 启动时，会从指定的注册中心获取 Service Mesh 集群的服务信息和实例列表，并将这些信息进行处理和转换，然后通过 xDS 下发给对应的数据平面，保证服务之间可以互相发现并正常访问。

同时，由于这些服务和实例信息都来源于服务网格内部，Istio 无法从注册中心直接获取网格外的服务，导致不利于网格内部与外部服务之间的通信和流量管理。为此，Istio 引入 ServiceEntry 实现对外通信和管理。

使用 ServiceEntry 可以将外部的服务条目添加到 Istio 内部的服务注册表中，以便让网格中的服务能够访问并路由到这些手动指定的服务。ServiceEntry 描述了服务的属性（DNS 名称、VIP、端口、协议、端点）。这些服务可能是位于网格外部（如，web APIs），也可能是处于网格内部但不属于平台服务注册表中的条目（如，需要和 Kubernetes 服务交互的一组虚拟机服务）。

## ServiceEntry 示例和属性介绍

对于网格外部的服务，下面的 ServiceEntry 示例表示网格内部的应用通过 https 访问外部的 API。
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: google
spec:
  hosts:
  - www.google.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL
```

对于在网格内部但不属于平台服务注册表的服务，使用下面的示例可以将一组在非托管 VM 上运行的 MongoDB 实例添加到 Istio 的注册中心，以便可以将这些服务视为网格中的任何其他服务。
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc-mongocluster
spec:
  hosts:
  - mymongodb.somedomain 
  addresses:
  - 192.192.192.192/24 # VIPs
  ports:
  - number: 27018
    name: mongodb
    protocol: MONGO
  location: MESH_INTERNAL
  resolution: STATIC
  endpoints:
  - address: 2.2.2.2
  - address: 3.3.3.3
```

结合上面给出的示例，这里对 ServiceEntry 涉及的关键属性解释如下：

* "hosts": 表示与该 ServiceEntry 相关的主机名，可以是带有通配符前缀的 DNS 名称。
* "address": 与服务相关的虚拟 IP 地址，可以是 CIDR 前缀的形式。
* "ports": 和外部服务相关的端口，如果外部服务的 endpoints 是 Unix socket 地址，这里必须只有一个端口。
* "location": 用于指定该服务属于网格内部（MESH_INTERNAL）还是外部（MESH_EXTERNAL）。
* "resolution": 主机的服务发现模式，可以是 NONE、STATIC、DNS。
* "endpoints": 与服务相关的一个或多个端点。
* "exportTo": 用于控制 ServiceEntry 跨命名空间的可见性，这样就可以控制在一个命名空间下定义的资源对象是否可以被其他命名空间下的 Sidecar、Gateway 和 VirtualService 使用。目前支持两种选项，"." 表示仅应用到当前命名空间，"*" 表示应用到所有命名空间。

## 使用ServiceEntry访问外部服务

Istio 提供了三种访问外部服务的方法：

1. 允许 Envoy 代理将请求传递到未在网格内配置过的任何外部服务。使用这种方法时，无法监控对外部服务的访问，也不能利用 Istio 的流量控制功能。
2. 配置 ServiceEntry 以提供对外部服务的受控访问。这是 Istio 官方推荐使用的方法。
3. 对于特定范围的 IP，完全绕过 Envoy 代理。仅当出于性能或其他原因无法使用 sidecar 配置外部访问时，才建议使用该配置方法。

这里，我们重点讨论第2种方式，也就是使用 ServiceEntry 完成对网格外部服务的受控访问。

对于 sidecar 对外部服务的处理方式，istio 提供了两种选项: 

* ALLOW_ANY: 默认值，表示 Istio 代理允许调用未知的外部服务。上面的第一种方法就使用了该配置项。
* REGISTRY_ONLY:  Istio 代理会阻止任何没有在网格中定义的 HTTP 服务或 ServiceEntry 的主机。

可以使用下面的命令查看当前所使用的模式:
```bash
$ kubectl get configmap istio -n istio-system -o yaml | grep -o "mode: ALLOW_ANY"
mode: ALLOW_ANY
```
如果当前使用的是 ALLOW_ANY 模式，可以使用下面的命令切换为 REGISTRY_ONLY 模式:
```bash
$ kubectl get configmap istio -n istio-system -o yaml | sed 's/mode: ALLOW_ANY/mode: REGISTRY_ONLY/g' | kubectl replace -n istio-system -f -
configmap "istio" replaced
```

在 REGISTRY_ONLY 模式下，需要使用 ServiceEntry 才能完成对外部服务的访问。当创建如下的 ServiceEntry 时，服务网格内部的应用就可以正常访问 httpbin.org 服务了。
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: httpbin-ext
spec:
  hosts:
  - httpbin.org
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
  location: MESH_EXTERNAL
```

## 管理外部流量

使用 ServiceEntry 可以使网格内部服务发现并访问外部服务，除此之外，还可以对这些到外部服务的流量进行管理。结合 VirtualService 为对应的 ServiceEntry 配置外部服务访问规则，如请求超时、故障注入等，实现对指定服务的受控访问。

下面的示例就是为外部服务 httpbin.org 设置了超时时间，当请求时间超过3s时，请求就会直接中断，避免因外部服务访问时延过高而影响内部服务的正常运行。由于外部服务的稳定性通常无法管控和监测，这种超时机制对内部服务的正常运行具有重要意义。
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin-ext
spec:
  hosts:
  - httpbin.org
  http:
  - timeout: 3s
    route:
      - destination:
          host: httpbin.org
        weight: 100
```

同样的，我们也可以为 ServiceEntry 设置故障注入规则，为系统测试提供基础。下面的示例表示为所有访问 httpbin.org 服务的请求注入一个403错误。
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
 name: httpbin-service
spec:
 hosts:
 - httpbin.org
 http:
 - route:
   - destination:
       host: httpbin.org
   fault:
     abort:
       percent: 100
       httpStatus: 403
```

## 小结

Istio 推荐使用 ServiceEntry 实现对网格外部服务的受控访问，本节围绕 ServiceEntry 的概念、属性和使用等方面进行了介绍。通过使用 ServiceEntry，可以使网格内部的服务正常发现和路由到外部服务，并在此基础上，结合 VirtualService 实现请求超时、故障注入等流量管控机制。
