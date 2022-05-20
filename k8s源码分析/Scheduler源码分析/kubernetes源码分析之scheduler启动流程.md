> 本文基于k8s 1.21版本



# 启动

老地方，`cmd/kube-scheduler/app/server.go`

```go
func NewSchedulerCommand(registryOptions ...Option) *cobra.Command {
  opts, err := options.NewOptions()  //配置参数初始化(1)
  ······
  // 通过 runCommand() 来启动，同时把 Options 作为第三个参数传入(3)
		Run: func(cmd *cobra.Command, args []string) {
			if err := runCommand(cmd, opts, registryOptions...); err != nil {
  ······
  // 创建完 Options 对象后，会为其添加命令行参数(2)
	fs := cmd.Flags()
	namedFlagSets := opts.Flags()
	verflag.AddFlags(namedFlagSets.FlagSet("global"))
	globalflag.AddGlobalFlags(namedFlagSets.FlagSet("global"), cmd.Name())
	for _, f := range namedFlagSets.FlagSets {
		fs.AddFlagSet(f)
	}
  
```

## 参数初始化

```go
func NewOptions() (*Options, error) {}
```

上面函数返回的是`Options`结构体。所有的配置都存储于 `Options` 结构体中。

```go
type Options struct {
	ComponentConfig kubeschedulerconfig.KubeSchedulerConfiguration 	// 调度策略相关的参数. 
	SecureServing           *apiserveroptions.SecureServingOptionsWithLoopback
	CombinedInsecureServing *CombinedInsecureServingOptions
	Authentication          *apiserveroptions.DelegatingAuthenticationOptions
	Authorization           *apiserveroptions.DelegatingAuthorizationOptions
	Metrics                 *metrics.Options
	Logs                    *logs.Options
  // 旧的调度策略相关的参数，它们已经标记为过期并且会在将来的版本中移除掉
	Deprecated              *DeprecatedOptions
	// ConfigFile is the location of the scheduler server's configuration file.
	ConfigFile string
	// WriteConfigTo is the path where the default configuration will be written.
	WriteConfigTo string
	Master string
}
```

## runCommand

根据上面启动的入口

```go
func runCommand(cmd *cobra.Command, opts *options.Options, registryOptions ...Option) error {
  ······
	cc, sched, err := Setup(ctx, opts, registryOptions...) //创建完整配置
  ······
	return Run(ctx, cc, sched) // Run函数，传入*scheduler.Scheduler
}
```

### Setup()

```go
func Setup(ctx context.Context, opts *options.Options, outOfTreeRegistryOptions ...Option) (*schedulerserverconfig.CompletedConfig, *scheduler.Scheduler, error) {
	if errs := opts.Validate() //对所有参数进行有效性验证

	c, err := opts.Config() //判断如果需要将配置写入文件，则执行写入操作并退出

	cc := c.Complete() //获取全部配置

	// Create the scheduler.
	sched, err := scheduler.New(cc.Client,
		cc.InformerFactory,
		recorderFactory,
		ctx.Done(),
		scheduler.WithProfiles(cc.ComponentConfig.Profiles...),
		scheduler.WithAlgorithmSource(cc.ComponentConfig.AlgorithmSource),
		scheduler.WithPercentageOfNodesToScore(cc.ComponentConfig.PercentageOfNodesToScore),
		scheduler.WithFrameworkOutOfTreeRegistry(outOfTreeRegistry),
		scheduler.WithPodMaxBackoffSeconds(cc.ComponentConfig.PodMaxBackoffSeconds),
		scheduler.WithPodInitialBackoffSeconds(cc.ComponentConfig.PodInitialBackoffSeconds),
		scheduler.WithExtenders(cc.ComponentConfig.Extenders...),
		scheduler.WithParallelism(cc.ComponentConfig.Parallelism),
		scheduler.WithBuildFrameworkCapturer(func(profile kubeschedulerconfig.KubeSchedulerProfile) {
			// Profiles are processed during Framework instantiation to set default plugins and configurations. Capturing them for logging
			completedProfiles = append(completedProfiles, profile)
		}),
	)

	return &cc, sched, nil
}
```

#### scheduler.New()

```go
// New()是Scheduler的构造函数，其中clientset,informerFactory,recorderFactory是调用者传入的，所以构造函数不用创建。
// opts就是在默认的schedulerOptions叠加的选项，golang这种用法非常普遍，不需要多解释。
func New(client clientset.Interface,
	informerFactory informers.SharedInformerFactory,
	recorderFactory profile.RecorderFactory,
	stopCh <-chan struct{},
	opts ...Option) (*Scheduler, error) {

	// 初始化stopEverything，如果调用者没有指定则永远不会停止(这种情况不可能，因为要优雅退出)。
	stopEverything := stopCh
	if stopEverything == nil {
		stopEverything = wait.NeverStop
	}

  // 在默认的schedulerOptions基础上应用所有的opts
	options := defaultSchedulerOptions
	for _, opt := range opts {
		opt(&options)
	}

	// 构造调度缓存，还记得第一个参数30秒是干什么的么？是绑定的超时阈值(TTL)，指定时间内没有确认假定调度的Pod将会被从Cache中移除
	schedulerCache := internalcache.New(30*time.Second, stopEverything)

	// 创建InTree的插件工厂注册表，并与OutTree的插件工厂注册表合并，形成最终的插件工厂注册表。
	// registry是一个map类型，键是插件名称，值是插件的工厂(构造函数)
	registry := frameworkplugins.NewInTreeRegistry()
	if err := registry.Merge(options.frameworkOutOfTreeRegistry); err != nil {
		return nil, err
	}

	// 初始化Cache的快照
	snapshot := internalcache.NewEmptySnapshot()

	// Configurator用于根据配置构造Scheduler。
	configurator := &Configurator{
  ·······
	}

	metrics.Register()

	var sched *Scheduler
	// 根据算法源调用Configurator不同的接口构造Scheduler。
	// 因为算法源已经不推荐使用，所以可以认为是用configurator.createFromProvider()构造Scheduler。
	source := options.schedulerAlgorithmSource
	switch {
	// 基于Provider的算法源
	case source.Provider != nil:
		sc, err := configurator.createFromProvider(*source.Provider)
		if err != nil {
			return nil, fmt.Errorf("couldn't create scheduler using provider %q: %v", *source.Provider, err)
		}
		sched = sc
	// 基于策略的算法源
	case source.Policy != nil:
  ······
	default:
		return nil, fmt.Errorf("unsupported algorithm source: %v", source)
	}
	// 设置Scheduler的Clientset和关闭信号
	sched.StopEverything = stopEverything
	sched.client = client

	// 注册事件处理函数，就是通过SharedIndexInformer监视Pod、Node、Service等调度依赖的API对象，并根据事件类型执行相应的操作。
	// 其中新建的Pod放入调度队列、Pod绑定成功更新调度缓存都是事件处理函数做的。
	addAllEventHandlers(sched, informerFactory)
	return sched, nil
}
```

该函数返回了一个scheduler。其中插件的new

```go
registry := frameworkplugins.NewInTreeRegistry()
```

创建的逻辑在如下位置

```go
// createFromProvider()根据名字找到已注册的算法提供者以此构造Scheduler。
func (c *Configurator) createFromProvider(providerName string) (*Scheduler, error) {
	klog.V(2).InfoS("Creating scheduler from algorithm provider", "algorithmProvider", providerName)
	// NewRegistry()返回了默认的插件配置，getDefaultConfig()就在里面
	r := algorithmprovider.NewRegistry()
	defaultPlugins, exist := r[providerName]
	if !exist {
		return nil, fmt.Errorf("algorithm provider %q is not registered", providerName)
	}

	// 遍历所有的Profile.
	for i := range c.profiles {
		prof := &c.profiles[i]
		plugins := &schedulerapi.Plugins{}
		// 在默认插件配置基础上应用用户的自定义配置，即使能新的插件或者禁止某些默认插件
		plugins.Append(defaultPlugins)
		plugins.Apply(prof.Plugins)
		// 将最终的插件配置更新到Profile中用于创建Framework，这个在前面已经提到了。
		prof.Plugins = plugins
	}
	// 创建Scheduler
	return c.create()
}
```

 `getDefaultConfig()` 这里面就能看到默认策略初始化plugins的哪些点注册了哪些插件

```go
func getDefaultConfig() *schedulerapi.Plugins {
	plugins := &schedulerapi.Plugins{
		QueueSort: schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: queuesort.Name},
			},
		},
		PreFilter: schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: noderesources.FitName},
				{Name: nodeports.Name},
				{Name: podtopologyspread.Name},
				{Name: interpodaffinity.Name},
				{Name: volumebinding.Name},
				{Name: nodeaffinity.Name},
			},
		},
		Filter: schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: nodeunschedulable.Name},
				{Name: nodename.Name},
				{Name: tainttoleration.Name},
				{Name: nodeaffinity.Name},
				{Name: nodeports.Name},
				{Name: noderesources.FitName},
				{Name: volumerestrictions.Name},
				{Name: nodevolumelimits.EBSName},
				{Name: nodevolumelimits.GCEPDName},
				{Name: nodevolumelimits.CSIName},
				{Name: nodevolumelimits.AzureDiskName},
				{Name: volumebinding.Name},
				{Name: volumezone.Name},
				{Name: podtopologyspread.Name},
				{Name: interpodaffinity.Name},
			},
		},
		PostFilter: schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: defaultpreemption.Name},
			},
		},
		PreScore: schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: interpodaffinity.Name},
				{Name: podtopologyspread.Name},
				{Name: tainttoleration.Name},
				{Name: nodeaffinity.Name},
			},
		},
		Score: schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: noderesources.BalancedAllocationName, Weight: 1},
				{Name: imagelocality.Name, Weight: 1},
				{Name: interpodaffinity.Name, Weight: 1},
				{Name: noderesources.LeastAllocatedName, Weight: 1},
				{Name: nodeaffinity.Name, Weight: 1},
				{Name: nodepreferavoidpods.Name, Weight: 10000},
				{Name: podtopologyspread.Name, Weight: 2},
				{Name: tainttoleration.Name, Weight: 1},
			},
		},
		Reserve: schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: volumebinding.Name},
			},
		},
		PreBind: schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: volumebinding.Name},
			},
		},
		Bind: schedulerapi.PluginSet{
			Enabled: []schedulerapi.Plugin{
				{Name: defaultbinder.Name},
			},
		},
	}
	if utilfeature.DefaultFeatureGate.Enabled(features.VolumeCapacityPriority) {
		plugins.Score.Enabled = append(plugins.Score.Enabled, schedulerapi.Plugin{Name: volumebinding.Name, Weight: 1})
	}
	return plugins
}
```

需要注意的是，有可能同一个插件实现了多个扩展点，因此会出现多次。

在获取了插件列表后，还会使用 `applyFeatureGates()` 做一些额外的修改，主要是根据是否开启了 `EvenPodsSpread` 和 `ResourceLimitsPriorityFunction` 特性来决定是否加入额外的插件

#### profile

Profile 是在 Kubernetes Scheduler 代码中完成初始化后，在调度过程中使用的一个对象

其实就是 `map[string]framework.Framework`，`framework.Framework`即调度框架

```go
func (c *Configurator) createFromProvider(providerName string) (*Scheduler, error) {
	r := algorithmprovider.NewRegistry()
  ······
	for i := range c.profiles {  //这里的c.profiles是KubeSchedulerProfile
  ······
		prof.Plugins = plugins
	}
	return c.create()  //在里面完成从 KubeSchedulerProfile 到 Profile 的转换
}
```

##### 使用

调度的主流程在 `scheduleOne()` 中。

```go
func (sched *Scheduler) scheduleOne(ctx context.Context) {
	podInfo := sched.NextPod()
	// pod could be nil when schedulerQueue is closed
	if podInfo == nil || podInfo.Pod == nil {
		return
	}
	pod := podInfo.Pod
	prof, err := sched.profileForPod(pod)

    ...
}
func (sched *Scheduler) profileForPod(pod *v1.Pod) (*profile.Profile, error) {
	prof, ok := sched.Profiles[pod.Spec.SchedulerName]
	if !ok {
		return nil, fmt.Errorf("profile not found for scheduler name %q", pod.Spec.SchedulerName)
	}
	return prof, nil
}
```

可以看出在调度时会从 Pod 中获取调度器名称，然后根据名称再从这里的 Map 对象中找到对应的 profile 对象，再使用 profile 以及它内部的 framework.Framework 对象来执行具体的调度过程。

### Run()

进入Run，又嵌套了一层

```go
func Run(ctx context.Context, cc *schedulerserverconfig.CompletedConfig, sched *scheduler.Scheduler) error {
  ······
  // 等待选主
	waitingForLeader := make(chan struct{})
······

	// Start all informers.
	cc.InformerFactory.Start(ctx.Done())

	// Wait for all caches to sync before scheduling.
	cc.InformerFactory.WaitForCacheSync(ctx.Done())

	sched.Run(ctx)
}
```

