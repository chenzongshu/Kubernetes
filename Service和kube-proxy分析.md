> 本文代码基于Kubernetes 1.15版本

# Service原理

## iptables模式

一旦创建service, kube-proxy 就可以通过 Service 的 Informer 感知到这样一个 Service 对象的添加。而作为对这个事件的响应，它就会在宿主机上创建这样一条 iptables 规则（你可以通过 iptables-save 看到它），如下所示：

```
-A KUBE-SERVICES -d 10.0.1.175/32 -p tcp -m comment --comment "default/hostnames: cluster IP" -m tcp --dport 80 -j KUBE-SVC-NWV5X2332I4OT4T3
```

可以看到，这条 iptables 规则的含义是：凡是目的地址是 10.0.1.175、目的端口是 80 的 IP 包，都应该跳转到另外一条名叫 KUBE-SVC-NWV5X2332I4OT4T3的 iptables 链进行处理。

这个10.0.1.175, 其实就是Service的VIP, 因为VIP只是iptables中一条规则的配置, 并没有任何网络设备, 所以ping不通

我们跳转到的 KUBE-SVC-NWV5X2332I4OT4T3 规则，又有什么作用呢？

实际上，它是一组规则的集合，如下所示：

```
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-WNBA2IHDGP2BOBGZ
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-X3P2623AGDH6CDF3
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -j KUBE-SEP-57KPRZ3JQVENLNBR
```

可以看到，这一组规则，实际上是一组随机模式（–mode random）的 iptables 链。

而随机转发的目的地，分别是 KUBE-SEP-WNBA2IHDGP2BOBGZ、KUBE-SEP-X3P2623AGDH6CDF3 和 KUBE-SEP-57KPRZ3JQVENLNBR。

而这三条链指向的最终目的地，其实就是这个 Service 代理的三个 Pod。所以这一组规则，就是 Service 实现负载均衡的位置。

需要注意的是，iptables 规则的匹配是从上到下逐条进行的，所以为了保证上述三条规则每条被选中的概率都相同，我们应该将它们的 probability 字段的值分别设置为 1/3（0.333…）、1/2 和 1。

这么设置的原理很简单：第一条规则被选中的概率就是 1/3；而如果第一条规则没有被选中，那么这时候就只剩下两条规则了，所以第二条规则的 probability 就必须设置为 1/2；类似地，最后一条就必须设置为 1。

通过查看上述三条链的明细，我们就很容易理解 Service 进行转发的具体原理了，如下所示：

```
-A KUBE-SEP-57KPRZ3JQVENLNBR -s 10.244.3.6/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-57KPRZ3JQVENLNBR -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.3.6:9376

-A KUBE-SEP-WNBA2IHDGP2BOBGZ -s 10.244.1.7/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-WNBA2IHDGP2BOBGZ -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.1.7:9376

-A KUBE-SEP-X3P2623AGDH6CDF3 -s 10.244.2.3/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-X3P2623AGDH6CDF3 -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.2.3:9376
```

可以看到，这三条链，其实是三条 DNAT 规则。DNAT 规则的作用，就是在 PREROUTING 检查点之前，也就是在路由之前，将流入 IP 包的目的地址和端口，改成–to-destination 所指定的新的目的地址和端口。可以看到，这个目的地址和端口，正是被代理 Pod 的 IP 地址和端口。

这样，访问 Service VIP 的 IP 包经过上述 iptables 处理之后，就已经变成了访问具体某一个后端 Pod 的 IP 包了。不难理解，这些 Endpoints 对应的 iptables 规则，正是 kube-proxy 通过监听 Pod 的变化事件，在宿主机上生成并维护的。



## ipvs模式

IPVS 模式的工作原理，其实跟 iptables 模式类似。当我们创建了前面的 Service 之后，kube-proxy 首先会在宿主机上创建一个虚拟网卡（叫作：kube-ipvs0），并为它分配 Service VIP 作为 IP 地址，如下所示：

```
# ip addr
...
73：kube-ipvs0：<BROADCAST,NOARP>  mtu 1500 qdisc noop state DOWN qlen 1000
link/ether  1a:ce:f5:5f:c1:4d brd ff:ff:ff:ff:ff:ff
inet 10.0.1.175/32  scope global kube-ipvs0
valid_lft forever  preferred_lft forever
```

而接下来，kube-proxy 就会通过 Linux 的 IPVS 模块，为这个 IP 地址设置三个 IPVS 虚拟主机，并设置这三个虚拟主机之间使用轮询模式 (rr) 来作为负载均衡策略。我们可以通过 ipvsadm 查看到这个设置，如下所示：

```
# ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
->  RemoteAddress:Port           Forward  Weight ActiveConn InActConn     
TCP  10.102.128.4:80 rr
->  10.244.3.6:9376    Masq    1       0          0         
->  10.244.1.7:9376    Masq    1       0          0
->  10.244.2.3:9376    Masq    1       0          0
```

可以看到，这三个 IPVS 虚拟主机的 IP 地址和端口，对应的正是三个被代理的 Pod。

这时候，任何发往 10.102.128.4:80 的请求，就都会被 IPVS 模块转发到某一个后端 Pod 上了。

而相比于 iptables，IPVS 在内核中的实现其实也是基于 Netfilter 的 NAT 模式，所以在转发这一层上，理论上 IPVS 并没有显著的性能提升。但是，IPVS 并不需要在宿主机上为每个 Pod 设置 iptables 规则，而是把对这些“规则”的处理放到了内核态，从而极大地降低了维护这些规则的代价

不过需要注意的是，IPVS 模块只负责上述的负载均衡和代理功能。而一个完整的 Service 流程正常工作所需要的包过滤、SNAT 等操作，还是要靠 iptables 来实现。只不过，这些辅助性的 iptables 规则数量有限，也不会随着 Pod 数量的增加而增加。

## nodePort与SNAT

nodePort原理几乎和ClusterIP一样, 就是在每台宿主机上生成这样一条 iptables 规则：

```
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/my-nginx: nodePort" -m tcp --dport 8080 -j KUBE-SVC-67RL4FN6JRUPOJYM
```

需要注意的是，在 NodePort 方式下，Kubernetes 会在 IP 包离开宿主机发往目的 Pod 时，对这个 IP 包做一次 SNAT 操作，如下所示：

```
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE
```

可以看到，这条规则设置在 POSTROUTING 检查点，也就是说，它给即将离开这台主机的 IP 包，进行了一次 SNAT 操作，将这个 IP 包的源地址替换成了这台宿主机上的 CNI 网桥地址，或者宿主机本身的 IP 地址（如果 CNI 网桥不存在的话）。

当然，这个 SNAT 操作只需要对 Service 转发出来的 IP 包进行（否则普通的 IP 包就被影响了）。而 iptables 做这个判断的依据，就是查看该 IP 包是否有一个“0x4000”的“标志”。

可是，**为什么一定要对流出的包做 SNAT操作呢**

道理很简单, 当Client通过nodePort访问Pod时候, 首先放到到Node1, 可能实际得Pod并不在上面, 还需要把数据包转发到Node2上, Node2处理完数据会根据数据包中的源地址回包, 即Node2直接给Client回包, 对Client来说, 明明给Node1发的包, 回包确是Node2, 很可能报错.

所以需要SNAT, 在离开Node1的时候把源IP改为Node1的CNI网桥地址或者Node1的地址, 这样Pod处理完后将回包到Node1, 再由Node1回包到Client

也可以将 Service 的 spec.externalTrafficPolicy 字段设置为 local, 关闭SNAT, 适用于那些Pod需要看到真正请求源地址的场景

# kube-proxy

## 功能简介

`kube-proxy`运行在 kubernetes集群中每个 worker节点上，负责实现 `service`这个概念提供的功能，kubeproxy会把访问 service VIP的请求转发到运行的`pods`上实现负载均衡

当用户创建 `service`的时候， `endpointcontroller`会根据 `service`的 `selector`找到对应的pod，然后生成`endpoints`对象保存到`etcd`中， `kube-proxy`的主要工作就是监听etcd（通过 apiserver的接口，而不是直接读取etcd）来实时更新节点上的 `iptables`

## kube-proxy入口

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

## NewProxyCommand()

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

### Options.Run()

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

### runLoop()

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

#### ServiceConfig

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

### Proxier

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




