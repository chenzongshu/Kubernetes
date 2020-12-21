> 本文基于kubernetes 1.19版本

kubelet的源码位于 `cmd/kubelet` 和 `pkg/kubelet`

# Main函数

`kubelet`采用了`cobra`命令行框架， 入口函数位于 `cmd/kubelet/kubelet.go`

```
func main() {
	rand.Seed(time.Now().UnixNano())

	command := app.NewKubeletCommand()
	logs.InitLogs()
	defer logs.FlushLogs()

	if err := command.Execute(); err != nil {
		os.Exit(1)
	}
}
```

其中`NewKubeletCommand`就是一个corba对象，具体定义如下

```go
func NewKubeletCommand() *cobra.Command {
	cleanFlagSet := pflag.NewFlagSet(componentKubelet, pflag.ContinueOnError)
	cleanFlagSet.SetNormalizeFunc(cliflag.WordSepNormalizeFunc)
	// kubeletFlags，kubeletConfig就是kubelet两个重要的参数对象
	kubeletFlags := options.NewKubeletFlags()
	kubeletConfig, err := options.NewKubeletConfiguration()
	······ //读取各种配置
			Run: func(cmd *cobra.Command, args []string) {
			······
			kubeletServer := &options.KubeletServer{
				KubeletFlags:         *kubeletFlags,
				KubeletConfiguration: *kubeletConfig,
			}
			kubeletDeps, err := UnsecuredDependencies(kubeletServer, utilfeature.DefaultFeatureGate)
        
			······
			if err := Run(ctx, kubeletServer, kubeletDeps, utilfeature.DefaultFeatureGate); 
```

最核心的就是Run函数，从名字也能看出来就是kubelet启动函数，入参是一个context，上面生成的两个配置，然后通过 `Run()` -> `run()` -> `RunKubelet()` 

## RunKubelet

 `RunKubelet()` 里面的核心函数就是`createAndInitKubelet`和`startKubelet`

### createAndInitKubelet

`createAndInitKubelet`创建了一个kubelet实例

```go
func createAndInitKubelet(...){
    ...
    k, err = kubelet.NewMainKubelet(...)
    if err != nil {
        return nil, err
    }
    k.BirthCry()  // 通知api-server 服务 kubelet 启动

    k.StartGarbageCollection() // 开启垃圾回收服务
}
```

`NewMainKubelet` 实例化一个 kubelet 对象，并对 kubelet 内部各个 component 进行初始化工作，包括

containerGC（容器的垃圾回收），statusManager（pod 状态的管理），imageManager （镜像的管路），probeManager （容器健康检测），gpuManager （GPU 的支持），PodCache（Pod 缓存的管理），secretManager （secret 资源的管理），configMapManager （configMap 资源的管理），InitNetworkPlugin     （网络插件的初始化），PodManager（对 pod 的管理, e.g., CRUD），makePodSourceConfig（pod 元数据的来源 [FILE, URL, api-server]），ContainerRuntime（容器运行时的选择[docker 或 rkt]）

### startKubelet

```
func startKubelet(k kubelet.Bootstrap, podCfg *config.PodConfig, kubeCfg *kubeletconfiginternal.KubeletConfiguration, kubeDeps *kubelet.Dependencies, enableCAdvisorJSONEndpoints, enableServer bool) {
	// start the kubelet
	go k.Run(podCfg.Updates())

	// start the kubelet server
	if enableServer {
```

`kubelet.Bootstrap`是一个interface， 对应实现在 `pkg/kubelet/kubelet.go`

# Run主流程

上面的interface的实现在`pkg/kubelet/kubelet.go` 

```go
func (kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate) {
	if kl.logServer == nil {
		kl.logServer = http.StripPrefix("/logs/", http.FileServer(http.Dir("/var/log/")))
	}
	if kl.kubeClient == nil {
		klog.Warning("No api server defined - no node status update will be sent.")
	}

	if kl.cloudResourceSyncManager != nil {
		go kl.cloudResourceSyncManager.Run(wait.NeverStop)
	}

	if err := kl.initializeModules(); err != nil {
		kl.recorder.Eventf(kl.nodeRef, v1.EventTypeWarning, events.KubeletSetupFailed, err.Error())
		klog.Fatal(err)
	}

	go kl.volumeManager.Run(kl.sourcesReady, wait.NeverStop)

	if kl.kubeClient != nil {
		go wait.Until(kl.syncNodeStatus, kl.nodeStatusUpdateFrequency, wait.NeverStop)
		go kl.fastStatusUpdateOnce()

		go kl.nodeLeaseController.Run(wait.NeverStop)
	}
	go wait.Until(kl.updateRuntimeUp, 5*time.Second, wait.NeverStop)

	if kl.makeIPTablesUtilChains {
		kl.initNetworkUtil()
	}

	go wait.Until(kl.podKiller.PerformPodKillingWork, 1*time.Second, wait.NeverStop)

	kl.statusManager.Start()
	kl.probeManager.Start()

	if kl.runtimeClassManager != nil {
		kl.runtimeClassManager.Start(wait.NeverStop)
	}

	kl.pleg.Start()
	kl.syncLoop(updates, kl)
}
```

这个函数结构相当精炼了，通过多个goroutine完成kubelet的启动

- 注册了logserver
- 如果设置了Cloud Provider，那么会启动云资源管理器
- 调用kl.initializeModules启动不依赖 container runtime 的一些模块
- 启动 volume manager
- 执行 kl.syncNodeStatus 定时同步 Node 状态
- 调用kl.fastStatusUpdateOnce启动一个循环更新pod CIDR、runtime状态以及node状态；
- 调用kl.nodeLeaseController.Run启动NodeLease机制，NodeLease机制是一种上报心跳的方式，可以通过更加轻量化节约资源的方式，并提升性能上报node的心跳信息
- 执行 kl.updateRuntimeUp 定时更新 Runtime 状态
- 执行 kl.syncNetworkUtil 定时同步 iptables 规则
- 获取 pk.podKillingCh异常pod， 并定时清理异常 pod
- 启动 statusManager、probeManager、runtimeClassManager
- 启动 pleg模块，该模块主要用于周期性地向 container runtime 上报当前所有容器的状态
- 调用kl.syncLoop启动kublet事件循环

## initializeModules

initializeModules初始化和runtime没有关联的模块

```go
func (kl *Kubelet) initializeModules() error {
	// Prometheus metrics注册
······
	servermetrics.Register()

	// 创建文件目录，
	if err := kl.setupDataDirs(); err != nil {······}

	// 创建容器日志目录
	if _, err := os.Stat(ContainerLogsDir); err != nil {
		if err := kl.os.MkdirAll(ContainerLogsDir, 0755); err != nil {
			return fmt.Errorf("failed to create directory %q: %v", ContainerLogsDir, err)
		}
	}

	// 启动 imageManager，这个管理器定义就是images.ImageGCManager，实际上是realImageGCManager
	kl.imageManager.Start()

	// 启动 certificate manager，证书相关
	if kl.serverCertificateManager != nil {
		kl.serverCertificateManager.Start()
	}

	// 启动 oomWatcher监视器
	if err := kl.oomWatcher.Start(kl.nodeRef); err != nil {·······}

	// 启动 resource analyzer,定时刷新volume stats到缓存中
	kl.resourceAnalyzer.Start()

	return nil
}
```

### imageManager

如上面注释，`kl.imageManager.Start()`最后调用 `pkg/kubelet/images/image_gc_manager.go`

```go
func (im *realImageGCManager) Start() {
	go wait.Until(func() {
	······
		_, err := im.detectImages(ts)
	······
	}, 5*time.Minute, wait.NeverStop)

	go wait.Until(func() {
		images, err := im.runtime.ListImages() //更新镜像缓存
		if err != nil {·······} 
    else {
			im.imageCache.set(images)
		}
	}, 30*time.Second, wait.NeverStop)
}
```

里面主要启动了2个协程，分别调用2个函数，一个是`detectImages`， 一个是`im.imageCache.set`

`detectImages`主要是获取所有Pod和里面container信息，找到正在使用的image，然后和image列表对比，删除没有使用的image

## syncLoop

```go
func (kl *Kubelet) syncLoop(updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
	defer syncTicker.Stop()
	housekeepingTicker := time.NewTicker(housekeepingPeriod)
	defer housekeepingTicker.Stop()

	for {
    ······
		kl.syncLoopMonitor.Store(kl.clock.Now())
		if !kl.syncLoopIteration(updates, handler, syncTicker.C, housekeepingTicker.C, plegCh) {
			break
		}
		kl.syncLoopMonitor.Store(kl.clock.Now())
	}
}
```

主要就是在一个循环中不断的调用syncLoopIteration方法执行主要逻辑。

### syncLoopIteration

这个函数主要是从5个channel里面消费，对应不同的handler

```go
func (kl *Kubelet) syncLoopIteration(configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
	syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {
	select {
  // configCh读取配置事件的管道，该模块将同时 watch 3 个不同来源的 pod 信息的变化（file，http，apiserver），一旦某个来源的 pod 信息发生了更新（创建/更新/删除），这个 channel 中就会出现被更新的 pod 信息和更新的具体操作
	case u, open := <-configCh:
    ······
		switch u.Op {
		case kubetypes.ADD:
			handler.HandlePodAdditions(u.Pods)
		case kubetypes.UPDATE:
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.REMOVE:
			handler.HandlePodRemoves(u.Pods)
		case kubetypes.RECONCILE:
			handler.HandlePodReconcile(u.Pods)
		case kubetypes.DELETE:
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.SET:
		default:
			klog.Errorf("Invalid event type received: %d.", u.Op)
		}
		kl.sourcesReady.AddSource(u.Source)
    
  // PLEG.Start的时候会每秒钟启动调用一次relist，根据最新的PodStatus生成PodLiftCycleEvent，然后存入到PLE Channel中。
  // syncLoop会调用pleg.Watch方法获取PLE Channel管道，然后传给syncLoopIteration方法，在syncLoopIteration方法中也就是plegCh这个管道，syncLoopIteration会消费plegCh中的数据，在 handler 中通过调用 dispatchWork 分发任务。
	case e := <-plegCh:
		if e.Type == pleg.ContainerStarted {

			kl.lastContainerStartedTime.Add(e.ID, time.Now())
		}
		if isSyncPodWorthy(e) {
			if pod, ok := kl.podManager.GetPodByUID(e.ID); ok {
				handler.HandlePodSyncs([]*v1.Pod{pod})
			} else {······}
		}

		if e.Type == pleg.ContainerDied {
			if containerID, ok := e.Data.(string); ok {
				kl.cleanUpContainersInPod(e.ID, containerID)
			}
		}
    
  // syncCh是由syncLoop方法里面创建的一个定时任务，每秒钟会向syncCh添加一个数据，然后就会执行到这里。这个方法会同步所有等待同步的pod。
	case <-syncCh:
		podsToSync := kl.getPodsToSync()
		if len(podsToSync) == 0 {
			break
		}
		handler.HandlePodSyncs(podsToSync)
    
  // 对失败的pod或者liveness检查失败的pod进行sync操作。
	case update := <-kl.livenessManager.Updates():
		// 如果探针检测失败，需要更新pod的状态    
		if update.Result == proberesults.Failure {
			pod, ok := kl.podManager.GetPodByUID(update.PodUID)
			if !ok {
				break
			}
			handler.HandlePodSyncs([]*v1.Pod{pod})
		}
    
  // housekeepingCh这个管道也是由syncLoop创建，每两秒钟会触发清理。
	case <-housekeepingCh:
		if !kl.sourcesReady.AllReady() {
      ······
		} else {
      //执行一些清理工作，包括终止pod workers、删除不想要的pod，移除volumes、pod目录
			if err := handler.HandlePodCleanups(); err != nil {······}
		}
	}
	return true
}
```

# Pod启动流程

Pod的启动在`syncLoop`方法下调用的`syncLoopIteration`方法开始。在`syncLoopIteration`方法内，有5个重要的channel

- configCh：获取Pod信息的channel，关于Pod相关的事件都从该channel获取；
- handler：处理Pod的handler；
- syncCh：同步所有等待同步的Pod；
- houseKeepingCh：清理Pod的channel；
- plegCh：获取PLEG信息，同步Pod。

Pod的启动就是和configCh相关，上面的代码可以看到，Pod新增的时候调用的是`HandlePodAdditions`函数

```go
//  pkg/kubelet/kubelet.go
func (kl *Kubelet) HandlePodAdditions(pods []*v1.Pod) {
······
	for _, pod := range pods {
		existingPods := kl.podManager.GetPods() 
		//将pod添加到pod管理器中，如果有pod不存在在pod管理器中，那么这个pod表示已经被删除了
		kl.podManager.AddPod(pod)
		······
		//如果该pod没有被Terminate
		if !kl.podIsTerminated(pod) { 
			// 获取目前还在active状态的pod
			activePods := kl.filterOutTerminatedPods(existingPods)
 
			//验证 pod 是否能在该节点运行，如果不可以直接拒绝
			if ok, reason, message := kl.canAdmitPod(activePods, pod); !ok {
				kl.rejectPod(pod, reason, message)
				continue
			}
		}
		mirrorPod, _ := kl.podManager.GetMirrorPodByPod(pod)
		//把 pod 分配给给 worker 做异步处理,创建pod
		kl.dispatchWork(pod, kubetypes.SyncPodCreate, mirrorPod, start)
		//在 probeManager 中添加 pod，如果 pod 中定义了 readiness 和 liveness 健康检查，启动 goroutine 定期进行检测
		kl.probeManager.AddPod(pod)
	}
}
```

`kl.podManager.AddPod` 和 `kl.probeManager.AddPod(pod)` 都只是将pod 纳入跟踪，真正创建pod的是`dispatchWork`

## dispatchWork

```go
func (kl *Kubelet) dispatchWork(pod *v1.Pod, syncType kubetypes.SyncPodType, mirrorPod *v1.Pod, start time.Time) {
	...
	kl.podWorkers.UpdatePod(&UpdatePodOptions{
		Pod:        pod,
		MirrorPod:  mirrorPod,
		UpdateType: syncType,
		OnCompleteFunc: func(err error) {
			if err != nil {
         metrics.PodWorkerDuration.WithLabelValues(syncType.String()).Observe(metrics.SinceInSeconds(start))
			}
		},
	})
	...
}
```

`dispatchWork`会封装一个UpdatePodOptions结构体丢给`podWorkers.UpdatePod`去执行。

### UpdatePod

```go
// pkg/kubelet/pod_workers.go

func (p *podWorkers) UpdatePod(options *UpdatePodOptions) {
  ······
	p.podLock.Lock()
	defer p.podLock.Unlock()
  //如果该pod在podUpdates数组里面找不到，那么就创建channel，并启动异步线程
	if podUpdates, exists = p.podUpdates[uid]; !exists {
		podUpdates = make(chan UpdatePodOptions, 1)
		p.podUpdates[uid] = podUpdates
		go func() {
			defer runtime.HandleCrash()
			p.managePodLoop(podUpdates)
		}()
	}
  ······
}
```

这个方法会加锁之后获取podUpdates数组里面数据，如果不存在那么会创建一个channel然后执行一个异步协程。

#### managePodLoop

```go
func (p *podWorkers) managePodLoop(podUpdates <-chan UpdatePodOptions) {
	var lastSyncTime time.Time
  //遍历channel
	for update := range podUpdates {
		err := func() error {
			podUID := update.Pod.UID
			// 直到cache里面有新数据之前这段代码会阻塞，这保证worker在cache里面有新的数据之前不会提前开始。
			status, err := p.podCache.GetNewerThan(podUID, lastSyncTime)
			······
      //syncPodFn会在kubelet初始化的时候设置，调用的是kubelet的syncPod方法
			err = p.syncPodFn(syncPodOptions{
				mirrorPod:      update.MirrorPod,
				pod:            update.Pod,
				podStatus:      status,
				killPodOptions: update.KillPodOptions,
				updateType:     update.UpdateType,
			})
			lastSyncTime = time.Now()
			return err
		}()
		······
	}
}
```

这个方法会遍历channel里面的数据，然后调用syncPodFn方法并传入一个syncPodOptions，kubelet会在执行NewMainKubelet方法的时候调用newPodWorkers方法设置syncPodFn为Kubelet的syncPod方法。

```go
func NewMainKubelet(...){
...
	klet := &Kubelet{...}
...
	klet.podWorkers = newPodWorkers(klet.syncPod, kubeDeps.Recorder, klet.workQueue, klet.resyncInterval, backOffPeriod, klet.podCache)
...
}
```

#### syncPod

```go
// pkg/kubelet/kubelet.go

func (kl *Kubelet) syncPod(o syncPodOptions) error {
	...

	apiPodStatus := kl.generateAPIPodStatus(pod, podStatus)
	
	// 校验该pod能否运行
	runnable := kl.canRunPod(pod)
	//如果不能运行，那么回写container的等待原因
	if !runnable.Admit {
		// Pod is not runnable; update the Pod and Container statuses to why.
		apiPodStatus.Reason = runnable.Reason
		apiPodStatus.Message = runnable.Message
		// Waiting containers are not creating.
		const waitingReason = "Blocked"
		for _, cs := range apiPodStatus.InitContainerStatuses {
			if cs.State.Waiting != nil {
				cs.State.Waiting.Reason = waitingReason
			}
		}
		for _, cs := range apiPodStatus.ContainerStatuses {
			if cs.State.Waiting != nil {
				cs.State.Waiting.Reason = waitingReason
			}
		}
	}

	// 更新状态管理器中的状态
	kl.statusManager.SetPodStatus(pod, apiPodStatus)
 
	// 如果校验没通过或pod已被删除或pod跑失败了，那么kill掉pod
	if !runnable.Admit || pod.DeletionTimestamp != nil || apiPodStatus.Phase == v1.PodFailed {
		var syncErr error
        ...
		kl.killPod(pod, nil, podStatus, nil)
        ....
		return syncErr
	}
 
	//校验网络插件是否已准备好
	if err := kl.runtimeState.networkErrors(); err != nil && !kubecontainer.IsHostNetworkPod(pod) {
		kl.recorder.Eventf(pod, v1.EventTypeWarning, events.NetworkNotReady, "%s: %v", NetworkNotReadyErrorMsg, err)
		return fmt.Errorf("%s: %v", NetworkNotReadyErrorMsg, err)
	}

	// 创建
	pcm := kl.containerManager.NewPodContainerManager()

	// 校验该pod是否已被Terminate
	if !kl.podIsTerminated(pod) { 
		firstSync := true
		// 校验该pod是否首次创建
		for _, containerStatus := range apiPodStatus.ContainerStatuses {
			if containerStatus.State.Running != nil {
				firstSync = false
				break
			}
		} 
		podKilled := false
		// 如果该pod 的cgroups不存在，并且不是首次启动，那么kill掉
		if !pcm.Exists(pod) && !firstSync {
			if err := kl.killPod(pod, nil, podStatus, nil); err == nil {
				podKilled = true
			}
		} 
		// 如果该pod在上面没有被kill掉，或重启策略不是永不重启
		if !(podKilled && pod.Spec.RestartPolicy == v1.RestartPolicyNever) {
			// 如果该pod的cgroups不存在，那么就创建cgroups
			if !pcm.Exists(pod) {
				if err := kl.containerManager.UpdateQOSCgroups(); err != nil {
					klog.V(2).Infof("Failed to update QoS cgroups while syncing pod: %v", err)
				}
				if err := pcm.EnsureExists(pod); err != nil {
					kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToCreatePodContainer, "unable to ensure pod container exists: %v", err)
					return fmt.Errorf("failed to ensure that the pod: %v cgroups exist and are correctly applied: %v", pod.UID, err)
				}
			}
		}
	} 

	//为静态pod 创建 镜像
	if kubetypes.IsStaticPod(pod) {
		...
	}
 
	// 创建pod的文件目录
	if err := kl.makePodDataDirs(pod); err != nil {
		kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedToMakePodDataDirectories, "error making pod data directories: %v", err)
		klog.Errorf("Unable to make pod data directories for pod %q: %v", format.Pod(pod), err)
		return err
	}
 
	// 如果该pod没有被终止，那么需要等待attach/mount volumes
	if !kl.podIsTerminated(pod) {
		// Wait for volumes to attach/mount
		if err := kl.volumeManager.WaitForAttachAndMount(pod); err != nil {
			kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedMountVolume, "Unable to attach or mount volumes: %v", err)
			klog.Errorf("Unable to attach or mount volumes for pod %q: %v; skipping pod", format.Pod(pod), err)
			return err
		}
	}
 
	// 如果有 image secrets，去 apiserver 获取对应的 secrets 数据
	pullSecrets := kl.getPullSecretsForPod(pod)
    // 真正的容器创建逻辑
	result := kl.containerRuntime.SyncPod(pod, podStatus, pullSecrets, kl.backOff)
	kl.reasonCache.Update(pod.UID, result)
	if err := result.Error(); err != nil { 
		for _, r := range result.SyncResults {
			if r.Error != kubecontainer.ErrCrashLoopBackOff && r.Error != images.ErrImagePullBackOff { 
				return err
			}
		}

		return nil
	}
	return nil
}
```

最后真正创建容器的逻辑是调用`containerRuntime.SyncPod`





https://www.jianshu.com/p/9184152aa118

https://www.cnblogs.com/luozhiyun/p/13736569.html