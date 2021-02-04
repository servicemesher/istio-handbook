---
authors: ["tony-Ma-yami"]
reviewers: ["malphi"]
---

# 流量镜像

## 流量镜像是什么

流量镜像（Mirroring / traffic-shadow），也叫作影子流量，是指通过一定的配置将线上的真实流量复制一份到镜像服务中去，我们通过流量镜像转发以达到在不影响线上服务的情况下对流量或请求内容做具体分析的目的，它的设计思想是只做转发而不接收响应（fire and forget）。这个功能在传统的微服务架构里是很难做到的，一方面，传统服务之间的通讯是由 SDK 支持的，那么对流量镜像的支持就代表着在业务服务逻辑中有着镜像逻辑相关代码的侵入，这会影响业务服务的代码的整洁性。另一方面，流量镜像的功能是需要非常灵活的，多维度，可动态管控的一个组件。如果将这样的一个组件集成到 SDK 中去完成它的使命后却无法及时清除，这样的设计势必是臃肿而繁琐。所幸的是，随着微服务架构的发展，Service Mesh 登上了历史舞台，成为新一代的服务架构引领者。而 Istio 作为 Service Mesh 优秀的落地架构，利用它本身使用 Envoy 代理转发流量的特性，轻松的支持了流量镜像的功能，再加上它的实现不需要任何代码的侵入，只需要在配置文件中简单加上几个配置节即可完成，这些设计以及实现方式足以让开发测试人员眼前一亮。那么流量镜像到底能解决什么样具体的问题呢？

## 流量镜像能够为我们带来什么

很多情况下，当我们对服务做了重构，或者我们对项目做了重大优化时，怎么样保证服务是健壮的呢？在传统的服务里，我们只能通过大量的测试，模拟在各种情况下服务的响应情况。虽然也有手工测试、自动化测试、压力测试等一系列手段去检测它，但是测试本身就是一个样本化的行为，即使测试人员再完善它的测试样例，无法全面的表现出线上服务的一个真实流量形态。往往当项目发布之后，总会出现一些意外，比如你服务里收到客户使用的某些数据库不认识的特殊符号，再比如用户在本该输入日期的输入框中输入了 “--” 字样的字符，又比如用户使用乱码替换你的 token 值批量恶意攻击服务等等，这样的情况屡见不鲜。数据的多样性，复杂性决定了开发人员在开发阶段根本是无法考虑周全的。

而流量镜像的设计，让这类问题得到了最大限度的解决。流量镜像讲究的不再是使用少量样本去评估一个服务的健壮性，而是在不影响线上坏境的前提下将线上流量持续的镜像到我们的预发布坏境中去，让重构后的服务在上线之前就结结实实地接受一波真实流量的冲击与考验，让所有的风险全部暴露在上线前夕，通过不断的暴露问题，解决问题让服务在上线前夕就拥有跟线上服务一样的健壮性。由于测试坏境使用的是真实流量，所以不管从流量的多样性，真实性，还是复杂性上都将能够得以展现，同时预发布服务也将表现出其最真实的处理能力和对异常的处理能力。运用这种模式，一方面，我们将不会再跟以前一样在发布服务前夕内心始终忐忑不安，只能祈祷上线之后不会出现问题。另一方面，当大量的流量流入重构服务之后，开发过程中难以评估的性能问题也将完完整整的暴露出来，此时开发人员将会考虑它服务的性能，测试人员将会更加完善他们的测试样例。通过暴露问题，解决问题，再暴露问题，再解决问题的方式循序渐进地完善预发布服务来增加我们上线的成功率。同时也变相的促进我们开发测试人员技能水平的提高。

当然，流量镜像的作用不仅仅只是解决上面这样的场景问题，我们可以根据它的特性，解决更多的问题。比如，假如我们在上线后突然发现一个线上问题，而这个问题在测试坏境中始终不能复现。那么这个时候我们就能利用它将异常流量镜像到一个分支服务中去，然后我们可以随意在这个分支服务上进行分析调试，这里所说的分支服务，可以是原服务的只用于问题分析而不处理正式业务的副本服务，也可以是一个只收集镜像流量的组件类服务。又比如突然需要收集某个时间段某些流量的特征数据做分析，像这种临时性的需求，使用流量镜像来处理非常合适，既不影响线上服务的正常运转，也达到了收集分析的目的。

## 流量镜像的实现原理

实际上在 Istio 中，服务间的通讯都是被 Envoy 代理拦截并处理的， Istio 流量镜像的设计也是基于 Envoy 特性实现的。它的流量转发如下图所示。可以看到，当流量进入到`Service A`时，因为在`Service A`的 Envoy 代理上配置了流量镜像规则，那么它首先会将原始流量转发到`v1`版本的 `Service B`服务子集中去 。同时也会将相同的流量复制一份，异步地发送给`v2`版本的`Service B` 服务子集中去，可以明显的看到，`Service A` 发送完镜像流量之后并不关心它的响应情况。

![concepts-traffic-shadow](../images/concepts-traffic-shadow-01.png)

在很多情况下，我们需要将真实的流量数据与镜像流量数据进行收集并分析，那么当我们收集完成后应该怎样区分哪些是真实流量，哪些是镜像流量呢？ 实际上，Envoy 团队早就考虑到了这样的场景，他们为了区分镜像流量与真实流量，在镜像流量中修改了请求标头中 `host` 值来标识，它的修改规则是：在原始流量请求标头中的 `host` 属性值拼接上`“-shadow”` 字样作为镜像流量的 `host` 请求标头。

为了能够更清晰的对比出原始流量与镜像流量的区别，我们使用以下的一个示例来说明：

如下图所示，我们发起一个`http://istio.gateway.xxxx.tech/serviceB/request/info`的请求，请求首先进入了`istio-ingressgateway` ，它是一个 Istio 的 Gateway 资源类型的服务，它本身就是一个 Envoy 代理。在这个例子里，就是它对流量进行了镜像处理。可以看到，它将流量转发给`v1`版本`Service B`服务子集的同时也复制了一份流量发送到了`v2`版本的`Service B`服务子集中去。

![concepts-traffic-shadow-request](../images/concepts-traffic-shadow-request.png)

在上面的请求链中，请求标头数据有什么变化呢？下图收集了它们请求标头中的所有信息，可以明显的对比出正式流量与镜像流量请求标头中`host`属性的区别（部分相同的属性值过长，这里只截取了前半段）。从图中我们可以看出，首先就是host属性值的不同，而区别就是多了一个`“-shadow”`的后缀。再者发现`x-forwarded-for`属性也不相同，`x-forwarded-for`协议头的格式是：`x-forwarded-for: client1, proxy1, proxy2`， 当流量经过 Envoy 代理时这个协议头将会把代理服务的 IP 添加进去。实例中`10.10.2.151`是我们云主机的 IP，而`10.10.2.121`是`istio-ingressgateway`所对应Pod的 IP 。从这里也能看到，镜像流量是由`istio-ingressgatway`发起的。除了这两个请求标头的不同，其他配置项是完全一样的。

![concepts-traffic-shadow-header](../images/concepts-traffic-shadow-header.png)

## 流量镜像的配置

上面我们介绍了流量镜像的原理及使用场景，接下来我们再介绍下流量的镜像如何配置才能生效。在 Istio 架构里，镜像流量是借助于 VirtualService 这个资源中的 `HTTPRoute` 配置项的`mirror`与`mirrorPercent`这两项子配置项来实现的，这两个配置项的定义也是非常的简单。

- **mirror**：配置一个 Destination 类型的对象，这里就是我们镜像流量转发的服务地址。具体的 [**VirtualService**](https://www.servicemesher.com/istio-handbook/concepts/virtualservice.html) 配置与[**DestinationRule**](https://www.servicemesher.com/istio-handbook/concepts/destinationrule.html) 对象配置属性请参考相关介绍页。
- **mirrorPercent**：配置一个数值，这个配置项用来指定有多少的原始流量将被转发到镜像流量服务中去，它的有效值为`0~100`，如果配置成`0`则表示不发送镜像流量。

下面的例子就是我们在示例中使用到的`Service B`的镜像流量配置，其中，`mirror.host`配置项是配置一个域名或者在Istio 注册表中注册过的服务名称，可以看到，该配置指定了镜像流量需要发送的目标服务地址为`serviceB`。`mirror.subset`配置项配置一个`Service B`服务的服务子集名称 ，指定了要将镜像流量镜像到`v2`版本的`Service B`服务子集中去。`mirror_percent`配置将`100%`的真实流量进行镜像发送。所以下面的配置整体表示当流量到来时，将请求转发到`v1`版本的`service B`服务子集中，再以镜像的方式发送到`v2`版本的`service B`服务上一份，并将真实流量全部镜像。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: serviceB
spec:
  hosts:
  - istio.gateway.xxxx.tech
  gateways:
  - ingressgateway.istio-system.svc.cluster.local
  http:
  - match:
    - uri:
        prefix: /serviceB
    rewrite:
      uri: /
    route:
    - destination:
        host: serviceB
        subset: v1
    mirror:
      host: serviceB
      subset: v2
    mirror_percent: 100
```

`service B` 服务对应的 DestinationRule 配置如下 ：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: serviceB
  namespace: default
spec:
  host: serviceB
  subsets:
  - name: v2
    labels:
      version: v2
  - name: v1
    labels:
      version: v1
```

## 实践

我们使用`httpbin`服务的两个版本`v1`,`v2`来对流量镜像功能进行测试。`v1`版本用来接收原始流量，`v2`版本用来接收镜像流量，这两个版本的服务使用相同的镜像名称，只在编写 Kubernetes 的 Deployment 资源时使用`version:v1`和`version:v2`这两个不同的 Labels 来区分。

我们的测试流程是：先将两个版本的服务都部署起来，并设置它的默认路由策略将所有流量转发到`v1`版本中，测试正常后我们通过配置 `v1` 版本服务的 VirtualService 来设置将流量镜像到`v2`版本的服务中，测试流量镜像成功后，再修改流量镜像的采样百分比来观察实际的表现效果。

### 部署测试服务

首先，通过以下命令来部署两个版本的`httpbin`服务并设置它的默认路由策略将所有流量转发到`v1`版本中。

- 创建`v1`版本`httpbin`服务的`Deployment`：

```shell
$ cat <<EOF | istioctl kube-inject -f - | kubectl create -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin-v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
      version: v1
  template:
    metadata:
      labels:
        app: httpbin
        version: v1
    spec:
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        command: ["gunicorn", "--access-logfile", "-", "-b", "0.0.0.0:80", "httpbin:app"]
        ports:
        - containerPort: 80
EOF

```

- 创建`v2`版本`httpbin`服务的`Deployment`：

```shell
$ cat <<EOF | istioctl kube-inject -f - | kubectl create -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
      version: v2
  template:
    metadata:
      labels:
        app: httpbin
        version: v2
    spec:
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: httpbin
        command: ["gunicorn", "--access-logfile", "-", "-b", "0.0.0.0:80", "httpbin:app"]
        ports:
        - containerPort: 80
EOF
```

- 创建`httpbin`服务的`Service`：

```shell
$ kubectl create -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: httpbin
  labels:
    app: httpbin
spec:
  ports:
  - name: http
    port: 8000
    targetPort: 80
  selector:
    app: httpbin
EOF

```

- 创建`httpbin`服务的默认路由策略，如下代码所示。
  - VirtualService 资源是 Istio 中的用来配置路由、流量转发权重占比等规则的一种 Kubernetes 自定义资源（CRDS），具体介绍详见： https://www.servicemesher.com/istio-handbook/concepts/virtualservice.html。
  - DestinationRule 资源是 Istio 中用来配置客户端负载策略、连接池配置、异常点检测、服务子集等功能的一种 Kubernetes 自定义资源（CRDS），具体介绍详见：https://www.servicemesher.com/istio-handbook/concepts/destinationrule.html

```shell
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
    - '*'
  gateways:
    - istio-system/ingressgateway
  http:
  - match:
    - uri:
        prefix: /httpbin
    rewrite:
      uri: /
    route:
    - destination:
        host: httpbin
        subset: v1
      weight: 100
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
EOF
```

部署好以上服务之后，我们来验证下服务是否正常启动，资源是否全部创建完成。

```shell
$ kubectl get svc,po,vs,dr
```

```shell
NAME              TYPE         CLUSTER-IP       EXTERNAL-IP  PORT(S)   AGE
service/httpbin   ClusterIP    172.20.227.164   <none>       8000/TCP  72s

NAME                              READY   STATUS        RESTARTS   AGE
pod/httpbin-v1-595647cbcc-h6s5m   2/2     Running       0          85s
pod/httpbin-v2-96486cc9d-jcz9f    2/2     Running       0          79s

NAME                                          GATEWAYS  HOSTS         AGE
virtualservice.networking.istio.io/httpbin              [httpbin]     34s

NAME                                              HOST          AGE
destinationrule.networking.istio.io/httpbin       httpbin       34s
```

接下来我们使用如下命令向`httpbin`服务发送一些流量请求。

```shell
# 将 istio-gateway 的 service 地址导入到环境变量中去。
$ export GATEWAY_URL=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# 对 httpbin 服务发送请求。
$ curl http://${GATEWAY_URL}/httpbin/headers
{
  "headers": {
    "Accept": "*/*",
    "Content-Length": "0",
    "Host": "abcc68c3b126c11eab2630a8eb933271-1419444084.us-west-2.elb.amazonaws.com",
    "User-Agent": "curl/7.29.0",
    "X-B3-Parentspanid": "1a8b1eced190c3a2",
    "X-B3-Sampled": "0",
    "X-B3-Spanid": "f2212d5df93bd3da",
    "X-B3-Traceid": "3bbee4b27efecd5d1a8b1eced190c3a2",
    "X-Envoy-Internal": "true",
    "X-Envoy-Original-Path": "/httpbin/headers",
    "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/default/sa/default;Hash=855455e7c4d742bb7ce76df7c3c3c9ecf4b978e2421097ae74f6b2deec91af9c;Subject=\"\";URI=spiffe://cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"
  }
}
```

分别查看`v1`和`v2`版本`httpbin`服务的日志信息发现，`v1`版本接收到了请求并打印了日志，`v2`版本则没有打印日志信息。

```shell
# 将 v1 版本 httpbin 服务的 pod 名称导入到环境变量中去。
$ export V1_POD=$(kubectl get pod -l app=httpbin,version=v1 -o jsonpath={.items..metadata.name})

# 查看 v1 版本 httpbin 服务对应 Pod 的应用日志信息。
$ kubectl logs -f $V1_POD -c httpbin
127.0.0.1 - - [30/Apr/2020:02:21:39 +0000] "GET //headers HTTP/1.1" 200 694 "-" "curl/7.29.0"
```

```shell
# 将 v2 版本 httpbin 服务的 pod 名称导入到环境变量中去。
$ export V2_POD=$(kubectl get pod -l app=httpbin,version=v2 -o jsonpath={.items..metadata.name})

# 查看 v2 版本 httpbin 服务对应 Pod 的应用日志信息。
$ kubectl logs -f $V2_POD -c httpbin
<none>
```



## 流量镜像测试

流量镜像最常用的场景是在一个集群内，将流量从服务的一个版本镜像到同源服务的另外一个版本中，比如在这个示例中，把访问`v1`版本`httpbin`服务的流量镜像到`v2`版本的`httpbin`服务中。它的场景模型如下图所示：

![场景模型：一个集群中不同版本服务间流量镜像](../images/shadow-in-one-mesh.png)

在以上配置下，使用一样的`kubectl apply -f  <file>` 命令修改  Virtual Service 资源。

```shell
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
    - '*'
  gateways:
    - istio-system/ingressgateway
  http:
  - match:
    - uri:
        prefix: /httpbin
    rewrite:
      uri: /
    route:
    - destination:
        host: httpbin
        subset: v1
      weight: 100
    mirror:
      host: httpbin
      subset: v2
    mirror_percent: 100
EOF
```

流量镜像相关的配置具体如下：

```yaml
    ...
    mirror:
      host: httpbin
      subset: v2
    mirror_percent: 100
```

`mirror`表示配置一个 Destination 类型的对象。表示将该请求的流量镜像到别的服务中去。Destination 类型可配置以下子属性：

- **host**：必配项，是一个 string 类型的值。表示上游资源的地址，host 可配置的资源为所有在集群中注册的所有的 service 域名或者资源， 可以是一个 Kubernetes 的 service，也可以是一个 Consul 的 services， 或者是一个 ServiceEntry 类型的资源。同样，这里配置的域名可以是一个 FQDN（Fully Qualified Domain Name - 全限定域名），也可以是一个短域名。注意，如果您是 Kubernetes 用户，那么这里的资源地址补全同样依赖于该 VirtualService 所在的命名空间，比如在 prod 命名空间下配置了 destination.host=reviews，那么 host 最终将被解释并补全为 reviews.prod.svc.cluster.local。
- **subset**：非必配项，是一个 string 类型的值。表示访问上游工作负载组中指定版本的工作负载，如果需要使用 subset 配置，必须先在 DestinationRule 中定义一个 subset 组，而这里的配置值即为 subset 组中某一个 subset 的 name 值。
- **port**：非必配项，是一个 PortSelector 类型的对象。指定上游工作负载的端口号，如果上游工作负载只公开了一个端口，那么这个配置将可以不用配置。


`mirror_percent`配置表示将`100%`的流量镜像到`v2`版本的`httpbin`服务上。

现在我们给`v1`版本`httpbin`服务发送一些流量请求，发送完成后，查看`v2`版本`httpbin`服务的日志信息，查看它是否接收到了镜像过来的流量。

```shell
# 对 httpbin 服务发送请求。
$ curl http://${GATEWAY_URL}/httpbin/headers
```

```shell
# 查看 v2 版本 httpbin 服务对应 Pod 的应用日志信息。
$ kubectl logs -f $V2_POD -c httpbin
127.0.0.1 - - [30/Apr/2020:03:10:56 +0000] "GET //headers HTTP/1.1" 200 668 "-" "curl/7.29.0"
```

观察后发现，`v2`版本的`httpbin`服务中打印了日志信息，这表示我们的流量镜像配置已经生效。

接下来我们发送5次请求给`v1`版本的`httpbin`服务并查看`v2`版本`httpbin`服务的日志信息，观察是否每一个请求都会被镜像。

```shell
# 查看 v2 版本 httpbin 服务对应 Pod 的最新5行应用日志信息。
$ kubectl logs -f $V2_POD -c httpbin --tail 5
127.0.0.1 - - [30/Apr/2020:03:18:44 +0000] "GET //headers HTTP/1.1" 200 668 "-" "curl/7.29.0"
127.0.0.1 - - [30/Apr/2020:03:18:44 +0000] "GET //headers HTTP/1.1" 200 668 "-" "curl/7.29.0"
127.0.0.1 - - [30/Apr/2020:03:18:45 +0000] "GET //headers HTTP/1.1" 200 668 "-" "curl/7.29.0"
127.0.0.1 - - [30/Apr/2020:03:18:47 +0000] "GET //headers HTTP/1.1" 200 668 "-" "curl/7.29.0"
127.0.0.1 - - [30/Apr/2020:03:18:48 +0000] "GET //headers HTTP/1.1" 200 668 "-" "curl/7.29.0"

```

可以看到新增了5条日志信息，所有流量都被复制了一份，这说明我们的`mirror_percent: 100`配置生效了。

接下来，我们设置镜像到`v2`版本服务的流量采样率为50%。一样使用`kubectl apply -f <file>`命令修改`mirror_percent`属性值为`50`。

```shell
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
    - '*'
  gateways:
    - istio-system/ingressgateway
  http:
  - match:
    - uri:
        prefix: /httpbin
    rewrite:
      uri: /
    route:
    - destination:
        host: httpbin
        subset: v1
      weight: 100
    mirror:
      host: httpbin
      subset: v2
    mirror_percent: 50
EOF
```

此时再连续发送20次请求给`v1`版本的`httpbin`服务，同时观察`v2`版本`httpbin`服务的日志信息变化。我们发现`v1`版本的`httpbin`服务在被调用20次时，只有9次请求镜像到了`v2`版本的服务当中，并不是准确的50%。实际上，Istio 允许这种误差的存在，该属性值只能表示它大概的一个百分比，读者可以再尝试发送更多的请求来观察效果。

除了上面的场景，我们经常也需要将某个服务的流量从一个 service mesh 网格中镜像到另外一个 service mesh 网格中，比如将生产环境的流量镜像到测试环境中去。它的场景模型如下图所示：

![场景模型：将一个集群的流量镜像到另外一个集群当中](../images/shadow-in-two-mesh.png)

这种场景的实现方式与前面介绍的第一种场景实现方式一样，也是修改`v1`版本`httpbin`服务的 VirtualService 资源。如下只需要替换`${OTHER_MESH_GATEWAY_URL}`为测试环境的真实主机地址即可：

```shell
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
    - '*'
  gateways:
    - istio-system/ingressgateway
  http:
  - match:
    - uri:
        prefix: /httpbin
    rewrite:
      uri: /
    route:
    - destination:
        host: httpbin
        subset: v1
      weight: 100
    mirror:
      host: ${OTHER_MESH_GATEWAY_URL}
    mirror_percent: 100
EOF
```

至此，我们就可以完成将一个集群中的流量镜像转发到另外一个集群的操作。当然，这只是简单的在不同集群中流量镜像的示例，在实际操作过程中，我们可能需要在业务集群中使用 ServiceEntry 将对测试环境的调用交给 Envoy 托管，也可能需要配置 TLS 实现安全访问等，这些功能这里不再一一展开说明。

实际上，对于生产环境中一些比较有代表性的流量，我们使用流量镜像复制到其他集群之后，可以将这些流量通过文件或者日志的形式收集起来。而这些数据既可以作为自动化测试脚本的数据源，也可以作为大数据分析客户画像等功能的部分数据源，通过对这些数据的提取以及二次开采，分析客户的使用习惯，行为等特征信息，将加工后的数据应用到推荐服务当中，可以有效帮助实现系统的千人千面，定向推荐等功能。

### 环境清除

使用以下命令清除上面创建的所有资源。

```shell
$ kubectl delete virtualservice httpbin
$ kubectl delete destinationrule httpbin
$ kubectl delete deploy httpbin-v1 httpbin-v2
$ kubectl delete svc httpbin
```

## 总结

- 镜像流量从设计上讲与业务服务解耦，这意味着是一种无侵入式的设计。这样我们就在业务服务上无须考虑副本流量的问题，专注于业务处理。而且在进行镜像流量的配置时也不用考虑对业务服务的影响。
- 从原理上讲，它使用这种异步发送不关心响应（fire and forget）的方式，丝毫不会影响到我们的线上坏境正常运作。
- 从配置复杂度上讲简单并且容易上手，通过简单的几个配置项即可完成，这将大大减轻了我们的学习成本，也将大大提高了我们调试、测试的效率。
- 从表现上讲完美复制真实的流量，除了请求标头的`host`、`x-forwarded-for`等属性，我们几乎可以认为它就是真实的流量。

综上介绍，Istio的镜像流量功能是一个非常优雅而实用的功能，我们的微服务架构有了这么强大的功能来加持，对我们调试、测试、预上线演练等工作是一个重大技术支撑。我们可以用它来收集补充我们的测试案例，使用峰值时间段流量来测试服务的抗压能力以及检测部分容错容灾能力，当绝大部分问题都在上线之前得到暴露、验证并解决时，上线这件事不再让人惧怕，它将变成一件非常容易的事情。

最后，我们说明一下，根据镜像流量特性，前面提到的使用分支服务调试测试环境上无法复现的线上问题是一个很巧妙的应用，虽然它能够支持这样的操作，但非常不建议这样做。一方面原因是调试这样的动作本身就不应该在线上坏境进行，况且操作线上 VirtualService 是非常敏感的，在VirtualService 更新的时候，很容易引起请求调用链的卡顿。另一方面，在调试完毕后，有时候有可能由于粗心会忘记删除调用链路中某处的镜像流量配置。那么可以想象一下，如果这些流量是商品详情页面的相关的，并且遇到了双11大促节日会发生什么样的后果。

## 参考

1. [VirtualService 介绍](https://www.servicemesher.com/istio-handbook/concepts/virtualservice.html)
2. [DestinationRule 介绍](https://www.servicemesher.com/istio-handbook/concepts/destinationrule.html)
