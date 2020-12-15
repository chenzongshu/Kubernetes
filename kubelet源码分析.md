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

最核心的就是Run函数，从名字也能看出来就是kubelet启动函数，入参是一个context，上面生成的两个配置，然后通过 `Run()` -> `run()` -> `RunKubelet()` -> `startKubelet()` ，中间是设置各种参数的过程

```
func startKubelet(k kubelet.Bootstrap, podCfg *config.PodConfig, kubeCfg *kubeletconfiginternal.KubeletConfiguration, kubeDeps *kubelet.Dependencies, enableCAdvisorJSONEndpoints, enableServer bool) {
	// start the kubelet
	go k.Run(podCfg.Updates())

	// start the kubelet server
	if enableServer {
```

- `kubelet.Bootstrap`是一个interface， 对应实现在 `pkg/kubelet/kubelet.go`
- `RunKubelet()`里面的`createAndInitKubelet()`配置kubelet参数，启了各种Manager，配置`Kubelet`

# Run函数

`pkg/kubelet/kubelet.go`

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

