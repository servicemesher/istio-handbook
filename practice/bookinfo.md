authors: ["linda01232003"]

reviewers: ["rootsongjc "]

# Bookinfo示例

Bookinfo是Istio社区官方推荐的示例应用之一。它可以用来演示多种Istio的特性，并且它是异构的微服务架构。该应用由四个单独的微服务构成。 这个应用模仿了在线书店，可以展示书店中书籍的信息。例如页面上会显示一本书的描述，书籍的细节（ISBN、页数等），以及关于这本书的一些评论。 

Bookinfo 应用分为四个单独的微服务， 这些服务对 Istio 并无依赖，但是构成了一个有代表性的服务网格的例子：它由多个服务、多个语言构成，并且其中有一个应用会包含多个版本。 
：

- `productpage`会调用 `details` 和 `reviews` 两个微服务，用来生成页面。
- `details` 中包含了书籍的信息。
- `reviews`中包含了书籍相关的评论。它还会调用 `ratings` 微服务。
- `ratings`中包含了由书籍评价组成的评级信息。

`reviews` 微服务有 3 个版本，可用来展示各服务之间的不同的调用链路：

- v1 版本不会调用 `ratings` 服务。
- v2 版本会调用 `ratings` 服务，并使用 1 到 5 个黑色星形图标来显示评分信息。
- v3 版本会调用 `ratings` 服务，并使用 1 到 5 个红色星形图标来显示评分信息。

下图展示了这个应用的端到端架构。

![Bookinfo Application without Istio](../images/Bookinfo Application without Istio.png)

## 部署应用

要想该应用接入 Istio 服务网格，无需对应用自身做出任何改变。 您只要在 Istio 环境中对服务所在的命名空间进行yaml配置并重新启动运行就可以完成设置，下文中详细解说了将命名空间打入自动注入 sidecar 的命令和方法。 最终的部署结果将如下图所示： 

![Bookinfo Application](../images/Bookinfo Application.png)

 所有的微服务都和 Envoy sidecar 集成在一起，被集成服务所有的出入流量都被 sidecar 所劫持，这样就为外部控制准备了所需的 Hook，然后就可以利用 Istio 控制平面下发对应的 XDS 协议从而使 Envoy Sidecar 为应用提供服务路由、遥测数据收集以及策略实施等功能。 

## 启动应用服务

1. 进入 Istio 安装目录。

2. Istio 默认自动注入 Sidecar. 请为 `default` 命名空间打上标签 `istio-injection=enabled`：

   ```
   $ kubectl label namespace default istio-injection=enabled
   ```

3. 使用 `kubectl` 部署应用：

   ```
   $ kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
   ```

   如果您在安装过程中禁用了 Sidecar 自动注入功能而选择手动注入 Sidecar，请在部署应用之前使用 `istioctl kube-inject`命令修改 `bookinfo.yaml` 文件。

   ```
$ kubectl apply -f <(istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml)
   ```
   
   上面的命令会启动全部的四个服务，其中也包括了 reviews 服务的三个版本（v1、v2 以及 v3）。

   在实际部署中，微服务版本的启动过程需要持续一段时间，并不是同时完成的。

4. 确认所有的服务和 Pod 都已经正确的定义和启动：

   ```
   $ kubectl get services
   NAME                       CLUSTER-IP   EXTERNAL-IP   PORT(S)              AGE
   details                    10.0.0.31    <none>        9080/TCP             6m
   kubernetes                 10.0.0.1     <none>        443/TCP              7d
   productpage                10.0.0.120   <none>        9080/TCP             6m
   ratings                    10.0.0.15    <none>        9080/TCP             6m
   reviews                    10.0.0.170   <none>        9080/TCP             6m
   ```

   还有：

   ```
$ kubectl get pods
   NAME                                        READY     STATUS    RESTARTS   AGE
   details-v1-1520924117-48z17                 2/2       Running   0          6m
   productpage-v1-560495357-jk1lz              2/2       Running   0          6m
   ratings-v1-734492171-rnr5l                  2/2       Running   0          6m
   reviews-v1-874083890-f0qf0                  2/2       Running   0          6m
   reviews-v2-1343845940-b34q5                 2/2       Running   0          6m
   reviews-v3-1813607990-8ch52                 2/2       Running   0          6m
   ```
   
5. 要确认 Bookinfo 应用是否正在运行，请在某个 Pod 中用 `curl` 命令对应用发送请求，例如 `ratings`：

   ```
   $ kubectl exec -it $(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}') -c ratings -- curl productpage:9080/productpage | grep -o "<title>.*</title>"
   <title>Simple Bookstore App</title>
   ```


### 确定 Ingress 的 IP 和端口

现在 Bookinfo 服务启动并运行中，您需要使应用程序可以从外部访问 Kubernetes 集群，例如使用浏览器。可以用 Istio Gateway来实现这个目标。

1. 为应用程序定义 Ingress 网关：

   ```
   $ kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
   ```

1. 确认网关创建完成：

   ```
   $ kubectl get gateway
   NAME               AGE
   bookinfo-gateway   32s
   ```

1. 根据[文档](https://preliminary.istio.io/zh/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-i-p-and-ports)设置访问网关的 `INGRESS_HOST` 和 `INGRESS_PORT` 变量。确认并设置。

1. 设置 `GATEWAY_URL`：

   ```
   $ export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
   ```


## 确认可以从集群外部访问应用

可以用 `curl` 命令来确认是否能够从集群外部访问 Bookinfo 应用程序：

```
$ curl -s http://${GATEWAY_URL}/productpage | grep -o "<title>.*</title>"
<title>Simple Bookstore App</title>
```

还可以用浏览器打开网址 `http://$GATEWAY_URL/productpage`，来浏览应用的 Web 页面。如果刷新几次应用的页面，就会看到 `productpage` 页面中会随机展示 `reviews` 服务的不同版本的效果（红色、黑色的星形或者没有显示）。`reviews` 服务出现这种情况是因为我们还没有使用 Istio 来控制版本的路由。

## 应用默认目标规则

在使用 Istio 控制 Bookinfo 版本路由之前，您需要在[目标规则](https://preliminary.istio.io/zh/docs/concepts/traffic-management/#destination-rules)中定义好可用的版本，命名为 *subsets* 。

运行以下命令为 Bookinfo 服务创建的默认的目标规则：

- 如果**没有**启用双向 TLS，请执行以下命令：

  如果您是 Istio 的新手，并且使用了 `demo` [配置文件](https://preliminary.istio.io/zh/docs/setup/additional-setup/config-profiles/)，请选择此步。

  ```
$ kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml
  ```
  
- 如果**启用了**双向 TLS，请执行以下命令：

  ```
  $ kubectl apply -f samples/bookinfo/networking/destination-rule-all-mtls.yaml
  ```


等待几秒钟，以使目标规则生效。

您可以使用以下命令查看目标规则：

```
$ kubectl get destinationrules -o yaml
```

## 下一步

现在就可以使用这一应用来体验 Istio 的特性了，其中包括了流量的路由、错误注入、速率限制等。 接下来可以根据个人爱好去阅读和演练 [Istio 实例](https://preliminary.istio.io/zh/docs/tasks)。这里为新手推荐[智能路由](https://preliminary.istio.io/zh/docs/tasks/traffic-management/request-routing/)功能作为起步课程。

## 清理

结束对 Bookinfo 示例应用的体验之后，就可以使用下面的命令来完成应用的删除和清理了：

1. 删除路由规则，并销毁应用的 Pod

   ```
   $ samples/bookinfo/platform/kube/cleanup.sh
   ```

1. 确认应用已经关停

   ```
   $ kubectl get virtualservices   #-- there should be no virtual services
   $ kubectl get destinationrules  #-- there should be no destination rules
   $ kubectl get gateway           #-- there should be no gateway
   $ kubectl get pods              #-- the Bookinfo pods should be deleted
   ```

## 参考资料：
 https://preliminary.istio.io/zh/docs/examples/bookinfo 

