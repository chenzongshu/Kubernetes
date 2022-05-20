# 启动

在Scheduler启动流程中，我们介绍了 `runCommand`->`Run`->`sched.Run`的过程

```go
func (sched *Scheduler) Run(ctx context.Context) {
	sched.SchedulingQueue.Run()
	wait.UntilWithContext(ctx, sched.scheduleOne, 0)  //scheduleOne就是调度主流程
	······
}
```

# ScheduleOne

scheduleOne()就是按照扩展点的顺序逐一执行，如果出错就报错就可以了

1. Scheduler从调度队列中弹出Pod，通过调度算法选出最优的Node；
2. 如果无Node能够满足Pod资源，则通过PostPlugin实现抢占；
3. 如果选出了最优Node，则经过`ReservePlugin`和`PermitPlugin`后Pod进入绑定周期；
4. Scheduler创建独立的协程执行绑定操作，自己的协程则开始下一个Pod的调度周期；
5. 调度Pod的流程中，如果失败则需要恢复预留的资源、删除假定调度、送回到调度队列等一系列操作，具体要看在哪个阶段失败的；
6. BindPlugin和Extender都有绑定能力，那么优先使用Extender绑定，但还有一个前提条件，就是Pod的有些资源是由Extender管理才行。否则，该Pod申请的资源与Extender没有任何关系，由Extender执行绑定没有任何意义，反而效率更低了；

```go
// scheduleOne()调度一个Pod。
func (sched *Scheduler) scheduleOne(ctx context.Context) {
	// 获取下一个需要调度Pod，可以理解为从调用ScheduleingQueuePop()，为什么要注入一个函数呢？下面会有注入函数的解析。
	podInfo := sched.NextPod()
	// 调度队列关闭的时候返回空的Pod，说明收到了关闭的信号，所以直接退出就行了，不用再判断ctx
	if podInfo == nil || podInfo.Pod == nil {
		return
	}
    // 根据Pod指定的调度器名字(Pod.Spec.SchedulerName)选择Framework。
	pod := podInfo.Pod
	fwk, err := sched.frameworkForPod(pod)
	if err != nil {
		klog.ErrorS(err, "Error occurred")
		return
	}
	// 是否需要忽略这个Pod，至于什么样的Pod会被忽略，后面有相关函数的注释。
	if sched.skipPodSchedule(fwk, pod) {
		return
	}

	klog.V(3).InfoS("Attempting to schedule pod", "pod", klog.KObj(pod))


	// 为调度Pod做准备，包括计时、创建CycleState以及调度上下文(schedulingCycleCtx)
	start := time.Now()
	state := framework.NewCycleState()
	state.SetRecordPluginMetrics(rand.Intn(100) < pluginMetricsSamplePercent)
	schedulingCycleCtx, cancel := context.WithCancel(ctx)
	defer cancel()
	// 通过调度算法找到最优的Node
	scheduleResult, err := sched.Algorithm.Schedule(schedulingCycleCtx, fwk, state, pod)
	if err != nil {
		// 调度算法可能因为任何Node都无法满足Pod资源需求返回失败，因此尝试进行抢占，并期望下一次尝试调度Pod时，由于抢占Node可以满足Pod的需求。
		// 也有一种可能存在，那就是另一个Pod被调度到被抢占的资源中，但这并没有什么损失，因为这说明这个Pod优先级更高。
		nominatedNode := ""
		// 判断调度算法返回的错误是否是因为资源无法满足造成的，如果是，那么就尝试抢占
		if fitError, ok := err.(*framework.FitError); ok {
			// 如果没有PostFilterPlugin(即抢占插件)，也就不用尝试抢占了
			if !fwk.HasPostFilterPlugins() {
				klog.V(3).InfoS("No PostFilter plugins are registered, so no preemption will be performed")
			} else {
				// 运行所有的PostFilterPlugin，尝试让Pod可在以未来调度周期中进行调度。
				// 为什么是未来的调度周期？道理很简单，需要等被强占Pod的退出。
				result, status := fwk.RunPostFilterPlugins(ctx, state, pod, fitError.Diagnosis.NodeToStatusMap)
				if status.Code() == framework.Error {
					klog.ErrorS(nil, "Status after running PostFilter plugins for pod", klog.KObj(pod), "status", status)
				} else {
					klog.V(5).InfoS("Status after running PostFilter plugins for pod", "pod", klog.KObj(pod), "status", status)
				}
				// 如果抢占成功，则记录提名Node的名字。
				if status.IsSuccess() && result != nil {
					nominatedNode = result.NominatedNodeName
				}
			}
			// metrics相关的代码不做注释
			metrics.PodUnschedulable(fwk.ProfileName(), metrics.SinceInSeconds(start))
		} else if err == core.ErrNoNodesAvailable {
			metrics.PodUnschedulable(fwk.ProfileName(), metrics.SinceInSeconds(start))
		} else {
			klog.ErrorS(err, "Error selecting node for pod", "pod", klog.KObj(pod))
			metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
		}
		// recordSchedulingFailure()用于统一实现调度Pod失败的处理，函数名字有record关键字，可以推测至少应该包含记录Pod调度失败的代码。
		// 也就是我们用kubectl describe pod xxx时，Events部分描述Pod因为什么原因不可调度的，所以参数有错误代码、不可调度原因等就很容易理解了。
		// 需要注意的是，即便抢占成功，Pod当前依然是不可调度状态，因为需要等待被强占的Pod退出，所以nominatedNode是否为空就可以判断是否抢占成功了。
		// 因为调度Pod失败的点非常多，后面有很多处都调用了recordSchedulingFailure()函数，笔者就不在重复注释了。
		// 下面有recordSchedulingFailure()函数注释，届时会揭开它神秘的面纱。
		sched.recordSchedulingFailure(fwk, podInfo, err, v1.PodReasonUnschedulable, nominatedNode)
		return
	}
	metrics.SchedulingAlgorithmLatency.Observe(metrics.SinceInSeconds(start))
	// 深度拷贝PodInfo赋值给假定调度Pod，为什么深度拷贝Pod？因为assume()会设置Pod.Status.NodeName = scheduleResult.SuggestedHost
	assumedPodInfo := podInfo.DeepCopy()
	assumedPod := assumedPodInfo.Pod
	// assume()会调用Cache.AssumePod()假定调度Pod，assume()函数下面有注释，此处暂时认为Cache.AssumePod()就行了。
	// 需要再解释一下为什么要假定调度Pod，因为Scheduler不用等到绑定结果就可以调度下一个Pod，如果无法理解建议阅读笔者关于调度缓存的文章。
	err = sched.assume(assumedPod, scheduleResult.SuggestedHost)
	if err != nil {
		metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
		// 什么情况会造成假定调度失败？根据Cache的源码可知Pod如果已经假定调度了，再次假定调度就会报错。
		// 那什么情况会重复假定调度Pod？根据Scheduler的事件处理函数源码可知，只要Pod未绑定，Add/Update事件都会将Pod放入调度队列。
		// 也就是在绑定周期事件内，如果Pod删除再添加亦或是更新，都有可能造成Pod重新调度并再次假定调度。
		// 还好，在注入的Error()函数中会检测Pod是否已经绑定，如果已经绑定则不会重新放入调度队列(否则，这将导致无限循环)。
		sched.recordSchedulingFailure(fwk, assumedPodInfo, err, SchedulerError, "")
		return
	}

	// 为Pod预留需要的全局资源，比如PV
	if sts := fwk.RunReservePluginsReserve(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost); !sts.IsSuccess() {
		metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
		// 即便预留资源失败了，还是调用一次恢复，可以清理一些状态，笔者认为理论上RunReservePluginsReserve()应该保证一定的事务性
		fwk.RunReservePluginsUnreserve(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
        // 因为Cache中已经假定Pod调度了，此处就应该删除假定调度的Pod
		if forgetErr := sched.SchedulerCache.ForgetPod(assumedPod); forgetErr != nil {
			klog.ErrorS(forgetErr, "scheduler cache ForgetPod failed")
		}
		// 见上面的注释
		sched.recordSchedulingFailure(fwk, assumedPodInfo, sts.AsError(), SchedulerError, "")
		return
	}

	// 判断Pod是否可以进入绑定阶段。
	runPermitStatus := fwk.RunPermitPlugins(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
	// 有插件没有批准Pod并且不是等待，那只能是拒绝或者发生了内部错误
	if runPermitStatus.Code() != framework.Wait && !runPermitStatus.IsSuccess() {
		// 获取失败的原因
		var reason string
		if runPermitStatus.IsUnschedulable() {
			metrics.PodUnschedulable(fwk.ProfileName(), metrics.SinceInSeconds(start))
			reason = v1.PodReasonUnschedulable
		} else {
			metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
			reason = SchedulerError
		}
		// 从此处开始，一旦调度失败，都会做如下几个事情 ：
		// 1. 恢复预留的资源：fwk.RunReservePluginsUnreserve()；
		// 2. 删除假定调度的Pod：sched.SchedulerCache.ForgetPod()；
		// 3. 汇报调度失败：sched.recordSchedulingFailure()；
		fwk.RunReservePluginsUnreserve(schedulingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		if forgetErr := sched.SchedulerCache.ForgetPod(assumedPod); forgetErr != nil {
			klog.ErrorS(forgetErr, "scheduler cache ForgetPod failed")
		}
		sched.recordSchedulingFailure(fwk, assumedPodInfo, runPermitStatus.AsError(), reason, "")
		return
	}

	// 进入绑定周期，创建一个协程异步绑定，因为绑定是一个相对比较耗时的操作，至少包含一次向apiserver写入绑定信息的操作。
	go func() {
		// 创建绑定周期的上下文，此时需要注意的是，现在已经处于另一个协程，Scheduler.scheduleOne已经开始调度下一个Pod了。
		// 从并行的角度讲，这属于时间并行(类似于流水线)。
		bindingCycleCtx, cancel := context.WithCancel(ctx)
		defer cancel()
		metrics.SchedulerGoroutines.WithLabelValues(metrics.Binding).Inc()
		defer metrics.SchedulerGoroutines.WithLabelValues(metrics.Binding).Dec()

		// 等待Pod批准通过，如果有PermitPlugin返回等待，Pod就会被放入waitingPodsMap直到所有的PermitPlug批准通过。
		// 好在调度框架帮我们实现了这些功能，此处只需要调用一个接口就全部搞定了。
		waitOnPermitStatus := fwk.WaitOnPermit(bindingCycleCtx, assumedPod)
		if !waitOnPermitStatus.IsSuccess() {
			// 获取失败的原因
			var reason string
			if waitOnPermitStatus.IsUnschedulable() {
				metrics.PodUnschedulable(fwk.ProfileName(), metrics.SinceInSeconds(start))
				reason = v1.PodReasonUnschedulable
			} else {
				metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
				reason = SchedulerError
			}
			// 见上面的注释
			fwk.RunReservePluginsUnreserve(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
			if forgetErr := sched.SchedulerCache.ForgetPod(assumedPod); forgetErr != nil {
				klog.ErrorS(forgetErr, "scheduler cache ForgetPod failed")
			}
			sched.recordSchedulingFailure(fwk, assumedPodInfo, waitOnPermitStatus.AsError(), reason, "")
			return
		}

		// 绑定预处理
		preBindStatus := fwk.RunPreBindPlugins(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		if !preBindStatus.IsSuccess() {
			metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
			// 见上面的注释
			fwk.RunReservePluginsUnreserve(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
			if forgetErr := sched.SchedulerCache.ForgetPod(assumedPod); forgetErr != nil {
				klog.ErrorS(forgetErr, "scheduler cache ForgetPod failed")
			}
			sched.recordSchedulingFailure(fwk, assumedPodInfo, preBindStatus.AsError(), SchedulerError, "")
			return
		}

		// 执行绑定操作，所谓绑定，就是向apiserver写入Pod的子资源/bind，里面包含有选择的Node。
		// 单独封装bind()函数用意是什么呢？下面有bind()函数的注释，到时候就明白了。
		err := sched.bind(bindingCycleCtx, fwk, assumedPod, scheduleResult.SuggestedHost, state)
		if err != nil {
			metrics.PodScheduleError(fwk.ProfileName(), metrics.SinceInSeconds(start))
			// 见上面的注释
			fwk.RunReservePluginsUnreserve(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
			if err := sched.SchedulerCache.ForgetPod(assumedPod); err != nil {
				klog.ErrorS(err, "scheduler cache ForgetPod failed")
			}
			sched.recordSchedulingFailure(fwk, assumedPodInfo, fmt.Errorf("binding rejected: %w", err), SchedulerError, "")
		} else {
			if klog.V(2).Enabled() {
				klog.InfoS("Successfully bound pod to node", "pod", klog.KObj(pod), "node", scheduleResult.SuggestedHost, "evaluatedNodes", scheduleResult.EvaluatedNodes, "feasibleNodes", scheduleResult.FeasibleNodes)
			}
			metrics.PodScheduled(fwk.ProfileName(), metrics.SinceInSeconds(start))
			metrics.PodSchedulingAttempts.Observe(float64(podInfo.Attempts))
			metrics.PodSchedulingDuration.WithLabelValues(getAttemptsLabel(podInfo)).Observe(metrics.SinceInSeconds(podInfo.InitialAttemptTimestamp))

			// 绑定后处理
			fwk.RunPostBindPlugins(bindingCycleCtx, state, assumedPod, scheduleResult.SuggestedHost)
		}
	}()
}
```

