> 代码基于1.21

# 初始化

HPA Controller和其他的Controller一样，都在NewControllerInitializers方法中进行注册，然后通过startHPAController来启动。

```go
// cmd/kube-controller-manager/app/controllermanager.go
func NewControllerInitializers(loopMode ControllerLoopMode) map[string]InitFunc {
	...
	controllers["horizontalpodautoscaling"] = startHPAController
	...
}
```

下面经过一些调用，最后走到Run函数

```go
//  cmd/kube-controller-manager/app/autoscaling.go
func startHPAController(ctx ControllerContext) (http.Handler, bool, error) {
	...
	return startHPAControllerWithLegacyClient(ctx)
}

func startHPAControllerWithLegacyClient(ctx ControllerContext) (http.Handler, bool, error) {
  ...
	return startHPAControllerWithMetricsClient(ctx, metricsClient)
}

func startHPAControllerWithMetricsClient(ctx ControllerContext, metricsClient metrics.MetricsClient) (http.Handler, bool, error) {
	...
	go podautoscaler.NewHorizontalController(
  ...
  ).Run(ctx.Stop)
	return nil, true, nil
}
```

既然 Pod 副本数量的计算是基于 Pod 的负载情况，那边需要途径获取负载数据，这个途径就是`MetricsClient`。

`MetricsClient`有两种实现：REST 方式和传统（Legacy）方式，分别是`restMetricsClient`和`HeapsterMetricsClient`。一个是REST 实现以支持自定义的指标；一个是传统的 Heapster 指标（heapster 已经从 1.13 版本开始被废弃了）。

## Run()

```go
//  pkg/controller/podautoscaler/horizontal.go
func (a *HorizontalController) Run(stopCh <-chan struct{}) {
  ...
  // 等待 informer 完成HorizontalPodAutoscaler相关事件的同步
	if !cache.WaitForNamedCacheSync("HPA", stopCh, a.hpaListerSynced, a.podListerSynced) {
		return
	}

	// 启动异步线程，每秒执行一次
	go wait.Until(a.worker, time.Second, stopCh)

	<-stopCh
}
```

这里会调用worker执行具体的扩缩容的逻辑

```go
func (a *HorizontalController) worker() {
	for a.processNextWorkItem() {
	}
}
```

`worker`的核心是从工作队列中获取一个 key（格式为：namespace/name），然后对 key 进行 reconcile

```go
func (a *HorizontalController) processNextWorkItem() bool {
	key, quit := a.queue.Get()
	...
	deleted, err := a.reconcileKey(key.(string))
	...
}
```

经过调用之后进入reconcileAutoscaler方法，这个是HPA的核心

# 核心代码

## reconcileAutoscaler

下面会调用 `computeReplicasForMetrics` 计算，然后 `ScaleInterface.Update`更新

简单来说就是先从`Informer`中拿到 key 对应的`HorizontalPodAutoscaler`资源实例；然后通过`HorizontalPodAutoscaler`实例中的信息，检查目标资源的Pod 负载以及当前的副本数，得到期望的 Pod 副本数；最终通过 Scale API 来调整 Pod 的副本数。最后会将调整的原因、计算的结果等信息写入 `HorizontalPodAutoscaler` 实例的 condition 中。

```go
func (a *HorizontalController) reconcileAutoscaler(hpav1Shared *autoscalingv1.HorizontalPodAutoscaler, key string) error {
	// 转换为autoscaling/v2,计算更容易 
	hpaRaw, err := unsafeConvertToVersionVia(hpav1, autoscalingv2.SchemeGroupVersion)

  // 副本数为0，不启动自动扩缩容
	if scale.Spec.Replicas == 0 && minReplicas != 0 {
    ...
    
  // 如果当前副本数大于最大期望副本数，那么设置期望副本数为最大副本数
  } else if currentReplicas > hpa.Spec.MaxReplicas {
		desiredReplicas = hpa.Spec.MaxReplicas
  
  // 如果当前副本数小于最小期望副本数，那么设置期望副本数为最小副本数
	} else if currentReplicas < minReplicas {
		desiredReplicas = minReplicas
	} else {
    // 计算需要扩缩容的数量，computeReplicasForMetrics是关键函数！
		metricDesiredReplicas, metricName, metricStatuses, metricTimestamp, err = a.computeReplicasForMetrics(hpa, scale, hpa.Spec.Metrics)
		...
		if metricDesiredReplicas > desiredReplicas {
			desiredReplicas = metricDesiredReplicas
			rescaleMetric = metricName
		}
		if desiredReplicas > currentReplicas {
			rescaleReason = fmt.Sprintf("%s above target", rescaleMetric)
		}
		if desiredReplicas < currentReplicas {
			rescaleReason = "All metrics below target"
		}
    // 如果设置了Behavior则调用normalizeDesiredReplicasWithBehaviors函数来修正最后的结果
		if hpa.Spec.Behavior == nil {
			desiredReplicas = a.normalizeDesiredReplicas(hpa, key, currentReplicas, desiredReplicas, minReplicas)
		} else {
			desiredReplicas = a.normalizeDesiredReplicasWithBehaviors(hpa, key, currentReplicas, desiredReplicas, minReplicas)
		}
    // rescale变量判断是否扩容
		rescale = desiredReplicas != currentReplicas
  } 
  if rescale {
    _, err = a.scaleNamespacer.Scales(hpa.Namespace).Update(context.TODO(), targetGR, scale, metav1.UpdateOptions{})
    ...
  }
}
```

## computeReplicasForMetrics

computeReplicasForMetrics 计算 HPA 中列出的指标规范所需的副本数，返回计算的副本计数的最大值、相关指标的描述以及所有计算指标的状态。

```go
func (a *HorizontalController) computeReplicasForMetrics(hpa *autoscalingv2.HorizontalPodAutoscaler, scale *autoscalingv1.Scale,
	metricSpecs []autoscalingv2.MetricSpec) (replicas int32, metric string, statuses []autoscalingv2.MetricStatus, timestamp time.Time, err error) {
  ...
  // 遍历度量目标的数组，这个数组就是yaml模板里面的多个type组成的数组
	for i, metricSpec := range metricSpecs {
		replicaCountProposal, metricNameProposal, timestampProposal, condition, err := a.computeReplicasForMetric(hpa, metricSpec, specReplicas, statusReplicas, selector, &statuses[i])
    ...
    // 记录最大的需要扩缩容的数量
		if err == nil && (replicas == 0 || replicaCountProposal > replicas) {
			timestamp = timestampProposal
			replicas = replicaCountProposal
			metric = metricNameProposal
		}
	}
  ...
	return replicas, metric, statuses, timestamp, nil
}
```

什么是metric数组？从k8s官方给的hpa文档可以看出

```yaml
apiVersion: autoscaling/v2beta2
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
        type: Utilization
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
        type: Value
        value: 10k
```

例如这个官方的例子中，设置了三个metric，所以我们在上面的代码中遍历所有的metrics，然后选取返回副本数最大的那个

## computeReplicasForMetric

根据类型来分类处理

```go
func (a *HorizontalController) computeReplicasForMetric(hpa *autoscalingv2.HorizontalPodAutoscaler, spec autoscalingv2.MetricSpec,
	specReplicas, statusReplicas int32, selector labels.Selector, status *autoscalingv2.MetricStatus) (...) {
	switch spec.Type {
	case autoscalingv2.ObjectMetricSourceType:
		...
		replicaCountProposal, timestampProposal, metricNameProposal, condition, err = a.computeStatusForObjectMetric(...)
	case autoscalingv2.PodsMetricSourceType:
		...
		replicaCountProposal, timestampProposal, metricNameProposal, condition, err = a.computeStatusForPodsMetric(...)
	case autoscalingv2.ResourceMetricSourceType:
		replicaCountProposal, timestampProposal, metricNameProposal, condition, err = a.computeStatusForResourceMetric(...)
	case autoscalingv2.ContainerResourceMetricSourceType:
		replicaCountProposal, timestampProposal, metricNameProposal, condition, err = a.computeStatusForContainerResourceMetric(...)
	case autoscalingv2.ExternalMetricSourceType:
		replicaCountProposal, timestampProposal, metricNameProposal, condition, err = a.computeStatusForExternalMetric(...)
  ...
	return replicaCountProposal, metricNameProposal, timestampProposal, autoscalingv2.HorizontalPodAutoscalerCondition{}, nil
}
```

上面度量类型不做详细解释，可以看另外一篇文章《弹性伸缩HPA简介》，下面以Pod类型来看看代码

