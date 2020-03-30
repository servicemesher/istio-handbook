---
authors: ["tony-Ma-yami"]
reviewers: [""]
---

# 流量镜像

流量镜像，也叫作阴影发布，是指将线上的真实流量复制一份到其他服务中去，我们通过流量复制转发达到在不影响线上服务的情况下对流量或请求内容做具体分析。这个功能在某些测试坏境不可复现的产品Bug定位分析，或者某些指标收集上起着巨大的作用。

Istio架构里，镜像流量是通过 VirtualService 中的 HTTPRoute 配置项下的`mirror`与`mirrorPercent`配置项来实现的。

- **mirror**：配置一个 Destination 类型的对象，这里就是我们镜像流量转发的服务地址。具体的 Destination 对象配置属性请参考 VirtualService 相关介绍页。
- **mirrorPercent**：这个配置项用来指定有多少的原始流量将被转发到镜像流量中去。

举例：在下面的例子中，表示将请求`v1`版本的`httpbin`服务的流量镜像到`v2`版本的`httpbin`服务上一份，并且镜像比例为`100%`，表示将全部流量镜像。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
    - httpbin
  http:
  - route:
    - destination:
        host: httpbin
        subset: v1
      weight: 100
    mirror:
      host: httpbin
      subset: v2
    mirror_percent: 100
```

### 总结

镜像发布在对我们定位产品坏境上Bug的确是一个很巧妙的设计，但笔者不太建议大家随意使用该配置。原因是首先镜像配置是在产品坏境的 VirtualService 上操作的，这就意味着如果我们要使用镜像发布，势必需要修改产品的 VirtualService 配置，而产品坏境上的配置，在非必要的情况下，最好不要动。第二个原因是，很多时候的粗心可能会在解决问题后忘记删除镜像流量配置。那么可以想象一下，如果这个接口是商品详情接口，并且遇到了双11会发生什么。所以，笔者认为，对于线上问题，日志分析仍然是第一选择。