# HPA

HPA全称是 Horizontal Pod Autoscaler，是对k8s的workload的副本数进行自动水平扩缩容（scale）机制，也是k8s里使用需求最广泛的一种Autoscaler机制

目前 HPA 已经支持了 多个版本，最新是`autoscaling/v2beta2`。大部分的开发者目前比较熟悉的是`autoscaling/v1` 的版本，这个版本的特点是只支持 CPU 一个指标的弹性伸缩

> v1版本就有autoscaling/v1、autoscaling/v1beta1 和 autoscaling/v1beta2三个版本，模板指标不同

`autoscaling/v2beta1` 支持了 `Resource Metrics` 和 `Custom Metrics`。而在 `autoscaling/v2beta2` 的版本中额外增加了 `External Metrics` 的支持

**在`autoscaling/v1`里只有targetCPUUtilizationPercentage，`autoscaling/v2beta1`开始就扩展为metrics数组了**

> HPA必须要设置request的值才会工作

HPA的实现是一个控制循环，由controller manager的–horizontal-pod-autoscaler-sync-period参数指定周期（默认值为15秒）。每个周期内，controller manager根据每个HorizontalPodAutoscaler定义中指定的指标查询资源利用率。controller manager可以从resource metrics API（pod 资源指标）和custom metrics API（自定义指标）获取指标。

## metrics分类

v2版本有五种metric： Resource、Pods、Object、ContainerResource、External，对应不同场景

- `Resource`：支持k8s里Pod的所有系统资源（包括cpu、memory等），但是一般只会用cpu，memory因为不太敏感而且跟语言相关：大多数语言都有内存池及内置GC机制导致进程内存监控不准确。
- `Pods`：表示cpu，memory等系统资源之外且是由Pod自身提供的自定义metrics数据，比如用户可以在web服务的pod里提供一个promesheus metrics的自定义接口，里面暴露了本pod的实时QPS监控指标，这种情况下就应该在HPA里直接使用Pods类型的metrics。
- `Object`：表示监控指标不是由Pod本身的服务提供，但是可以通过k8s的其他资源Object提供metrics查询，比如ingress等，一般Object是需要汇聚关联的Deployment下的所有的pods总的指标。
- `External`：也属于自定义指标，与Pods和Object不同的是，其监控指标的来源跟k8s本身无关，metrics的数据完全取自外部的系统。
- `ContainerResource`: 对于带有多个Sidecar容器的Pod，可以使用`ContainerResource`来计算所有Pod中某一个容器的平均值来扩容

## 算法

### 扩缩容threshold控制

为了避免数值抖动造成频繁扩缩容，kube-controller-manager里有个HPA的专属参数 horizontal-pod-autoscaler-tolerance 表示HPA可容忍的最小副本数变化比例，默认是0.1，如果期望变化的副本倍数在[0.9, 1.1] 之间就直接停止计算返回

HPA控制器为了避免副本倍增过快还加了个约束：**单次倍增的倍数不能超过2倍，而如果原副本数小于2，则可以一次性扩容到4副本，注意这里的速率是代码写死不可用配置的**。(这个也是HPA controller默认的扩缩容速率控制，autoscaling/v2beta2的HPA Behavior属性可以覆盖这里的全局默认速率)

### 缩容冷却机制

虽然HPA同时支持扩容和缩容，但是在生产环境上扩容一般来说重要性更高，特别是流量突增的时候，能否快速扩容决定了系统的稳定性，所以HPA的算法里对扩容的时机是没有额外限制的，只要达到扩容条件就会在reconcile里执行扩容（当前一次至少扩容到原来的1.1倍）。但是为了避免过早缩导致来回波动（thrashing ），而容影响服务稳定性甚，HPA的算法对缩容的要求比较严格，通过设置一个默认5min（可配置horizontal-pod-autoscaler-downscale-stabilization）的滑动窗口，来记录过去5分钟的期望副本数，只有连续5分钟计算出的期望副本数都比当前副本数小，才执行scale缩容操作，缩容的目标副本数取5分钟窗口的最大值。

### 扩缩容速度

为了能更精准灵活地控制HPA的autoscale速度，从k8s v1.18（依赖HPA autoscaling/v2beta2）开始HPA的spec里新增了behavior结构，可以控制扩缩容行为

# v1

因为很多语言里面，**应用的内存都要交给语言层面的 VM 进行管理，也就是说，内存的回收是由 VM 的 GC 来决定的**，所以v1版本只关注CPU资源

假设有个deployment 包含3个 `Pod`，每个副本的 Request 值是 1 核，当前 3 个 `Pod` 的 CPU 利用率分别是 60%、70% 与 80%，此时我们设置 HPA 阈值为 50%，最小副本为 1，最大副本为 10

HPA模板为

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: default
spec:
  scaleTargetRef:   #表示要伸缩的对象是谁
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50  #整体的资源利用率阈值
```

- 总的 `Pod` 的利用率是 60%+70%+80% = 210%；
- 当前的 `Target` 是 3；
- 算式的结果是 70%，大于50%阈值，因此当前的 `Target` 数目过小，需要进行扩容；
- 重新设置 `Target` 值为 5，此时算式的结果为 42% 低于 50%，判断还需要扩容容器；
- 此时 HPA 设置 `Replicas` 为 5，进行 `Pod` 的水平扩容。

# v2

按照下面部署HPA

```yaml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: AverageUtilization
        averageUtilization: 50
  - type: Pods
    pods:
      metric:
        name: packets-per-second
      target:
        type: AverageValue
        averageValue: 1k
  - type: Object
    object:
      metric:
        name: requests-per-second
      describedObject:
        apiVersion: networking.k8s.io/v1beta1
        kind: Ingress
        name: main-route
      target:
        kind: Value
        value: 10k
```

你的 HorizontalPodAutoscaler 将会尝试确保每个 Pod 的 CPU 利用率在 50% 以内， 每秒能够服务 1000 个数据包请求， 并确保所有在 Ingress 后的 Pod 每秒能够服务的请求总数达到 10000 个。