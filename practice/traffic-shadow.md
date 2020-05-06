---
authors: ["tony-Ma-yami"]
reviewers: ["malphi"]
---

# 流量镜像

在前面的章节中，我们介绍了流量镜像的概念以及它的一些实现原理，明确了流量镜像能帮助我们解决什么问题，具体参考[**流量镜像概念介绍**](https://www.servicemesher.com/istio-handbook/concepts/traffic-shadow.html)。本章节则介绍下实践中如何配置流量镜像。

我们使用`httpbin`服务的两个版本`v1`,`v2`来对流量镜像功能进行测试。`v1`版本用来接收原始流量，`v2`版本用来接收镜像流量，这两个版本的服务使用相同的镜像名称，只在编写 Kubernetes 的 Deployment 资源时使用`version:v1`和`version:v2`这两个不同的 Labels 来区分。

我们的测试流程是：先将两个版本的服务都部署起来，并设置它的默认路由策略将所有流量转发到`v1`版本中，测试正常后我们通过配置 `v1` 版本服务的 VirtualService 来设置将流量镜像到`v2`版本的服务中，测试流量镜像成功后，再修改流量镜像的采样百分比来观察实际的表现效果。

## 部署测试服务

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

## 环境清除

使用以下命令清除上面创建的所有资源。

```shell
$ kubectl delete virtualservice httpbin
$ kubectl delete destinationrule httpbin
$ kubectl delete deploy httpbin-v1 httpbin-v2
$ kubectl delete svc httpbin 
```



## 参考

1. [VirtualService 介绍](https://www.servicemesher.com/istio-handbook/concepts/virtualservice.html)
2. [DestinationRule 介绍](https://www.servicemesher.com/istio-handbook/concepts/destinationrule.html)



