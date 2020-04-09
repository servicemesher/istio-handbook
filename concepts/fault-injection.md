---
authors: ["tony-Ma-yami"]
reviewers: ["rootsongjc "，"stormgbs "]
---

# 故障注入

故障注入配置是用来模拟上游服务请求响应异常行为的一种手段。

在一个微服务架构的系统中，为了达到服务较高的健壮性要求，比如电商中的订单系统，当订单相关的服务出现异常时，一般需要第一时间通知维护人员，这对于服务的异常处理以及监听方面有很高的要求。从运维方面，我们可以对服务订单服务的相关主机或者云虚拟主机的 CPU、内存等重要指标进行监听，若发现异常，以短信、消息或者邮件的方式告知。另一方面，若程序内部访问异常，那么异常也要第一时间通知到维护人员与开发负责人。而这些异常在开发测试阶段人为干涉或者模拟是一个非常繁杂的工作。这这种坏境下， Istio 提供了一种无侵入式的故障注入机制，让开发测试人员在不用调整服务的前提下，通过代理配置即可完成对服务的异常模拟。大家可以通过配置不同的 HTTPFaultInjection 策略来检测当上游服务异常时当前服务的健壮性以及对于异常处理的能力。

**HTTPFaultInjection** 是 **[VirtualService](https://www.servicemesher.com/istio-handbook/concepts/virtualservice.html)** 中的一种规则，它包括了两种类型的故障注入：

- **abort**：非必配项，配置一个 Abort 类型的对象。用来注入请求异常类故障。简单的说，就是用来模拟上游服务对请求返回指定异常码时，当前的服务是否具备处理能力。
- **delay**：非必配项，配置一个 Delay 类型的对象。用来注入延时类故障。通俗一点讲，就是人为模拟上游服务的响应时间，测试在高延迟的情况下，当前的服务是否具备容错容灾的能力。

### HTTPFaultInjection.Abort：

- **httpStatus**：必配项，是一个整型的值。表示注入 HTTP 请求的故障状态码。
- **percentage**：非必配项，是一个 Percent 类型的值。表示对多少请求进行故障注入。如果不指定该配置，那么所有请求都将会被注入故障。
- **percent**：已经废弃的一个配置，与 percentage 配置功能一样，已经被 percentage 代替。

如下的配置表示对 `v1` 版本的 `ratings.prod.svc.cluster.local` 服务访问的时候进行故障注入，`0.1`表示有千分之一的请求被注入故障， `400` 表示故障为该请求的 HTTP 响应码为 `400` 。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings.prod.svc.cluster.local
  http:
  - route:
    - destination:
        host: ratings.prod.svc.cluster.local
        subset: v1
    fault:
      abort:
        percentage:
          value: 0.1
        httpStatus: 400
```

### HTTPFaultInjection.Delay：

- **fixedDelay**：必配项，表示请求响应的模拟处理时间。格式为：`1h/1m/1s/1ms`， 不能小于 `1ms`。
- **percentage**：非必配项，是一个 Percent 类型的值。表示对多少请求进行故障注入。如果不指定该配置，那么所有请求都将会被注入故障。
- **percent**：已经废弃的一个配置，与 `percentage` 配置功能一样，已经被 `percentage` 代替。

如下的配置表示对 `v1` 版本的 `reviews.prod.svc.cluster.local` 服务访问的时候进行延时故障注入，`0.1` 表示有千分之一的请求被注入故障，`5s` 表示访问`reviews.prod.svc.cluster.local` 时延时 `5s`。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews.prod.svc.cluster.local
  http:
  - match:
    - sourceLabels:
        env: prod
    route:
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v1
    fault:
      delay:
        percentage:
          value: 0.1
        fixedDelay: 5s
```



接下来，我们使用一个例子对比下没有配置故障注入、配置 Abort 类型故障、配置 Delay 类型故障这三种情况下请求的响应请况。我们的例子中是使用`Service A`服务向 `Service B`服务发送一个请求。

首先我们收集下未配置故障注入情况下请求的响应信息。如下图所示，`Service B`服务正常响应，响应码为`200`。响应头中`content-length`标头表示本次响应体的大小、`content-type`标头表示本次响应内容的数据格式与字符集、`server`标头表示本次响应来源，这里可以看到响应来自`Service B`服务的 istio-envoy 代理。`x-envoy-upstream-service-time`表示上游服务的处理时间。

![concepts-no-fault-inject](../images/concepts-no-fault-inject.png)

接下来我们 在`Service B`服务的 VirtualService 上配置 Abort 类型的故障注入。

![concepts-fault-inject-abort](../images/concepts-fault-inject-abort.png)

如上图所示，相比较正常的 Response 响应体，发现本次请求的响应体中返回的响应体为 “fault filter abort”，这说明配置生效了。返回的响应码为`400`，与我们的配置一致。而请求响应体中没有返回`x-envoy-upstream-service-time`参数。这说明当请求到达`Service B`的 Envoy 代理后直接被拦截并返回`400`，请求并没有被转发处理。

接下来，我们在`Service B`服务的 VirtualService 上配置 Delay 类型的故障注入。

![concepts-fault-inject-abort](../images/concepts-fault-inject-delay.png)

如上图所示，我们配置了将所有对`Service B`服务的请求全部延迟 `5s` 处理。相比较正常的 Response 响应体，发现本次请求正常响应，响应数据大小与未配置故障注入时一致。唯一的区别在于`x-envoy-upstream-service-time`值为`5`， 表示`Service B`服务处理了`5s`的时间才返回。这里需要注意的是并不是`Service B`服务本身处理需要这么长的时间，而是 Envoy 将请求拦截后等待了`5s`才将请求转发。

### 总结

通过以上对 Istio 故障注入的介绍、配置、及实现原理的介绍，不难看住故障注入一般适用在开发测试阶段，非常方便开发阶段对于功能及接口的测试。它依赖于 Envoy 的特性将故障注入与业务代码分离，使得业务代码更加的纯粹，故障注入测试时更加简洁方便，这个功能大大降低了模拟测试的复杂度。但需要注意的是，在上线前一定要对配置文件做检查校正，防止此类配置推送到产品坏境。

