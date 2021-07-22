# 前言



# 源码结构

`client-go`已经集成到 `kubernetes`源码当中， 目录是`vendor\k8s.io\client-go`

```
# tree . -L 1
.
......
├── discovery
├── dynamic
├── examples
├── Godeps
├── informers
├── INSTALL.md
├── kubernetes
├── kubernetes_test
├── listers
├── metadata
├── OWNERS
├── pkg
├── plugin
├── rest
├── restmapper
├── scale
├── SECURITY_CONTACTS
├── testing
├── third_party
├── tools
├── transport
└── util
```

| 源码目录   | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| discovery  | 提供discovery client客户端                                   |
| dynamic    | 提供dynamic client客户端                                     |
| informers  | 每种kubernetes资源的informer实现                             |
| kubernetes | 提供clientsetclient客户端                                    |
| listers    | 为每种kubernetes资源提供Lister功能, 该功能对Get和List请求提供只读的缓存数据 |
| plugin     | 提供openstack, GCP, Azure等云服务商的授权插件                |
| rest       | 提供RESTClient客户端, 对APIServer执行Restful 操作            |
| scale      | 提供ScaleClient客户端, 用于扩缩容Deployment, RS, RC等资源对象 |
| tools      | 提供常用工具, 例如SharedInformer, Reflector, DealtFIFO, Indexers. 提供Client查询和缓存机制, 以减少向APIServer发起的请求 |
| transport  | 提供安全的TCP连接, 支持HTTP Stream, 某些操作需要在客户端和容器之间传输二进制流, 例如exec, attach等操作. 该功能由内部的spdy包提供 |
| util       | 提供常用方法, 例如WorkQueue工作队列, Certificate证书管理等   |

# Client-go架构

client-go来实现的controller架构图见官方文档 [**client-go-controller-interaction**](https://github.com/kubernetes/sample-controller/blob/master/docs/images/client-go-controller-interaction.jpeg)

里面最主要包含的就是三块东西

1、informer：

2、workqueue：

3、客户端：使用

# Client-go客户端

目前4种客户端，最基础的是`RESTClient`，其他都是封装`RESTClient`而来的

- `RESTClient`：最基础的客户端，封装了HTTP Request，实现Restful风格
- `ClientSet`：对`RESTClient`进行了对象方式的封装，以Resource和Version方式暴露，但是不能实现CRD
- `DynamicClient`：动态客户端，可以操作任意资源包括CRD，返回结果是 `map[string]interface{}`
- `DiscoveryClient`：可以获取Resource，Version。

# Informer

kubernetes的组件之间都是通过http通信，在不使用中间件的情况下，就是informer机制来保证消息的实时性，可靠性，顺序性。其他组件都是通过client-go的informer机制来和apiserver通信的。

Informer中有多个核心组件

- Reflector：把从API Server数据获取到的数据放到DeltaFIFO队列中，充当生产者角色
- DeltaFIFO：FIFO这个先进先出的队列加上Delta这个可以保存资源操作类型的存储
- Indexer：本地缓存的存储组件并自带索引

核心点：

**在初始化的时候会从Kubernetes API Server获取对应Resource的全部Object，后续只会通过Watch机制接收API Server推送过来的数据，不会再主动从API Server拉取数据，直接使用本地缓存中的数据以减少API Server的压力。**

Watch机制基于HTTP的Chunked实现，维护一个长连接，减少请求的数据量。

SharedInformer可以让同一种资源使用的是同一个Informer，例如v1版本的Deployment和v1beta1版本的Deployment同时存在的时候，共享一个Informer。

每一个Kubernetes资源都实现了Informer机制，比如PodInformer，就实现了Informer() ， Lister()

```
//  client-go/informers/core/v1/pod.go
type PodInformer interface {
	Informer() cache.SharedIndexInformer
	Lister() v1.PodLister
}
```



## 概念释义

### resourceVersion

每个 Kubernetes 对象都有一个 `resourceVersion` 字段，代表该资源在下层数据库中 存储的版本。检视资源集合（名字空间作用域或集群作用域）时，服务器返回的响应 中会包含 `resourceVersion` 值，可用来向服务器发起 watch 请求。 服务器会返回所提供的 `resourceVersion` 之后发生的所有变更（创建、删除和更新）。 这使得客户端能够取回当前的状态并监视其变更，且不会错过任何变更事件。 

客户端的监视连接被断开时，可以从最后返回的 `resourceVersion` 重启新的监视连接， 或者执行一个新的集合请求之后从头开始监视操作

## Indexer

Reflector从DeltaFIFO中消费出来的资源对象存储至Indexer中，Indexer和Etcd集群中的数据保持完全一样，这样减轻了APIServer和Etcd的压力。

indexer定义在`client-go/tools/cache/index.go`中， 

```go
type Indexer interface {
	Store
	Index(indexName string, obj interface{}) ([]interface{}, error)
	IndexKeys(indexName, indexedValue string) ([]string, error)
	ListIndexFuncValues(indexName string) []string
	ByIndex(indexName, indexedValue string) ([]interface{}, error)
	GetIndexers() Indexers
	AddIndexers(newIndexers Indexers) error
}
```

上面可以看到有`Store`的继承，原来的Interface定义在`cliet-go/tool/cache/store.go`中，里面就是对象的CRUD

在`Store`的基础上，Indexer怎么实现索引的呢？ 就是通过下面这些：

```go
// k8s.io/client-go/tools/cache/indexer.go

// 用于计算一个对象的索引键集合
type IndexFunc func(obj interface{}) ([]string, error)

// 索引键与对象键集合的映射
type Index map[string]sets.String

// 索引器名称（或者索引分类）与 IndexFunc 的映射，相当于存储索引的各种分类
type Indexers map[string]IndexFunc

// 索引器名称与 Index 索引的映射
type Indices map[string]Index
```

这几种概念非常晦涩，下面举个例子来说明

**`Index`**: 如果我们需要查找某个节点上所有Pod，对应的就是Index，如果按照ns进行索引，对应的Index就是 `map[namespace]sets.pod`；

**`IndexFunc`**：Pod我们可以按照NodeName来找，也可以按照Namespaces来找，不同的查找方式就有不同的IndexFunc，这里面就有`NamespaceIndexFunc` 与`NodeNameIndexFunc`

**`Indexers`**：存储索引器，key 为索引器名称，value 为索引器的实现函数，如果按照ns进行索引就是 `map["namespace"]MetaNamespaceIndexFunc`

**`Indices`**: 存储缓存器，key 为索引器名称，value 为缓存的数据，如果按照ns进行索引就是 `map["namespace"]map[namespace]sets.pod`。

对于kubernetes来说，所有的对象(Pod、Node、Service等等)都是有属性/标签的，如果属性/标签就是索引器，Indexer就会把相同属性/标签的所有对象放在一个集合中，如果在对属性/标签分一下类，就上面的思想。



store是一个cache类型的CRUD，cache是一个包内私有的，定义如下：

```go
type cache struct {
	cacheStorage ThreadSafeStore
	keyFunc KeyFunc
}
```

可以看到，本质上`cache`是通过`ThreadSafeStore`来实现的

```go
type threadSafeMap struct {
	lock  sync.RWMutex
	items map[string]interface{}
	indexers Indexers
	indices Indices
}
```

`threadSafeMap`实现了该接口，其就是一个线程安全并带有锁的map，数据只会存放在内存中，每次涉及操作都会进行加锁

## DeltaFIFO



```go

// client-go/tools/cache/delta_fifo.go
type DeltaFIFO struct {
    lock sync.RWMutex             // 读写锁，因为涉及到同时读写，读写锁性能要高
    cond sync.Cond                // 给Pop()接口使用，在没有对象的时候可以阻塞，内部锁复用读写锁
    items map[string]Deltas       // 这个应该是Store的本质了，按照kv的方式存储对象，但是存储的是对象的Deltas数组
    queue []string                // 这个是为先入先出实现的，存储的就是对象的键
    populated bool                // 通过Replace()接口将第一批对象放入队列，或者第一次调用增、删、改接口时标记为true
    initialPopulationCount int    // 通过Replace()接口将第一批对象放入队列的对象数量
    keyFunc KeyFunc               // 对象键计算函数
    knownObjects KeyListerGetter  // 该对象指向的就是Indexer，
    closed     bool               // 是否已经关闭的标记
    emitDeltaTypeReplaced bool    // 专为关闭设计的所
}
```

DeltaFIFO结构中比较难以理解的是knownObjects，它的类型为KeyListerGetter。其接口中的方法ListKeys和GetByKey也是Store接口中的方法，因此knownObjects能够被赋值为实现了Store的类型指针；同样地，由于Indexer继承了Store方法，因此knownObjects能够被赋值为实现了Indexer的类型指针。

DeltaFIFO.knownObjects.GetByKey就是执行的store.go中的GetByKey函数，用于获取Indexer中的对象键。

### Delta

```go
// k8s.io/client-go/tools/cache/delta_fifo.go

// DeltaType 是变化的类型（添加、删除等）
type DeltaType string

// 变化的类型定义
const (
 Added   DeltaType = "Added"     // 增加
 Updated DeltaType = "Updated"   // 更新
 Deleted DeltaType = "Deleted"   // 删除
 // 当遇到 watch 错误，不得不进行重新list时，就会触发 Replaced。我们不知道被替换的对象是否发生了变化。
 // 注意：以前版本的 DeltaFIFO 也会对 Replace 事件使用 Sync。所以只有当选项 EmitDeltaTypeReplaced 为真时才会触发 Replaced。
 Replaced DeltaType = "Replaced"
 // Sync 是针对周期性重新同步期间的合成事件
 Sync DeltaType = "Sync"          // 同步
)

// Delta 是 DeltaFIFO 存储的类型。它告诉你发生了什么变化，以及变化后对象的状态。除非变化是删除操作，否则你将得到对象被删除前的最终状态。
type Delta struct {
 Type   DeltaType
 Object interface{}
}
```

Delta 其实就是 Kubernetes 系统中带有变化类型的资源对象，比如我们现在添加了一个 Pod，那么这个 Delta 就是带有 Added 这个类型的 Pod，如果是删除了一个 Deployment，那么这个 Delta 就是带有 Deleted 类型的 Deployment，为什么要带上类型呢？因为我们需要根据不同的类型去执行不同的操作，增加、更新、删除的动作显然是不一样的。

### FIFO

```go
// k8s.io/client-go/tools/cache/fifo.go

type FIFO struct {
 lock sync.RWMutex
 cond sync.Cond

  // items 中的每一个 key 也在 queue 中
 items map[string]interface{}
 queue []string

 // 如果第一批 items 被 Replace() 插入或者先调用了 Deleta/Add/Update
  // 则 populated 为 true。
 populated bool
 // 第一次调用 Replace() 时插入的 items 数
 initialPopulationCount int

  // keyFunc 用于生成排队的 item 插入和检索的 key。
 keyFunc KeyFunc

  // 标识队列已关闭，以便在队列清空时控制循环可以退出。
 closed     bool
 closedLock sync.Mutex
}
```

FIFO 数据结构中定义了 items 和 queue 两个属性来保存队列中的数据，其中 queue 中存的是资源对象的 key 列表，而 items 是一个 map 类型，其 key 就是 queue 中保存的 key，value 值是真正的资源对象数据

## Reflector

Informer的核心就是Reflector

```go
type Reflector struct {
	name string
	expectedTypeName string
	expectedType reflect.Type             // 反射的类型，也就是要监控的对象类型，比如Pod
	expectedGVK *schema.GroupVersionKind
	store Store                           // 存储，就是DeltaFIFO
	listerWatcher ListerWatcher           // 核心，这个是用来从apiserver获取资源用的
	backoffManager wait.BackoffManager
	resyncPeriod time.Duration
	ShouldResync func() bool
	clock clock.Clock
	paginatedResult bool
	lastSyncResourceVersion string
	isLastSyncResourceVersionUnavailable bool
	lastSyncResourceVersionMutex sync.RWMutex
	WatchListPageSize int64
	watchErrorHandler WatchErrorHandler
}
```

Reflector有一个run函数来启动监控并处理监控事件，里面就是周期循环，直到 stopCh被关闭

```go
func (r *Reflector) Run(stopCh <-chan struct{}) {
	wait.BackoffUntil(func() {
		if err := r.ListAndWatch(stopCh); err != nil {
			r.watchErrorHandler(r, err)
		}
	}, r.backoffManager, true, stopCh)
}
```

里面关键的就是`ListAndWatch()`这个函数

<img src="https://img2018.cnblogs.com/blog/1334952/201907/1334952-20190708213617054-1429375536.png" alt="逻辑" style="zoom:50%;" />

```go
// ListAndWatch 函数首先列出所有的对象，并在调用的时候获得资源版本，然后使用该资源版本来进行 watch 操作。
// 如果 ListAndWatch 返回error， watch都不会初始化
// 下面代码会删除部分，只保留主干分支流程
func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
	var resourceVersion string

	options := metav1.ListOptions{ResourceVersion: r.relistResourceVersion()}

	if err := func() error {
    ······
		listCh := make(chan struct{}, 1)
		panicCh := make(chan interface{}, 1)
		go func() {
			······
      // 如果 listWatcher 支持，会尝试 chunks（分块）收集 List 列表数据
      // 如果不支持，第一个 List 列表请求将返回完整的响应数据。
			pager := pager.New(pager.SimplePageFunc(func(opts metav1.ListOptions) (runtime.Object, error) {
				return r.listerWatcher.List(opts)
			}))
			switch {
			case r.WatchListPageSize != 0:
				pager.PageSize = r.WatchListPageSize
			case r.paginatedResult:
			case options.ResourceVersion != "" && options.ResourceVersion != "0":
				pager.PageSize = 0
			}
      // list调用
			list, paginatedResult, err = pager.List(context.Background(), options)      
			if isExpiredError(err) || isTooLargeResourceVersionError(err) {
				r.setIsLastSyncResourceVersionUnavailable(true)
				list, paginatedResult, err = pager.List(context.Background(), metav1.ListOptions{ResourceVersion: r.relistResourceVersion()})
			}
			close(listCh)
			close(listCh)
		}()
		select {
		case <-stopCh:
			return nil
		case r := <-panicCh:
			panic(r)
		case <-listCh:
		}
		if err != nil {
			return fmt.Errorf("failed to list %v: %v", r.expectedTypeName, err)
		}
······
		r.setIsLastSyncResourceVersionUnavailable(false) // list was successful

		listMetaInterface, err := meta.ListAccessor(list)

		resourceVersion = listMetaInterface.GetResourceVersion()

		// 将资源对象列表中的资源对象和资源版本号存储在 Store 中
		if err := r.syncWith(items, resourceVersion); err != nil {
			return fmt.Errorf("unable to sync list result: %v", err)
		}

		r.setLastSyncResourceVersion(resourceVersion)
		return nil
	}(); err != nil {
		return err
	}

······
	for {
		select {
		case <-stopCh:
			return nil
		default:
		}
······
		start := r.clock.Now()
		w, err := r.listerWatcher.Watch(options)
		if err != nil {
			if utilnet.IsConnectionRefused(err) {
				time.Sleep(time.Second)
				continue
			}
			return err
		}
    
    // watchHandler实现从watch返回的chan中持续读取变化的资源，并转换为DeltaFIFO相应的调用
		if err := r.watchHandler(start, w, &resourceVersion, resyncerrc, stopCh); err != nil {
			if err != errorStopRequested {
				switch {
				case isExpiredError(err):
					klog.V(4).Infof("%s: watch of %v closed with: %v", r.name, r.expectedTypeName, err)
				default:
					klog.Warningf("%s: watch of %v ended with: %v", r.name, r.expectedTypeName, err)
				}
			}
			return nil
		}
	}
}
```

1. Reflector利用apiserver的client列举全量对象(版本为0以后的对象全部列举出来)
2. 将全量对象采用Replace()接口同步到DeltaFIFO中，并且更新资源的版本号，这个版本号后续会用到；
3. 开启一个协程定时执行resync，如果没有设置定时同步则不会执行，同步就是把全量对象以同步事件的方式通知出去；
4. 通过apiserver的client监控(watch)资源，监控的当前资源版本号以后的对象，因为之前的都已经获取到了；
5. 一旦有对象发生变化，那么就会根据变化的类型(新增、更新、删除)调用DeltaFIFO的相应接口，产生一个相应的对象Delta，同时更新当前资源的版本；

# WorkQueue

 indexer用于保存apiserver的资源信息，而**workqueue用于保存informer中的handler处理之后的数据**

WorkQueue支持三种队列：

- Interface：FIFO队列接口
- DelayingInterface：延迟队列接口，基于 Interface 接口封装，延迟一段时间再把元素存入队列
- RateLimitingInterface：限速队列接口，基于DelayingInterface封装，支持元素存入队列是进行限速
  - 令牌桶算法
  - 排队指数算法
  - 计数器算法
  - 混合模式



