---
authors: ["hb-chen"]
reviewers: [""]
---

# 拒绝授权

本节主要介绍拒绝授权的使用，包括**默认全部拒绝**和**全部拒绝**两个授权策略，通过配置分析理解授权策略的匹配流程。

## 默认全部拒绝

前文实践有用到的默认全部拒绝的配置如下：

```yaml
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1 
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  {}
EOF
```

刚接触授权策略可能多少有些不解，首先看下官方文档有关 [Implicit Enablement](https://istio.io/docs/concepts/security/#implicit-enablement) 的说明，从中提取两个重点信息:
1. For workloads without authorization policies applied, Istio doesn’t enforce access control allowing all requests.
    - 对于**没有使用授权策略**的工作负载，Istio 不会执行访问控制，**允许全部请求**。
1. If any allow policies are applied to a workload, access to that workload is denied by default, unless explicitly allowed by the rule in the policy.
    - 如果将任何**允许策略**应用到工作负载，则**默认**情况下将**拒绝**对该工作负载的访问，除非符合`ALLOW`策略中的规则。

其次参考 [Authorization Policy Reference](https://istio.io/docs/reference/config/security/authorization-policy/) 有关匹配规则的说明:
1. If there are any DENY policies that match the request, deny the request.
1. If there are no ALLOW policies for the workload, allow the request.
1. If any of the ALLOW policies match the request, allow the request.
1. Deny the request.

结合这两处参考再来解读这个配置，允许全部请求是默认的策略，即匹配规则的第2条：没有`ALLOW`策略则全部允许。默认拒绝全部所添加的`spec:{}`策略，其实是一个`action: ALLOW`而`rules`为`nil`的策略，这样就跳过了匹配规则的第2条，`rules`为`nil`是没有请求能够匹配的，所以使默认策略变为了全部拒绝。

理解拒绝授权策略另一个关键点是`rules`为`nil`和`- {}`的差别，`nil`是空的规则，没有请求能够匹配；`- {}`可以理解为没有任何`from`、`to`和`when`条件的`rule`，而`rule`的三个条件在为空时默认都是允许，所以`rules: - {}`可以匹配所有请求。这样看`spec: {}`实际是一个用于跳过`no ALLOW`条件的空配置，如果在**同一个策略目标**下有其它`ALLOW`规则可以省略。不过为了避免遗漏，对于访问控制有严格要求的场景，还是建议使用`spec: {}`将指定`namespace`或整个网格的授权策略改为默认全部拒绝，因为还要考虑策略目标的因素，一般`ALLOW`策略会有明确的目标，如果某一 workload 没有使用任何`ALLOW`策略，那么就会被遗漏。

下面的配置与`spec: {}`是等价的

```yaml
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1 
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: defaul
spec:
  action: ALLOW
EOF
```

而下面的配置是全部允许

```yaml
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1 
kind: AuthorizationPolicy
metadata:
  name: allow-all
  namespace: default
spec:
  action: ALLOW
  rules:
  - {}
EOF
```

## 全部拒绝

前面了解到**默认全部拒绝**是一个空的`ALLOW`配置，可以理解为站位，在此基础上再补充其它`ALLOW`策略。那如果一个服务就是要拒绝全部请求呢？和**全部允许**类似，将`action`改为`DENY`，就没有请求能够到达这个服务，这对于一些没有入口流量的服务也是有应用场景的。

```yaml
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1 
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  action: DENY
  rules:
  - {}
EOF
```

有关拒绝授权在理解了**默认全部拒绝**和**全部拒绝**后，其它有关`action: DENY`的使用，主要是不同`rule`规则的配置，可以参考前面章节中`HTTP`、`TCP`、`JWT`等授权策略的条件配置，这里不再赘述。

## 小结

拒绝授权的使用重点是理解`AuthorizationPolicy`的四个步骤:
1. 拒绝条件是否匹配，匹配则拒绝；
1. 是否有`ALLOW`条件，没有全部允许；
1. `ALLOW`条件是否匹配，匹配则允许；
1. 拒绝。

另外就是对`rules`为`nil`和`- {}`的理解，有了这些再配合不同的策略范围和规则可以灵活的配置授权策略。

## 参考

- [Istio Concepts / Security#Implicit enablement](https://istio.io/docs/concepts/security/#implicit-enablement)
- [Istio Reference / Authorization Policy](https://istio.io/docs/reference/config/security/authorization-policy/)
