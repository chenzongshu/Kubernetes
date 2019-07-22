> 本文代码基于Kubernetes 1.15版本

# kube-proxy功能简介

`kube- proxy`运行在 kubernetes集群中每个 worker节点上，负责实现 `service`这个概念提供的功能，kubeproxy会把访问 service VIP的请求转发到运行的`pods`上实现负载均衡

当用户创建 `service`的时候， `endpointcontroller`会根据 `service`的 `selector`找到对应的pod，然后生成`endpoints`对象保存到`etcd`中， `kube-proxy`的主要工作就是监听etcd（通过 apiserver的接口，而不是直接读取etcd）来实时更新节点上的 `iptables`

# kube-proxy入口

和 kubernetes其他组件一样，`kube-proxy`入口在 `kubernetes／cmd／ kube-proxy`，具体的代码如下

```go
func main() {
......
	command := app.NewProxyCommand()

	pflag.CommandLine.SetNormalizeFunc(cliflag.WordSepNormalizeFunc)
	pflag.CommandLine.AddGoFlagSet(goflag.CommandLine)

	logs.InitLogs()
	defer logs.FlushLogs()

	if err := command.Execute(); err != nil {
		fmt.Fprintf(os.Stderr, "error: %v\n", err)
		os.Exit(1)
	}
}
```

- `app.NewProxyCommand()`创建了一个 `cobra.Command`对象 cobra是 go语言通用框架, 构建一个标准的命令应用
- 建命令行的库初始化log，默认刷新间隔30秒
- 执行命令`command.Execute()`

# NewProxyCommand()

先看看入口

```go
func NewProxyCommand() *cobra.Command {
	opts := NewOptions()

	cmd := &cobra.Command{
        Use: "kube-proxy",
        
		Run: func(cmd *cobra.Command, args []string) {
            ......
			if err := opts.Validate(args); err != nil {
				klog.Fatalf("failed validate: %v", err)
			}
			klog.Fatal(opts.Run())
		},
	}
......
	return cmd
}
```

主要步骤如下

- `NewOptions()`创建了一个 `Options`结构体并初始化完成参数的初始化, `Options`包含了 `kube-proxy`需要的一切数据
- 调用 `opts.Run()` 函数

## Options.Run()

下面来看看`opts.Run()`函数就是跑个 proxy server起来, 里面逻辑很少

```go
func (o *Options) Run() error {
	defer close(o.errCh)
	if len(o.WriteConfigTo) > 0 {
		return o.writeConfigFile()
	}

	proxyServer, err := NewProxyServer(o)
	if err != nil {
		return err
	}

	if o.CleanupAndExit {
		return proxyServer.CleanupAndExit()
	}

	o.proxyServer = proxyServer
	return o.runLoop()
}
```

里面的主要操作就是两步

- 1、`NewProxyserver()`调用了 `newProxyServer()` 初始化了一个 `ProxyServer`，只是做了一堆参数初始化分析过程略
- 2、然后执行 `o.runLoop()` 把第一步初始化的 `proxyServer`跑起来

我们先来看看返回的 `ProxyServer`结构体，这个结构体包含了所有 `kube-proxy`需要的参数

```go
type ProxyServer struct {
	Client                 clientset.Interface
	EventClient            v1core.EventsGetter
	IptInterface           utiliptables.Interface
	IpvsInterface          utilipvs.Interface
	IpsetInterface         utilipset.Interface
	execer                 exec.Interface
	Proxier                proxy.ProxyProvider
	Broadcaster            record.EventBroadcaster
	Recorder               record.EventRecorder
	ConntrackConfiguration kubeproxyconfig.KubeProxyConntrackConfiguration
	Conntracker            Conntracker // if nil, ignored
	ProxyMode              string
	NodeRef                *v1.ObjectReference
	CleanupIPVS            bool
	MetricsBindAddress     string
	EnableProfiling        bool
	OOMScoreAdj            *int32
	ResourceContainer      string
	ConfigSyncPeriod       time.Duration
	HealthzServer          *healthcheck.HealthzServer
}
```

部分参数解释如下:

- `client`：连接到 kubernetes api server 的客户端对象
- `config`: proxyServer 配置对象，包含了所有的配置信息
- `iptInterface:` iptables 对象，运行执行所有的 iptables 操作
- `proxier`: proxier 是具体执行转发逻辑的对象，不管 userspace 模式还是 iptables 模式，都是一个 proxier 对象
- `eventBroadcaster`: 事件广播对象，把 kube-proxy 内部发生的事件发送到 apiserver
- `recorder`: 事件记录对象
- `conntracker`: connection track 有关的操作
- `proxyMode`: 代理的模式，目前4种: iptables, userspace, ipvs, kernelspace, 1.2版本之前默认是kernelspace

然后看看 `runLoop()`函数

## runLoop()

该函数较短, 主要做了watcher和proxyServer的启动, 调用了2个Run()函数, 其中proxyServer是通过goroutine来启动

```go
func (o *Options) runLoop() error {
	if o.watcher != nil {
		o.watcher.Run()
	}

	// run the proxy in goroutine
	go func() {
		err := o.proxyServer.Run()
		o.errCh <- err
	}()

	for {
		err := <-o.errCh
		if err != nil {
			return err
		}
	}
}
```

其中`o.proxyServer.Run()`就是主流程, 这个是一个interface, 具体实现代码就在该文件中, 如下(省略部分):

```go
func (s *ProxyServer) Run() error {
......
	if s.OOMScoreAdj != nil {
		oomAdjuster = oom.NewOOMAdjuster()
		if err := oomAdjuster.ApplyOOMScoreAdj(0, int(*s.OOMScoreAdj)); err != nil {
			klog.V(2).Info(err)
		}
	}
......
	if s.Broadcaster != nil && s.EventClient != nil {
		s.Broadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: s.EventClient.Events("")})
	}

	if s.HealthzServer != nil {
		s.HealthzServer.Run()
	}

	if len(s.MetricsBindAddress) > 0 {
		proxyMux := mux.NewPathRecorderMux("kube-proxy")
        ......
	}

	informerFactory := informers.NewSharedInformerFactoryWithOptions(s.Client, s.ConfigSyncPeriod,
		informers.WithTweakListOptions(func(options *v1meta.ListOptions) {
			options.LabelSelector = "!" + apis.LabelServiceProxyName
		}))

	serviceConfig := config.NewServiceConfig(informerFactory.Core().V1().Services(), s.ConfigSyncPeriod)
	serviceConfig.RegisterEventHandler(s.Proxier)
	go serviceConfig.Run(wait.NeverStop)

	endpointsConfig := config.NewEndpointsConfig(informerFactory.Core().V1().Endpoints(), s.ConfigSyncPeriod)
	endpointsConfig.RegisterEventHandler(s.Proxier)
	go endpointsConfig.Run(wait.NeverStop)

	informerFactory.Start(wait.NeverStop)

	s.birthCry()

	s.Proxier.SyncLoop()
	return nil
}
```

干了如下几件事情, 其中6是关键

- 1、设置OOM数值
- 2、`StartRecordingTosink`开始将从指定的 `eventBroadcaster`接收的事件发送到给定的接收器
- 3、起一个 `HealthzServer`
- 4、起一个 `metrics server`，性能度量
- 5、使用 `Conntracker`设置了 `netfilter`参数, 该部分代码上面省略
- 6、创建 `service`和 `endpoint`的 `Informer`，并运行，即通过这两个`Informer`来及时获取集群中 `service`和`endpoint`资源的变化, 这一过程下面讲解
- 7、调用 `birthCry`方法。这个方法没什么特别的，就是记录一个 `kube-proxy`启动的事件
- 8、调用 `SyncLoop`方法，持续运行 `Proxier`, `Proxier`实例化对象是在proxy server对象创建时通过config配置文件或"-proxy-mode"指定

### ServiceConfig

`ServiceConfig`和 `endpointsConfig`是一样的, 是 `kube-proxy`中用于监听 `service`和 `endpoint`变化的组件，其本质就是`Informer`，代码在 `pkg/proxy/config/config`：

从上面的第6步的代码可以看出 `ServiceConfig` 经历了三步第一步是新建，结构体如下

```go
struct {
    listerSynced InformerSynced
    eventHandlers []ServiceHandler
}
```

```go
func NewServiceConfig(serviceInformer coreinformers.ServiceInformer, resyncPeriod time.Duration) *ServiceConfig {
	result := &ServiceConfig{
		listerSynced: serviceInformer.Informer().HasSynced,
	}

	serviceInformer.Informer().AddEventHandlerWithResyncPeriod(
		cache.ResourceEventHandlerFuncs{
			AddFunc:    result.handleAddService,
			UpdateFunc: result.handleUpdateService,
			DeleteFunc: result.handleDeleteService,
		},
		resyncPeriod,
	)

	return result
}
```

这个函数为`serviceInformer`添加了三个回调函数, 当`service`发生变化的时候就会那个调用相应的函数

第二步是注册, 把`Proxier`注册进去, 比如使用iptables就是对于的`Proxier`

第三步就是启动一个goroutine来执行`serviceConfig.Run()`

```go
func (c *ServiceConfig) Run(stopCh <-chan struct{}) {
	klog.Info("Starting service config controller")

	if !controller.WaitForCacheSync("service config", stopCh, c.listerSynced) {
		return
	}

	for i := range c.eventHandlers {
		klog.V(3).Info("Calling handler.OnServiceSynced()")
		c.eventHandlers[i].OnServiceSynced()
	}
}
```

主要是做了cache同步, 此方法的核心就是调用handler的`OnServiceSynced`方法。进入`OnServiceSynced`, 此方法是一个interface

比如iptables就是 `pkg/proxy/iptables/proxier.go`里面

```go
func (proxier *Proxier) OnServiceSynced() {
	proxier.mu.Lock()
	proxier.servicesSynced = true
	proxier.setInitialized(proxier.servicesSynced && proxier.endpointsSynced)
	proxier.mu.Unlock()

	proxier.syncProxyRules()
}
```

首先调用`setInitialized`方法，将`proxier`初始化。只有经过初始化后，`proxier`才会开始调用回调函数。最后，就是执行一次`syncProxyRules`方法。

总而言之，Run方法是将刚创建好的`ServiceConfig`初始化，并在初始化后先调用一次`syncProxyRules`方法。而这个方法，就是`kube-proxy`维护iptables(或者ipvs)的具体操作, 以iptables举例, 代码位于`/pkg/proxy/iptables/proxier.go`

这里面是个很长很长的故事, 代码越有800行, 里面就是各种往iptables里面写规则, 这是iptables proxer最核心的逻辑代码

```go
func (proxier *Proxier) syncProxyRules() {
......
}
```

## Proxier

下面来看看`ProxyServer`中另外一个结构体`Proxier`, 是其中一个成员变量`Proxier proxy.ProxyProvider` ,它是interface的具体实现, interface如下:

```go
type ProxyProvider interface {
	config.EndpointsHandler
	config.ServiceHandler

	Sync()

	SyncLoop()
}
```

对应的实现在`pkg/proxy/iptables/proxier.go`中




