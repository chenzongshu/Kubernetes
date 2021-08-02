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

```go
func (a *HorizontalController) computeStatusForPodsMetric(currentReplicas int32, metricSpec autoscalingv2.MetricSpec, hpa *autoscalingv2.HorizontalPodAutoscaler, selector labels.Selector, status *autoscalingv2.MetricStatus, metricSelector labels.Selector) (replicaCountProposal int32, timestampProposal time.Time, metricNameProposal string, condition autoscalingv2.HorizontalPodAutoscalerCondition, err error) {
	//计算需要扩缩容的数量
	replicaCountProposal, utilizationProposal, timestampProposal, err := a.replicaCalc.GetMetricReplicas(currentReplicas, metricSpec.Pods.Target.AverageValue.MilliValue(), metricSpec.Pods.Metric.Name, hpa.Namespace, selector, metricSelector)
	if err != nil {
		condition = a.getUnableComputeReplicaCountCondition(hpa, "FailedGetPodsMetric", err)
		return 0, timestampProposal, "", condition, err
	}
	...
	return replicaCountProposal, timestampProposal, fmt.Sprintf("pods metric %s", metricSpec.Pods.Metric.Name), autoscalingv2.HorizontalPodAutoscalerCondition{}, nil
}

// pkg/controller/podautoscaler/replica_calculator.go
func (c *ReplicaCalculator) GetMetricReplicas(currentReplicas int32, targetUtilization int64, metricName string, namespace string, selector labels.Selector, metricSelector labels.Selector) (replicaCount int32, utilization int64, timestamp time.Time, err error) {
	//获取pod对应的度量数据
	metrics, timestamp, err := c.metricsClient.GetRawMetric(metricName, namespace, selector, metricSelector)
	if err != nil {
		return 0, 0, time.Time{}, fmt.Errorf("unable to get metric %s: %v", metricName, err)
	}
	//通过结合度量数据来计算希望扩缩容的数量是多少
	replicaCount, utilization, err = c.calcPlainMetricReplicas(metrics, currentReplicas, targetUtilization, namespace, selector, v1.ResourceName(""))
	return replicaCount, utilization, timestamp, err
}
```

这里会调用`GetRawMetric`方法来获取pod对应的度量数据，然后再调用`calcPlainMetricReplicas`方法结合度量数据与目标期望来计算希望扩缩容的数量是多少。

### calcPlainMetricReplicas

calcPlainMetricReplicas方法逻辑比较多

```go
func (c *ReplicaCalculator) calcPlainMetricReplicas(metrics metricsclient.PodMetricsInfo, currentReplicas int32, targetUtilization int64, namespace string, selector labels.Selector, resource v1.ResourceName) (replicaCount int32, utilization int64, err error) {

	podList, err := c.podLister.Pods(namespace).List(selector)
	...
  /******* part1 先计算使用率   *********/
	//将pod分成三类进行统计，得到ready的pod数量、unready、ignored、missing Pod集合
	readyPodCount, unreadyPods, missingPods, ignoredPods := groupPods(podList, metrics, resource, c.cpuInitializationPeriod, c.delayOfInitialReadinessStatus)
	//在度量的数据里移除ignored、unready Pods集合的数据
	removeMetricsForPods(metrics, ignoredPods)
	removeMetricsForPods(metrics, unreadyPods)
	//计算pod中container request 设置的资源之和
	requests, err := calculatePodRequests(podList, resource)
  //metric取了2值：使用率和利用率，其中使用率是 utilization/targetUtilization
	usageRatio, utilization := metricsclient.GetMetricUtilizationRatio(metrics, targetUtilization)  
	
  /******* part2 把特殊状态的Pod单独处理使用率，再计算使用率   *********/
  if len(missingPods) > 0 {
        ...
				metrics[podName] = metricsclient.PodMetric{Value: targetUtilization}
			}
		} else {
        ...
				metrics[podName] = metricsclient.PodMetric{Value: 0}

	if rebalanceIgnored {
		  ...
			metrics[podName] = metricsclient.PodMetric{Value: 0}
  // 再计算使用率
  newUsageRatio, _ := metricsclient.GetMetricUtilizationRatio(metrics, targetUtilization)
  ...
  newReplicas := int32(math.Ceil(newUsageRatio * float64(len(metrics))))
}
```

- ceil函数是向上取整
- 最后的算法基本和[官网文档](https://kubernetes.io/zh/docs/tasks/run-application/horizontal-pod-autoscale/)一致，`Ceil(newUsageRatio * float64(len(metrics)`，本身`newUsageRatio`就是 `utilization/targetUtilization` 算出来的

## normalizeDesiredReplicasWithBehaviors

下面代码里面包含了behavior的几个设置

- 稳定窗口：stabilizationWindowSeconds，为了防止指标抖动造成频繁扩缩容，所以有这么一个概念，默认是300秒即5分钟。会去统计5分钟的所有数据来确定是否扩缩容。默认情况只有缩容才有稳定窗口
- type有Pod和Percent，分别是按照Pod数和百分比扩容

```go
func (a *HorizontalController) normalizeDesiredReplicasWithBehaviors(hpa *autoscalingv2.HorizontalPodAutoscaler, key string, currentReplicas, prenormalizedDesiredReplicas, minReplicas int32) int32 {
	//如果StabilizationWindowSeconds设置为空，那么给一个默认的值,默认300s
	a.maybeInitScaleDownStabilizationWindow(hpa)
	normalizationArg := NormalizationArg{
		Key:               key,
		ScaleUpBehavior:   hpa.Spec.Behavior.ScaleUp,
		ScaleDownBehavior: hpa.Spec.Behavior.ScaleDown,
		MinReplicas:       minReplicas,
		MaxReplicas:       hpa.Spec.MaxReplicas,
		CurrentReplicas:   currentReplicas,
		DesiredReplicas:   prenormalizedDesiredReplicas}
	//根据参数获取建议副本数
	stabilizedRecommendation, reason, message := a.stabilizeRecommendationWithBehaviors(normalizationArg)
	normalizationArg.DesiredReplicas = stabilizedRecommendation
	... 
	//根据scaleDown或scaleUp指定的参数做限制
	desiredReplicas, reason, message := a.convertDesiredReplicasWithBehaviorRate(normalizationArg)
	... 
	return desiredReplicas
}
```

这个方法主要分为两部分

- 一部分是调用stabilizeRecommendationWithBehaviors方法来根据时间窗口来获取一个建议副本数；
- 另一部分convertDesiredReplicasWithBehaviorRate方法是根据scaleDown或scaleUp指定的参数做限制。

### stabilizeRecommendationWithBehaviors

```go
func (a *HorizontalController) stabilizeRecommendationWithBehaviors(args NormalizationArg) (int32, string, string) {
  ...

	// 如果期望的副本数大于等于当前的副本数,则延迟时间=scaleUpBehaviro的稳定窗口时间
	if args.DesiredReplicas >= args.CurrentReplicas {
		scaleDelaySeconds = *args.ScaleUpBehavior.StabilizationWindowSeconds
		betterRecommendation = min
	} else {
		// 期望副本数<当前的副本数
		scaleDelaySeconds = *args.ScaleDownBehavior.StabilizationWindowSeconds
		betterRecommendation = max
	}
	//获取一个最大的时间窗口
	maxDelaySeconds := max(*args.ScaleUpBehavior.StabilizationWindowSeconds, *args.ScaleDownBehavior.StabilizationWindowSeconds)
	obsoleteCutoff := time.Now().Add(-time.Second * time.Duration(maxDelaySeconds))

	cutoff := time.Now().Add(-time.Second * time.Duration(scaleDelaySeconds))
	for i, rec := range a.recommendations[args.Key] {
		if rec.timestamp.After(cutoff) {
			// 在截止时间之后，则当前建议有效, 则根据之前的比较函数来决策最终的建议副本数
			recommendation = betterRecommendation(rec.recommendation, recommendation)
		}
		//如果被遍历到的建议时间是在obsoleteCutoff之前，那么需要重新设置建议
		if rec.timestamp.Before(obsoleteCutoff) {
			foundOldSample = true
			oldSampleIndex = i
		}
	}
	//如果被遍历到的建议时间是在obsoleteCutoff之前，那么需要重新设置建议
	if foundOldSample {
		a.recommendations[args.Key][oldSampleIndex] = timestampedRecommendation{args.DesiredReplicas, time.Now()}
	} else {
		a.recommendations[args.Key] = append(a.recommendations[args.Key], timestampedRecommendation{args.DesiredReplicas, time.Now()})
	}
	return recommendation, reason, message
}
```

# 总结

```
┌─────────────────────────┐                                                                                                                            
│           Run           │                                                                                                                            
└─────────────────────────┘                                                                                                                            
             │                                                    ┌────────────────────────┐                                                           
             │                                                    │computeStatusForObjectMe│                                                           
             ▼                                               ┌───▶│          tric          │                                                           
┌─────────────────────────┐                                  │    └────────────────────────┘                                                           
│   reconcileAutoscaler   │                                  │                                                                                         
└─────────────────────────┘                                  │    ┌────────────────────────┐                                                           
             │                                               │    │computeStatusForPodsMetr│   ┌────────────────────────┐                              
             │                                               ├───▶│           ic           │──▶│   GetMetricReplicas    │                              
             │                                               │    └────────────────────────┘   └────────────────────────┘                              
             │                                               │                                              │                                          
             ▼                                               │    ┌────────────────────────┐                ▼                                          
┌─────────────────────────┐    ┌────────────────────────┐    │    │computeStatusForResource│   ┌────────────────────────┐                              
│computeReplicasForMetrics│───▶│computeReplicasForMetric│────┼───▶│         Metric         │   │computeStatusForPodsMetr│    ┌────────────────────────┐
└─────────────────────────┘    └────────────────────────┘    │    └────────────────────────┘   │           ic           │───▶│      GetRawMetric      │
             │                                               │                                 └────────────────────────┘    └────────────────────────┘
             │                                               │    ┌────────────────────────┐                                              │            
             │                                               │    │computeStatusForContaine│                                              ▼            
 hpa.Spec.Behavior != nil                                    ├───▶│    rResourceMetric     │                                 ┌────────────────────────┐
             │                                               │    └────────────────────────┘                                 │calcPlainMetricReplicas │
             │                                               │                                                               │                        │
             ▼                                               │    ┌────────────────────────┐                                 └────────────────────────┘
┌─────────────────────────┐                                  │    │computeStatusForExternal│                                                           
│normalizeDesiredReplicasW│                                  └───▶│         Metric         │                                                           
│      ithBehaviors       │──────────┐                            └────────────────────────┘                                                           
└─────────────────────────┘          │                                                                                                                 
             │                       │          ┌─────────────────────────┐                                                                            
             │                       │          │stabilizeRecommendationWi│                                                                            
             │                       ├─────────▶│       thBehaviors       │                                                                            
        if rescale                   │          └─────────────────────────┘                                                                            
             │                       │                                                                                                                 
             │                       │                                                                                                                 
             ▼                       │          ┌─────────────────────────┐                                                                            
┌─────────────────────────┐          │          │convertDesiredReplicasWit│                                                                            
│a.scaleNamespacer.Scales(│          └─────────▶│      hBehaviorRate      │                                                                            
│       ).Update()        │                     └─────────────────────────┘                                                                            
└─────────────────────────┘                                                                                                                            
```

