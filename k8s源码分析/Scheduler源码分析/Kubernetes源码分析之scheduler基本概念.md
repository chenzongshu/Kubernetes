# 调度器

Scheduler要想实现调度一个Pod的全流程，那么必须有调度队列、调度缓存、调度框架、调度插件、调度算法]等模块的支持。所以，Scheduler的成员变量势必会包含这些类型

```go
// pkg/scheduler/scheduler.go
// Scheduler监视未调度的Pod并尝试找到适合的Node，并将绑定信息写回到apiserver。
type Scheduler struct {
	// 调度缓存，用来缓存所有的Node状态
	// 了解过调度算法的读者应该知道，调度缓存也会被调度算法使用，用来每次调度前更新快照
	SchedulerCache internalcache.Cache
    // 调度算法
	Algorithm core.ScheduleAlgorithm

	// NextPod()获取下一个需要调度的Pod(如果没有则阻塞当前协程)，是不是想到了调度队列(SchedulingQueue)了？
	NextPod func() *framework.QueuedPodInfo

	// 当调度一个Pod出现错误的时候调用Error()函数，是Scheduler使用者注入的错误函数，相当于回调。
	Error func(*framework.QueuedPodInfo, error)

	// 关闭Scheduler的信号
	StopEverything <-chan struct{}

	// 调度队列，用来缓存等待调度的Pod
	SchedulingQueue internalqueue.SchedulingQueue

	// 别看名字是Profile，其实是Framework，因为Profile和Framework是一对一的，Profile只是Framework的配置而已。
	Profiles profile.Map

	// apiserver的客户端，主要用来向apiserver执行写操作，比如写Pod的绑定信息
	client clientset.Interface
}
```



# 调度队列

`PriorityQueue` 优先队列实现了调度队列（SchedulingQueue），优先队列的头部是优先级最高的挂起（Pending）Pod。优先队列有三个子队列：一个子队列包含准备好调度的Pod，称为activeQ（是堆类型）；另一个队列包含已尝试并且确定为不可调度的Pod，称为unschedulableQ（UnschedulablePodsMap）；第三个队列是backoffQ,包含从unschedulableQ移出的Pod，退避完成后的Pod将其移到activeQ，

```go
//  pkg/scheduler/internal/queue/scheduling_queue.go
type PriorityQueue struct {
	framework.PodNominator
	stop  chan struct{}
	clock util.Clock

  // Pod的初始退避时间，默认值是1秒，可配置.当Pod调度失败后第一次的退避时间就是podInitialBackoffDuration。
  // 第二次尝试调度失败后的退避时间就是podInitialBackoffDuration*2，第三次是podInitialBackoffDuration*4以此类推
	podInitialBackoffDuration time.Duration
	// Pod的最大退避时间，默认值是10秒，可配置。因为退避时间随着尝试次数指数增长，这个变量就是退避时间的上限值
	podMaxBackoffDuration time.Duration
	lock sync.RWMutex
	cond sync.Cond
  // 下面3个就是关键队列，heap类型
	activeQ *heap.Heap
	podBackoffQ *heap.Heap
	unschedulableQ *UnschedulablePodsMap
  // 调度周期，在Pop()中自增，SchedulingCycle()获取这个值
	schedulingCycle int64
  // 这个变量以调度周期为tick，记录上一次调用MoveAllToActiveOrBackoffQueue()的调度周期。
  // 因为调用AddUnschedulableIfNotPresent()接口的时候需要提供调度周期，当调度周期小于moveRequestCycle时，
  // 说明该不可调度Pod应该也被挪走，只是在调用接口的时候抢锁晚于MoveAllToActiveOrBackoffQueue()。
	moveRequestCycle int64

	clusterEventMap map[framework.ClusterEvent]sets.String

  // 让Pop()在等待一个item的时候退出它的控制循环
	closed bool
	nsLister listersv1.NamespaceLister
}
```

1. 虽然名字上叫做调度队列，但是实际上都是按照map组织的，其实map也是一种队列，只是序是按照对象的key排列的（需要注意的是golang中的map是无序的），此处需要与数据结构中介绍的queue区分开。在调度过程中会频繁的判断对象是否已经存在（有一个高大上的名字叫幂等）、是否不存在，从一个子队列挪到另一个子队列，伴随添加、删除、更新等操作，map相比于其他数据结构是有优势的。
2. 调度队列虽然用map组织了Pod，同时用Pod的key的slice对Pod进行排序，这样更有队列的样子了。
3. 调度队列分为三个自队列：
   1. activeQ：即ready队列，里面都是准备调度的Pod，新添加的Pod都会被放到该队列，从调度队列中弹出需要调度的Pod也是从该队列获取；
   2. unschedulableQ：不可调度队列，里面有各种原因造成无法被调度Pod，至于有哪些原因，笔者在分析调度器的文章中会说明；
   3. backoffQ：退避队列，里面都是需要可以调度但是需要退避一段时间后才能调度的Pod；
4. 新创建的Pod通过Add()接口将Pod放入activeQ。 调度器每调度一个Pod都需要调用Pop()接口获取一个Pod，如果调度失败需要调用AddUnschedulableIfNotPresent()把Pod返回到调度队列中，如果调度成功了调用AssignedPodAdded()把依赖的Pod从unschedulableQ挪到activeQ或者backoffQ。
5. 调度队列后台每30秒刷一次unschedulableQ，如果入队列已经超过unschedulableQTimeInterval(60秒)，则将Pod从unschedulableQ移到activeQ，当然，如果还在退避时间内，只能放入backoffQ。
6. 调度队列后台每1秒刷一次backoffQ，把退避完成的所有Pod从backoffQ移到activeQ。
7. 虽然调度队列的实现是优先队列，但是基本没看到优先级相关的内容，是因为创建优先队列的时候需要传入lessFunc，该函数决定了activeQ的顺序，也就是调度优先级。所以调度优先级是可自定义的。
8. Add()和AddUnschedulableIfNotPresent()会设置Pod入队列时间戳，这个时间戳会用来计算Pod的退避时间、不可调度时间。

## Heap

里提到的堆与排序有关，总所周知，golang的快排采用的是堆排序。所以不要与内存管理中"堆"概念混淆。

在kube-scheduler中，堆既有map的高效检索能力，有具备slice的顺序，这对于调度队列来说非常关键。因为调度对象随时可能添加、删除、更新，需要有高效的检索能力快速找到对象，map非常适合。但是golang中的map是无序的，访问map还有一定的随机性（每次range的第一个对象是随机的）。而调度经常会因为优先级、时间、依赖等原因需要对对象排序，slice非常合适，所以就有了堆这个类型。

```go

// Heap定义
type Heap struct {
    // Heap继承了data
    data *data
    // 监控相关
    metricRecorder metrics.MetricRecorder
}
// data是Heap核心实现
type data struct {
    // 用map管理所有对象
    items map[string]*heapItem
    // 在用slice管理所有对象的key，对象的key就是Pod的namespace+name，这是在kubernetes下唯一键
    queue []string
 
    // 获取对象key的函数，毕竟对象是interface{}类型的
    keyFunc KeyFunc
    // 判断两个对象哪个小的函数，了解sort.Sort()的读者此时应该能够猜到它是用来排序的。
    // 所以可以推断出来queue中key是根据lessFunc排过序的，而lessFunc又是传入进来的，
    // 这让Heap中对象的序可定制，这个非常有价值，非常有用
    lessFunc lessFunc
}
// 堆存储对象的定义
type heapItem struct {
    obj   interface{} // 对象，更准确的说应该是指针，应用在调度队列中就是*QueuedPodInfo
    index int         // Heap.queue的索引
}
```

# Cache

为什么要cache？client-go已经提供了cache能力，kube-scheduler增加一层cache的目的是什么呢？答案很简单，为了调度。Cache不仅缓存了Pod和Node信息，更关键的是聚合了调度结果，让调度变得更容易

1. Cache缓存了Pod和Node信息，并且Node信息聚合了运行在该Node上所有Pod的资源量和镜像信息；Node有虚实之分，已删除的Node，Cache不会立刻删除它，而是继续维护一个虚的Node，直到Node上的Pod清零后才会被删除；但是nodeTree中维护的是实际的Node，调度使用nodeTree就可以避免将Pod调度到虚Node上；
2. kube-scheduler利用client-go监控(watch)Pod和Node状态，当有事件发生时调用Cache的AddPod，RemovePod，UpdatePod，AddNode，RemoveNode，UpdateNode更新Cache中Pod和Node的状态，这样kube-scheduler开始新一轮调度的时候可以获得最新的状态；
3. kube-scheduler每一轮调度都会调用UpdateSnapshot更新本地(局部变量)的Node状态，因为Cache中的Node按照最近更新排序，只需要将Cache中Node.Generation大于kube-scheduler本地的快照generation的Node更新到snapshot中即可，这样可以避免大量不必要的拷贝；
4. kube-scheduler找到合适的Node调度Pod后，需要调用Cache.AssumePod假定Pod已调度，然后启动协程异步Bind Pod到Node上，当Pod完成Bind后，调用Cache.FinishBinding通知Cache；
5. kube-scheudler调用Cache.AssumePod后续的所有造作一旦有错误就会调用Cache.ForgetPod删除假定的Pod，释放资源；
6. 完成Bind的Pod默认超时为30秒，Cache有一个协程定时(1秒)清理超时的Bind超时的Pod，如果超时依然没有收到Pod确认消息(调用AddPod)，则将删除超时的Pod，进而释放出Cache.AssumePod占用的资源;
7. Cache的核心功能就是统计Node的调度状态(比如累加Pod的资源量、统计镜像)，然后以镜像的形式输出给kube-scheduler，kube-scheduler从调度队列(SchedulingQueue)中取出等待调度的Pod，根据镜像计算最合适的Node；

## cache抽象

```go
// pkg/scheduler/internal/cache/interface.go
type Cache interface {
	// 获取node的数量，用于单元测试使用.
	NodeCount() int
  // 获取Pod的数量，用于单元测试使用
	PodCount() (int, error)

	// 此处需要给出一个概念：假定Pod,就是将Pod假定调度到指定的Node,但还没有Bind完成。为什么要这么设计？
  // 因为kube-scheduler是通过异步的方式实现Bind，在Bind完成前，调度器还要调度新的Pod，
  // 此时就先假定Pod调度完成了。至于什么是Bind？为什么Bind？
  // 怎么Bind?此处简单理解为：需要将Pod的调度结果写入etcd,持久化调度结果，所以也是相对比较耗时的操作。
  // AssumePod会将Pod的资源需求累加到Node上，这样kube-scheduler在调度其他Pod的时候,就不会占用这部分资源
	AssumePod(pod *v1.Pod) error

  // 前面提到了，Bind是一个异步过程，当Bind完成后需要调用这个接口通知Cache，
  // 如果完成Bind的Pod长时间没有被确认(确认方法是AddPod)，那么Cache就会清理掉假定过期的Pod。
	FinishBinding(pod *v1.Pod) error

	// 删除假定的Pod，kube-scheduler在调用AssumePod后如果遇到其他错误，就需要调用这个接口
	ForgetPod(pod *v1.Pod) error

	// 添加Pod既确认了假定的Pod，也会将假定过期的Pod重新添加回来。
	AddPod(pod *v1.Pod) error

	// 更新Pod，其实就是删除再添加
	UpdatePod(oldPod, newPod *v1.Pod) error
	RemovePod(pod *v1.Pod) error
	GetPod(pod *v1.Pod) (*v1.Pod, error)

	// 判断Pod是否假定调度
	IsAssumedPod(pod *v1.Pod) (bool, error)

	// 添加Node的全部信息
	AddNode(node *v1.Node) error
	UpdateNode(oldNode, newNode *v1.Node) error
	RemoveNode(node *v1.Node) error

  // 其实就是产生Cache的快照并输出到nodeSnapshot中，那为什么是更新呢？
  // 因为快照比较大，产生快照也是一个比较重的任务，如果能够基于上次快照把增量的部分更新到上一次快照中，
  // 就会变得没那么重了，这就是接口名字是更新快照的原因。文章后面会重点分析这个函数，
  // 因为其他接口非常简单，理解了这个接口基本上就理解了Cache的精髓所在。
	UpdateSnapshot(nodeSnapshot *Snapshot) error

	// Dump会快照Cache，用于调试使用，不是重点
	Dump() *Dump
}
```

从Cache的接口设计上可以看出，Cache只缓存了Pod和Node信息，而Pod和Node信息存储在etcd中(可以通过kubectl增删改查)，所以可以确认Cache缓存了etcd中的Pod和Node信息。

- `AssumePod`：当kube-scheduler找到最优的Node调度Pod的时候会调用AssumePod假定Pod调度，在通过另一个协程异步Bind。假定其实就是预先占住资源，kube-scheduler调度下一个Pod的时候不会把这部分资源抢走，直到收到确认消息AddPod确认调度成功，亦或是Bind失败ForgetPod取消假定调度
- `ForgetPod`：假定Pod预先占用了一些资源，如果之后的操作(比如Bind)有什么错误，就需要取消假定调度，释放出资源
- `FinishBinding`：当假定Pod绑定完成后，需要调用FinishBinding通知Cache开始计时，直到假定Pod过期如果依然没有收到AddPod的请求，则将过期假定Pod删除
- `AddPod`：当Pod Bind成功，kube-scheduler会收到消息，然后调用AddPod确认调度结果。
- `RemovePod`：kube-scheduler收到删除Pod的请求，如果Pod在Cache中，就需要调用RemovePod
- `AddNode`：有新的Node添加到集群，kube-scheduler调用该接口通知Cache

## 快照

快照是对Cache某一时刻的复制，随着时间的推移，Cache的状态在持续更新，kube-scheduler在调度一个Pod的时候需要获取Cache的快照。相比于直接访问Cache，快照可以解决如下几个问题：

快照不会再有任何变化，可以理解为只读，那么访问快照不需要加锁保证保证原子性； 快照和Cache让读写分离，可以避免大范围的锁造成Cache访问性能下降； 虽然快照的状态从创建开始就落后于(因为Cache可能随时都会更新)Cache，但是对于kube-scheduler调度一个Pod来说是没问题的

```go
// pkg/scheduler/internal/cache/snapshot.go
// 从定义上看，快照只有Node信息，没有Pod信息，其实Node信息中已经有Pod信息了，这个在NodeInfo中已经说明了
type Snapshot struct {
    // nodeInfoMap用于根据Node的key(NS+Name)快速查找Node
    nodeInfoMap map[string]*framework.NodeInfo
    // nodeInfoList是Cache中Node全集列表（不包含已删除的Node），按照nodeTree排序.
    nodeInfoList []*framework.NodeInfo
    // 只要Node上有任何Pod声明了亲和性，那么该Node就要放入havePodsWithAffinityNodeInfoList。
    // 为什么要有这个变量？当然是为了调度，比如PodA需要和PodB调度在一个Node上。
    havePodsWithAffinityNodeInfoList []*framework.NodeInfo
    // havePodsWithRequiredAntiAffinityNodeInfoList和havePodsWithAffinityNodeInfoList相似，
    // 只是Pod声明了反亲和，比如PodA不能和PodB调度在一个Node上
    havePodsWithRequiredAntiAffinityNodeInfoList []*framework.NodeInfo
    // generation是所有NodeInfo.Generation的最大值，因为所有NodeInfo.Generation都源于一个全局的Generation变量，
    // 那么Cache中的NodeInfo.Gerneraion大于该值的就是在快照产生后更新过的Node。
    // kube-scheduler调用Cache.UpdateSnapshot的时候只需要更新快照之后有变化的Node即可
    generation                                   int64
}
```

## cache实现

```go
// pkg/scheduler/internal/cache/cache.go
// schedulerCache实现了Cache接口
type schedulerCache struct {
    // 这个比较好理解，用来通知schedulerCache停止的chan，说明schedulerCache有自己的协程
    stop   <-chan struct{}
    // 假定Pod一旦完成绑定，就要在指定的时间内确认，否则就会超时，ttl就是指定的过期时间，默认30秒
    ttl    time.Duration
    // 定时清理“假定过期”的Pod，period就是定时周期，默认是1秒钟
    // 前面提到了schedulerCache有自己的协程，就是定时清理超时的假定Pod.
    period time.Duration

    // 锁，说明schedulerCache利用互斥锁实现协程安全，而不是用chan与其他协程交互。
    // 这一点实现和SchedulingQueue是一样的。
    mu sync.RWMutex
    // 假定Pod集合，map的key与podStates相同，都是Pod的NS+NAME，值为true就是假定Pod
    // 其实assumedPods的值没有false的可能，感觉assumedPods用set类型(map[string]struct{}{})更合适
    assumedPods map[string]bool
    // 所有的Pod，此处用的是podState，后面有说明，与SchedulingQueue中提到的QueuedPodInfo类似，
    // podState继承了Pod的API定义，增加了Cache需要的属性
    podStates map[string]*podState
    // 所有的Node，键是Node.Name，值是nodeInfoListItem，后面会有说明，只需要知道map类型就可以了
    nodes     map[string]*nodeInfoListItem
    // 所有的Node再通过双向链表连接起来
    headNode *nodeInfoListItem
    // 节点按照zone组织成树状，前面提到用nodeTree中Node的名字再到nodes中就可以查找到NodeInfo.
    nodeTree *nodeTree
    // 镜像状态，Cache还统计了镜像的信息
    imageStates map[string]*imageState
}
 
// podState与继承了Pod的API类型定义，同时扩展了schedulerCache需要的属性.
type podState struct {
    pod *v1.Pod
    // 假定Pod的超时截止时间，用于判断假定Pod是否过期。
    deadline *time.Time
    // 调用Cache.AssumePod的假定Pod不是所有的都需要判断是否过期，因为有些假定Pod可能还在Binding
    // bindingFinished就是用于标记已经Bind完成的Pod，然后开始计时，计时的方法就是设置deadline
    // 还记得Cache.FinishBinding接口么？就是用来设置bindingFinished和deadline的，后面代码会有解析
    bindingFinished bool
}
 
// nodeInfoListItem定义了nodeInfoList双向链表的item,nodeInfoList的实现非常简单，不多解释。
type nodeInfoListItem struct {
    info *framework.NodeInfo
    next *nodeInfoListItem
    prev *nodeInfoListItem
}
```

## UpdateSnapshot

Cache存在的核心目的就是给kube-scheduler提供Node镜像，让kube-scheduler根据Node的状态调度新的Pod。而Cache中的Pod是为了计算Node的资源状态存在的，毕竟二者在etcd中是两个路径

```go
// UpdateSnapshot更新的是参数nodeSnapshot，不是更新Cache.
// 也就是Cache需要找到当前与nodeSnapshot的差异，然后更新它，这样nodeSnapshot就与Cache状态一致了
// 至少从函数执行完毕后是一致的。
func (cache *schedulerCache) UpdateSnapshot(nodeSnapshot *Snapshot) error {
    cache.mu.Lock()
    defer cache.mu.Unlock()
    balancedVolumesEnabled := utilfeature.DefaultFeatureGate.Enabled(features.BalanceAttachedNodeVolumes)
 
    // 获取nodeSnapshot的版本，笔者习惯叫版本，其实就是版本的概念。
    // 此处需要多说一点：kube-scheudler为Node定义了全局的generation变量，每个Node状态变化都会造成generation+=1然后赋值给该Node
    // nodeSnapshot.generation就是最新NodeInfo.Generation，就是表头的那个NodeInfo。
    snapshotGeneration := nodeSnapshot.generation
 
    // 介绍Snapshot的时候提到了，快照中有三个列表，分别是全量、亲和性和反亲和性列表
    // 全量列表在没有Node添加或者删除的时候，是不需要更新的
    updateAllLists := false
    // 当有Node的亲和性状态发生了变化(以前没有任何Pod有亲和性声明现在有了，抑或反过来)，
    // 则需要更新快照中的亲和性列表
    updateNodesHavePodsWithAffinity := false
    // 同上
    updateNodesHavePodsWithRequiredAntiAffinity := false
 
    // 遍历Node列表，为什么不遍历Node的map？因为Node列表是按照Generation排序的
    // 只要找到大于nodeSnapshot.generation的所有Node然后把他们更新到nodeSnapshot中就可以了
    for node := cache.headNode; node != nil; node = node.next {
        // 说明Node的状态已经在nodeSnapshot中了，因为但凡Node有任何更新，那么NodeInfo.Generation 
        // 肯定会大于snapshotGeneration，同时该Node后面的所有Node也不用在遍历了，因为他们的版本更低
        if node.info.Generation <= snapshotGeneration {
            break
        }
        // 先忽略
        if balancedVolumesEnabled && node.info.TransientInfo != nil {
            // Transient scheduler info is reset here.
            node.info.TransientInfo.ResetTransientSchedulerInfo()
        }
        // node.info.Node()获取*v1.Node，前文说了，如果Node被删除，那么该值就是为nil
        // 所以只有未被删除的Node才会被更新到nodeSnapshot，因为快照中的全量Node列表是按照nodeTree排序的
        // 而nodeTree都是真实的node
        if np := node.info.Node(); np != nil {
            // 如果nodeSnapshot中没有该Node，则在nodeSnapshot中创建Node，并标记更新全量列表，因为创建了新的Node
            existing, ok := nodeSnapshot.nodeInfoMap[np.Name]
            if !ok {
                updateAllLists = true
                existing = &framework.NodeInfo{}
                nodeSnapshot.nodeInfoMap[np.Name] = existing
            }
            // 克隆NodeInfo，这个比较好理解，肯定不能简单的把指针设置过去，这样会造成多协程读写同一个对象
            // 因为克隆操作比较重，所以能少做就少做，这也是利用Generation实现增量更新的原因
            clone := node.info.Clone()
            // 如果Pod以前或者现在有任何亲和性声明，则需要更新nodeSnapshot中的亲和性列表
            if (len(existing.PodsWithAffinity) > 0) != (len(clone.PodsWithAffinity) > 0) {
                updateNodesHavePodsWithAffinity = true
            }
            // 同上，需要更新非亲和性列表
            if (len(existing.PodsWithRequiredAntiAffinity) > 0) != (len(clone.PodsWithRequiredAntiAffinity) > 0) {
                updateNodesHavePodsWithRequiredAntiAffinity = true
            }
            // 将NodeInfo的拷贝更新到nodeSnapshot中
            *existing = *clone
        }
    }
    // Cache的表头Node的版本是最新的，所以也就代表了此时更新镜像后镜像的版本了
    if cache.headNode != nil {
        nodeSnapshot.generation = cache.headNode.info.Generation
    }
 
    // 如果nodeSnapshot中node的数量大于nodeTree中的数量，说明有node被删除
    // 所以要从快照的nodeInfoMap中删除已删除的Node，同时标记需要更新node的全量列表
    if len(nodeSnapshot.nodeInfoMap) > cache.nodeTree.numNodes {
        cache.removeDeletedNodesFromSnapshot(nodeSnapshot)
        updateAllLists = true
    }
 
    // 如果需要更新Node的全量或者亲和性或者反亲和性列表，则更新nodeSnapshot中的Node列表
    if updateAllLists || updateNodesHavePodsWithAffinity || updateNodesHavePodsWithRequiredAntiAffinity {
        cache.updateNodeInfoSnapshotList(nodeSnapshot, updateAllLists)
    }
 
    // 如果此时nodeSnapshot的node列表与nodeTree的数量还不一致，需要再做一次node全列表更新
    // 此处应该是一个保险操作，理论上不会发生，谁知道会不会有Bug发生呢？多一些容错没有坏处
    if len(nodeSnapshot.nodeInfoList) != cache.nodeTree.numNodes {
        ······
        cache.updateNodeInfoSnapshotList(nodeSnapshot, true)
        return fmt.Errorf(errMsg)
    }
    return nil
}
 
// 先思考一个问题：为什么有Node添加或者删除需要更新快照中的全量列表？如果是Node删除了，
// 需要找到Node在全量列表中的位置，然后删除它，最悲观的复杂度就是遍历一遍列表，然后再挪动它后面的Node
// 因为快照的Node列表是用slice实现，所以一旦快照中Node列表有任何更新，复杂度都是Node的数量。
// 那如果是有新的Node添加呢？并不知道应该插在哪里，所以重新创建一次全量列表最为简单有效。
// 亲和性和反亲和性列表道理也是一样的。
func (cache *schedulerCache) updateNodeInfoSnapshotList(snapshot *Snapshot, updateAll bool) {
    // 快照创建亲和性和反亲和性列表
    snapshot.havePodsWithAffinityNodeInfoList = make([]*framework.NodeInfo, 0, cache.nodeTree.numNodes)
    snapshot.havePodsWithRequiredAntiAffinityNodeInfoList = make([]*framework.NodeInfo, 0, cache.nodeTree.numNodes)
    // 如果更新全量列表
    if updateAll {
        // 创建快照全量列表
        snapshot.nodeInfoList = make([]*framework.NodeInfo, 0, cache.nodeTree.numNodes)
        nodesList, err := cache.nodeTree.list()
        if err != nil {
            klog.Error(err)
        }
        // 遍历nodeTree的Node
        for _, nodeName := range nodesList {
            // 理论上快照的nodeInfoMap与nodeTree的状态是一致，此处做了判断用来检测BUG，下面的错误日志也是这么写的
            if nodeInfo := snapshot.nodeInfoMap[nodeName]; nodeInfo != nil {
                // 追加全量、亲和性(按需)、反亲和性列表(按需)
                snapshot.nodeInfoList = append(snapshot.nodeInfoList, nodeInfo)
                if len(nodeInfo.PodsWithAffinity) > 0 {
                    snapshot.havePodsWithAffinityNodeInfoList = append(snapshot.havePodsWithAffinityNodeInfoList, nodeInfo)
                }
                if len(nodeInfo.PodsWithRequiredAntiAffinity) > 0 {
                    snapshot.havePodsWithRequiredAntiAffinityNodeInfoList = append(snapshot.havePodsWithRequiredAntiAffinityNodeInfoList, nodeInfo)
                }
            } else {
                klog.Errorf("node %q exist in nodeTree but not in NodeInfoMap, this should not happen.", nodeName)
            }
        }
    } else {
        // 如果更新全量列表，只需要遍历快照中的全量列表就可以了
        for _, nodeInfo := range snapshot.nodeInfoList {
            // 按需追加亲和性和反亲和性列表
            if len(nodeInfo.PodsWithAffinity) > 0 {
                snapshot.havePodsWithAffinityNodeInfoList = append(snapshot.havePodsWithAffinityNodeInfoList, nodeInfo)
            }
            if len(nodeInfo.PodsWithRequiredAntiAffinity) > 0 {
                snapshot.havePodsWithRequiredAntiAffinityNodeInfoList = append(snapshot.havePodsWithRequiredAntiAffinityNodeInfoList, nodeInfo)
            }
        }
    }
}
```

# 调度框架

1. Profile和Framework是一一对应的，kube-scheduler会为一个Profile创建一个Framework，可以通过设置Pod.Spec.SchedulerName选择Framework执行调度；
2. Framework将调度一个Pod分为调度周期和绑定周期，每个Pod的调度周期是串行的，但是绑定周期可能是并行的；
3. Framework定义了扩展点的概念，并且为每个扩展定定义了接口，即XxxPlugin；
4. Framework为每个扩展点定义了一个入口，RunXxxPlugins，Framework会按照Profile配置的插件顺序依次调用插件；
5. Framework插件定义了句柄和抢占句柄，为插件实现特定的功能提供接口；

```go
// pkg/scheduler/framework/interface.go
type Framework interface {
    // 调度框架句柄，构建调度插件时使用
    Handle
    // 获取调度队列排序需要的函数，其实就是QueueSortPlugin.Less()
    QueueSortFunc() LessFunc

    // 有没有发现和PluginsRunner很像？应该说PluginsRunner是Framework的子集，也可以理解为Framework的实现也是PluginsRunner的实现。
    //Framework这些接口主要是为了每个扩展点抽象一个接口，因为每个扩展点有多个插件，该接口的实现负责按照配置顺序调用插件。
    // 大部分扩展点如果有一个插件返回不成功，则后续的插件就不会被调用了，所以同一个扩展点的插件间大多是“逻辑与”的关系。
    RunPreFilterPlugins(ctx context.Context, state *CycleState, pod *v1.Pod) *Status
    RunFilterPlugins(ctx context.Context, state *CycleState, pod *v1.Pod, nodeInfo *NodeInfo) PluginToStatus
    RunPostFilterPlugins(ctx context.Context, state *CycleState, pod *v1.Pod, filteredNodeStatusMap NodeToStatusMap) (*PostFilterResult, *Status)
    RunPreFilterExtensionAddPod(ctx context.Context, state *CycleState, podToSchedule *v1.Pod, podToAdd *v1.Pod, nodeInfo *NodeInfo) *Status
    RunPreFilterExtensionRemovePod(ctx context.Context, state *CycleState, podToSchedule *v1.Pod, podToAdd *v1.Pod, nodeInfo *NodeInfo) *Status
    RunPreScorePlugins(ctx context.Context, state *CycleState, pod *v1.Pod, nodes []*v1.Node) *Status
    RunScorePlugins(ctx context.Context, state *CycleState, pod *v1.Pod, nodes []*v1.Node) (PluginToNodeScores, *Status)
    RunPreBindPlugins(ctx context.Context, state *CycleState, pod *v1.Pod, nodeName string) *Status
    RunPostBindPlugins(ctx context.Context, state *CycleState, pod *v1.Pod, nodeName string)
    RunReservePluginsReserve(ctx context.Context, state *CycleState, pod *v1.Pod, nodeName string) *Status
    RunReservePluginsUnreserve(ctx context.Context, state *CycleState, pod *v1.Pod, nodeName string)
    RunPermitPlugins(ctx context.Context, state *CycleState, pod *v1.Pod, nodeName string) *Status
    RunBindPlugins(ctx context.Context, state *CycleState, pod *v1.Pod, nodeName string) *Status

    // 如果Pod正在等待（PermitPlugin返回等待状态），则会被阻塞直到Pod被拒绝或者批准。
    WaitOnPermit(ctx context.Context, pod *v1.Pod) *Status
    
    // 用来检测Framework是否有FilterPlugin、PostFilterPlugin、ScorePlugin，没难度，不解释
    HasFilterPlugins() bool
    HasPostFilterPlugins() bool
    HasScorePlugins() bool

    // 列举插件，其中：
    // 1. key是扩展点的名字，比如PreFilterPlugin对应扩展点的名字就是PreFilter，规则就是将插件类型名字去掉Plugin；
    // 2. value是扩展点所有插件配置，类型是slice
    ListPlugins() map[string][]config.Plugin

    // 此处需要引入SchedulingProfile概念，SchedulingProfile允许配置调度框架的扩展点。
    // SchedulingProfile包括使能/禁止每个扩展点的插件，每个插件的权重(仅在Score扩展点有效)以及插件参数；
    // 这个接口是用来获取SchedulingProfile的名字，因为v1.Pod.Spec.SchedulerName就是指定SchedulingProfile名字来选择Framework。
    ProfileName() string
}
```

- Framework是根据SchedulingProfile创建的，是一对一的关系，Framework中插件的使能/禁止、权重以及参数都来自SchedulingProfile;

- Framework按照调度框架设计图中的扩展点依次提供了运行所有插件的RunXxxPlugins()接口，扩展点插件的调用顺序也是通过SchedulingProfile配置的；

- 一个Framework就是一个调度器，配置好的调度插件就是调度算法，kube-scheduler按照固定的流程调用Framework接口即可，这样kube-scheduler就相对简单且易于维护；

## Handle

调度框架的句柄Handle就是调度框架的指针，句柄是提供给调度插件(在插件注册表章节有介绍)使用的。因为调度插件的接口参数只包含PodInfo、NodeInfo、以及CycleState，如果调度插件需要访问其他对象只能通过调度框架句柄。比如绑定插件需要的Clientset、抢占调度插件需要的SharedIndexInformer、批准插件需要的等待Pod访问接口等等

```go
type Handle interface {
    // SharedLister是基于快照的Lister，这个快照就是在调度Cache文章中介绍的NodeInfo的快照。
    // 此处不扩展对SharedLister进行说明，只要有调度Cache的基础就可以了，无非是NodeInfo快照基础上抽象新的接口。
    // 在调度周期开始时获取快照，保持不变，直到Pod完成'Permit'扩展点为止，在绑定阶段是不保证快照不变的。
    // 因此绑定周期中插件不应使用它，否则会发生并发度/写错误的可能性，它们应该改用调度Cache。
    SnapshotSharedLister() SharedLister

    // 此处需要说明一下WaitingPod，就是在'Permit'扩展点返回等待的Pod，需要Handle的实现提供访问这些Pod的接口。
    // 遍历等待中的Pod。
    IterateOverWaitingPods(callback func(WaitingPod))

    // 通过UID获取等待中的Pod
    GetWaitingPod(uid types.UID) WaitingPod

    // 拒绝等待中的Pod
    RejectWaitingPod(uid types.UID)

    // 获取Clientset，用于创建、更新API对象，典型的应用就是绑定Pod。
    ClientSet() clientset.Interface

    // 用于记录调度过程中的各种事件
    EventRecorder() events.EventRecorder

    // 当调度Cache和队列中的信息无法满足插件需求时，可以利用SharedIndexInformer获取指定的API对象。
    // 比如抢占调度插件需要获取所有运行中的Pod。
    SharedInformerFactory() informers.SharedInformerFactory

    // 获取抢占句柄，只开放给实现抢占调度插件的句柄，用来协助实现抢占调度，详情见后面章节注释。
    PreemptHandle() PreemptHandle
}
```

句柄存在的意义就是为插件提供服务接口，协助插件实现其功能，且无需提供调度框架的实现。虽然句柄就是调度框架指针，但是interface的魅力就在于使用者只需要关注接口的定义，并不关心接口的实现，这也是一种解耦方法。

## Framework实现

为什么称之为框架？因为他是固定的，差异在于配置导致插件的组合不同。也就是说，即便有多个SchedulingProfile，但是只有一个Framework实现，无非Framework成员变量引用的插件不同而已。来看看Framework实现代码

```go
// pkg/scheduler/framework/runtime/framework.go
type frameworkImpl struct {
    // 调度插件注册表，这个非常有用，frameworkImpl的构造函数需要用它来创建配置的插件。
    registry              Registry
    // 这个是为实现Handle.SnapshotSharedLister()接口准备的，是创建frameworkImpl时传入的，不是frameworkImpl自己创建的。
    // 至于framework.SharedLister类型会在调度器的文章中介绍，此处暂时不用关心，因为不影响阅读。
    snapshotSharedLister  framework.SharedLister
    // 这是为实现Handle.GetWaitingPod/RejectWaitingPod/IterateOverWaitingPods()接口准备的。
    // 关于waitingPodsMap请参看https://github.com/jindezgm/k8s-src-analysis/blob/master/kube-scheduler/WaitingPods.md
    waitingPods           *waitingPodsMap
    // 插件名字与权重的映射，用来根据插件名字获取权重，为什么要有这个变量呢？因为插件的权重是配置的，对于每个Framework都不同。
    // 所以pluginNameToWeightMap是在构造函数中根据SchedulingProfile生成的。
    pluginNameToWeightMap map[string]int
    // 所有扩展点的插件，为实现Framework.RunXxxPlugins()准备的，不难理解，就是每个扩展点遍历插件执行就可以了。
    // 下面所有插件都是在构造函数中根据SchedulingProfile生成的。
    queueSortPlugins      []framework.QueueSortPlugin
    preFilterPlugins      []framework.PreFilterPlugin
    filterPlugins         []framework.FilterPlugin
    postFilterPlugins     []framework.PostFilterPlugin
    preScorePlugins       []framework.PreScorePlugin
    scorePlugins          []framework.ScorePlugin
    reservePlugins        []framework.ReservePlugin
    preBindPlugins        []framework.PreBindPlugin
    bindPlugins           []framework.BindPlugin
    postBindPlugins       []framework.PostBindPlugin
    permitPlugins         []framework.PermitPlugin

    // 这些是为实现Handle.Clientset/EventHandler/SharedInformerFactory()准备的，因为Framework继承了Handle。
    // 这些成员变量是同构构造函数的参数出入的，不是frameworkImpl自己创建的。
    clientSet       clientset.Interface
    eventRecorder   events.EventRecorder
    informerFactory informers.SharedInformerFactory

    metricsRecorder *metricsRecorder
    // 为实现Handle.ProfileName()准备的。
    profileName     string

    // 为实现Framework.PreemptHandle()准备的
    preemptHandle framework.PreemptHandle

    // 如果为true，RunFilterPlugins第一次失败后不返回，应该积累有失败的状态。
    runAllFilters bool
}

```

Framework的接口实现见其构造函数

```go
// NewFramework()是frameworkImpl的构造函数，前三个参数：
// 1. r: 插件注册表，可以根据插件的名字创建插件；
// 2. plugins: 插件开关配置，就是使能/禁止哪些插件；
// 3. args: 插件自定义参数，用来创建插件；
// 4. opts: 构造frameworkImpl的选项参数，frameworkImpl有不少成员变量是通过opts传入进来的；
func NewFramework(r Registry, plugins *config.Plugins, args []config.PluginConfig, opts ...Option) (framework.Framework, error) {
    // 应用所有选项，Option这种玩法不稀奇了，golang有太多项目采用类似的方法
    options := defaultFrameworkOptions
    for _, opt := range opts {
        opt(&options)
    }

    // 可以看出来frameworkImpl很多成员变量来自opts
    f := &frameworkImpl{
        registry:              r,
        snapshotSharedLister:  options.snapshotSharedLister,
        pluginNameToWeightMap: make(map[string]int),
        waitingPods:           newWaitingPodsMap(),
        clientSet:             options.clientSet,
        eventRecorder:         options.eventRecorder,
        informerFactory:       options.informerFactory,
        metricsRecorder:       options.metricsRecorder,
        profileName:           options.profileName,
        runAllFilters:         options.runAllFilters,
    }
    // preemptHandle实现了PreemptHandle接口，其中Extenders和PodNominator都是通过opts传入的。
    // 因为frameworkImpl实现了Framework接口，所以也就是实现了PluginsRunner接口。
    f.preemptHandle = &preemptHandle{
        extenders:     options.extenders,
        PodNominator:  options.podNominator,
        PluginsRunner: f,
    }
    if plugins == nil {
        return f, nil
    }

    // pluginsNeeded()函数名是获取需要的插件，就是把plugins中所有使能(Enable)的插件转换成map[string]config.Plugin。
    // pg中是所有使能的插件，key是插件名字，value是config.Plugin(插件权重)，试问为什么要转换成map？
    // 笔者在调度插件的文章中提到了，很多插件实现了不同扩展点的插件接口，这会造成plugins中有很多相同名字的插件，只是分布在不同的扩展点。
    // 因为map有去重能力，这样可以避免相同的插件创建多个对象。
    pg := f.pluginsNeeded(plugins)

    // pluginConfig存储的是所有使能插件的参数
    pluginConfig := make(map[string]runtime.Object, len(args))
    for i := range args {
        // 遍历所有插件参数，并找到使能插件的参数，因为pg是所有使能的插件。
        name := args[i].Name
        if _, ok := pluginConfig[name]; ok {
            return nil, fmt.Errorf("repeated config for plugin %s", name)
        }
        pluginConfig[name] = args[i].Args
    }
    // outputProfile是需要输出的KubeSchedulerProfile对象。
    // 前文提到的SchedulingProfile对应的类型就是KubeSchedulerProfile，包括Profile的名字、插件配置以及插件参数。
    // 因为在构造frameworkImpl的时候会过滤掉不用的插件参数，所以调用者如果需要，可以通过opts传入回调函数捕获KubeSchedulerProfile。
    outputProfile := config.KubeSchedulerProfile{
        SchedulerName: f.profileName,
        Plugins:       plugins,
        PluginConfig:  make([]config.PluginConfig, 0, len(pg)),
    }

    // 此处需要区分framework.Plugin和config.Plugin，前者是插件接口基类，后者是插件权重配置，所以接下来就是创建所有使能的插件了。
    pluginsMap := make(map[string]framework.Plugin)
    var totalPriority int64
    // 遍历插件注册表
    for name, factory := range r {
        // 如果没有使能，则跳过。这里的遍历有点意思，为什么不遍历pg，然后查找r，这样不是更容易理解么？
        // 其实遍历pg会遇到一个问题，如果r中没有找到怎么办？也就是配置了一个没有注册的插件，报错么？
        // 而当前的遍历方法传达的思想是，就这多插件，使能了哪个就创建哪个，不会出错，不需要异常处理。
        if _, ok := pg[name]; !ok {
            continue
        }
        // getPluginArgsOrDefault()根据插件名字从pluginConfig获取参数，如果没有则返回默认参数，函数名字已经告诉我们一切了。
        args, err := getPluginArgsOrDefault(pluginConfig, name)
        if err != nil {
            return nil, fmt.Errorf("getting args for Plugin %q: %w", name, err)
        }
        // 输出插件参数，这也是为什么有捕获KubeSchedulerProfile的功能，因为在没有配置参数的情况下要创建默认参数。
        // 捕获KubeSchedulerProfile可以避免调用者再实现一次类似的操作，前提是有捕获的需求。
        if args != nil {
            outputProfile.PluginConfig = append(outputProfile.PluginConfig, config.PluginConfig{
                Name: name,
                Args: args,
            })
        }
        // 利用插件工厂创建插件，传入插件参数和框架句柄
        p, err := factory(args, f)
        if err != nil {
            return nil, fmt.Errorf("initializing plugin %q: %w", name, err)
        }
        pluginsMap[name] = p

        // 记录配置的插件权重
        f.pluginNameToWeightMap[name] = int(pg[name].Weight)
        // f.pluginNameToWeightMap[name] == 0有两种可能：1）没有配置权重，2）配置权重为0
        // 无论哪一种，权重为0是不允许的，即便它不是ScorePlugin，如果为0则使用默认值1
        if f.pluginNameToWeightMap[name] == 0 {
            f.pluginNameToWeightMap[name] = 1
        }
        // 需要了解的是framework.MaxTotalScore的值是64位整数的最大值，framework.MaxNodeScore的值是100；
        // 每个插件的标准化分数是[0, 100], 所有插件最大标准化分数乘以权重的累加之和不能超过int64最大值，
        // 否则在计算分数的时候可能会溢出，所以此处需要校验配置的权重是否会导致计算分数溢出。
        if int64(f.pluginNameToWeightMap[name])*framework.MaxNodeScore > framework.MaxTotalScore-totalPriority {
            return nil, fmt.Errorf("total score of Score plugins could overflow")
        }
        totalPriority += int64(f.pluginNameToWeightMap[name]) * framework.MaxNodeScore
    }

    // pluginsMap是所有已经创建好的插件，但是是map结构，现在需要把这些插件按照扩展点分到不同的slice中。
    // 即map[string]framework.Plugin->[]framework.QueueSortPlugin,[]framework.PreFilterPlugin...的过程。
    // 关于getExtensionPoints()和updatePluginList()的实现还是挺有意思的，用了反射，感兴趣的同学可以看一看。
    for _, e := range f.getExtensionPoints(plugins) {
        if err := updatePluginList(e.slicePtr, e.plugins, pluginsMap); err != nil {
            return nil, err
        }
    }

    // 检验ScorePlugin的权重不能为0，当前已经起不了作用了，因为在创建插件的时候没有配置权重的插件的权重都是1.
    // 但是这些代码还是有必要的，未来保不齐会修改前面的代码，这些校验可能就起到作用了。关键是在构造函数中，只会执行一次，这点校验计算量可以忽略不计。
    for _, scorePlugin := range f.scorePlugins {
        if f.pluginNameToWeightMap[scorePlugin.Name()] == 0 {
            return nil, fmt.Errorf("score plugin %q is not configured with weight", scorePlugin.Name())
        }
    }

    // 不能没有调度队列的排序插件，并且也不能有多个，多了没用。
    // 那么如果同时存在两种优先级计算方法怎么办？用不同的Framework，即不同的SchedulingProfile.
    if len(f.queueSortPlugins) == 0 {
        return nil, fmt.Errorf("no queue sort plugin is enabled")
    }
    if len(f.queueSortPlugins) > 1 {
        return nil, fmt.Errorf("only one queue sort plugin can be enabled")
    }
    // 不能没有绑定插件，否则调度的结果没法应用到集群
    if len(f.bindPlugins) == 0 {
        return nil, fmt.Errorf("at least one bind plugin is needed")
    }
    // 如果调用者需要捕获KubeSchedulerProfile，则将KubeSchedulerProfile回调给调用者
    if options.captureProfile != nil {
        if len(outputProfile.PluginConfig) != 0 {
            // 按照Profile名字排序
            sort.Slice(outputProfile.PluginConfig, func(i, j int) bool {
                return outputProfile.PluginConfig[i].Name < outputProfile.PluginConfig[j].Name
            })
        } else {
            outputProfile.PluginConfig = nil
        }
        options.captureProfile(outputProfile)
    }
    return f, nil
}
```

## 扩展点

| 扩展点名称      | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| Sort            | 对队列中的 Pod 进行排序，同时只能启用一个 Sort 扩展点的插件  |
| PreFilter       | 对 Filter 扩展点的数据做一些预处理操作，然后将其存入缓存中待 Filter 扩展点执行的时候使用 |
| Filter          | 针对当前 Pod，对所有节点进行过滤，在这个扩展点会过滤掉那些不适合的节点 |
| PostFilter      | 如果在 Filter 扩展点全部节点都被过滤掉了，没有合适的节点进行调度，才会执行 PostFilter 扩展点，如果启用了 Pod 抢占特性，那么会在这个扩展点进行抢占操作 |
| PreScore        | 对 Score 扩展点的数据做一些预处理操作，然后将其存入缓存中待 Score 扩展点执行的时候使用 |
| Score           | 针对当前 Pod，对所有的节点进行打分                           |
| Normalize Score | 针对 Score 扩展点的打分结果进行修正                          |
| Reserve         | 预留 PVC 等资源                                              |
| Permit          | 在执行绑定操作之前对当前 Pod 的调度进行最后的决策，包括：批准、拒绝或者延时调度 |
| WaitOnPermit    | 与 Permit 扩展点配合使用实现延时调度功能                     |
| PreBind         | 预绑定操作，可以为 Bind 扩展点做一些准备工作，也可以对附属资源进行绑定，例如 PVC 和 PV |
| Bind            | 将当前 Pod 与选出的节点进行绑定                              |
| PostBind        | 绑定的后置动作，可以执行通知操作，或者做一些资源清理工作     |

# 调度插件

1. 从调度队列的排序到最终Pod绑定到Node，整个过程分为QueueSortPlugin、PreFilterPlugin、FilterPlugin、PostFilterPlugin、PreScorePlugin、ScorePlugin、ReservePlugin、PermitPlugin、PreBindPlugin、BindPlugin、PostBindPlugin总共11个接口，官方称这些接口为调度框架的11个扩展点，可以按需扩展调度能力；
2. 每种类型的插件可以有多种实现，比如PreFiltePlugin就有InterPodAffinity、PodTopologySpread、SelectorSpread和TaintToleration 4种实现，每种实现都对应一种调度需求；
3. 一个类型可以实现多种类型插件接口（多继承），比如VolumeBinding就实现了PreFilterPlugin、FilterPlugin、ReservePlugin和PreBindPlugin 4种插件接口，因为PVC与PV绑定需要在调度过程的不同阶段做不同的事情；
4. 所有插件都需要静态编译注册到注册表中，可以通过Provider、Policy(File、ConfigMap)方式配置插件；
5. 每个配置都有一个名字，对应一个Scheduling Profile，调度Pod的时候可以指定调度器名字；

## 插件注册表

所有的插件统一注册在runtime.Registry的对象中

```go
// 插件注册表就是插件名字与插件工厂的映射，也就是可以根据插件名字创建插件。
type Registry map[string]PluginFactory

// 插件工厂用来创建插件，创建插件需要输入插件配置参数和框架句柄，其中runtime.Object可以想象为interface{}，可以是结构。
// framework.Handle就是调度框架句柄，为插件开放的接口
type PluginFactory = func(configuration runtime.Object, f framework.Handle) (framework.Plugin, error)
```

该注册表是静态编译的

```go
// 构建插件注册表，代码很简单，需要注意，如果自行实现插件，需要到这里来注册
func NewInTreeRegistry() runtime.Registry {
    return runtime.Registry{
        selectorspread.Name:                        selectorspread.New,
        imagelocality.Name:                         imagelocality.New,
        ·······
    }
}
```

## 插件配置

kube-scheduler当前支持三种方式(原理上是两种：1.Provider;2.Policy，只是Policy有两种而已)配置插件：

1. Provider: 源于插件注册表中已注册的Provider;
2. File: 从文件中读取配置；
3. ConfigMap: 从ConfigMap中读取配置

```go
switch {
    case source.Provider != nil:
        // 通过Provider创建配置
        sc, err := configurator.createFromProvider(*source.Provider)
        if err != nil {
            return nil, fmt.Errorf("couldn't create scheduler using provider %q: %v", *source.Provider, err)
        }
        sched = sc
    case source.Policy != nil:
        // 通过用户指定的policy源创建配置
        policy := &schedulerapi.Policy{}
        switch {
        // 通过文件创建配置
        case source.Policy.File != nil:
            if err := initPolicyFromFile(source.Policy.File.Path, policy); err != nil {
                return nil, err
            }
        // 通过ConfigMap创建配置
        case source.Policy.ConfigMap != nil:
            if err := initPolicyFromConfigMap(client, source.Policy.ConfigMap, policy); err != nil {
                return nil, err
            }
        }
        ...
    default:
        return nil, fmt.Errorf("unsupported algorithm source: %v", source)
    }
```

kube-scheduler内部运行着多个不同配置的调度器，每个调度器有一个名字，调度器是通过Scheduling Profile配置

```go
type KubeSchedulerProfile struct {
    // 调度器名字
    SchedulerName string
    // 指定应启用或禁止的插件集合，下面有代码注释
    Plugins *Plugins
    // 一组可选的自定义插件参数，无参数等于使用该插件的默认参数
    PluginConfig []PluginConfig
}

// Plugins包含了各个阶段插件的配置，PluginSet见下面代码注释
type Plugins struct {
    QueueSort *PluginSet
    PreFilter *PluginSet
    Filter *PluginSet
    PostFilter *PluginSet
    PreScore *PluginSet
    Score *PluginSet
    Reserve *PluginSet
    Permit *PluginSet
    PreBind *PluginSet
    Bind *PluginSet
    PostBind *PluginSet
}

type PluginSet struct {
    // 指定默认插件外还应启用的插件，这些将在默认插件之后以此处指定的顺序调用
    Enabled []Plugin
    // 指定应禁用的默认插件，当需要禁用所有的默认插件时，应提供仅包含一个"*"的数组
    Disabled []Plugin
}

type Plugin struct {
    // 插件名字
    Name string
    // 插件的权重，仅用于ScorePlugin插件
    Weight int32
}

type PluginConfig struct {
    // 插件名字
    Name string
    // 插件初始化时的参数，可以是任意结构
    Args runtime.Object
}
```

# 调度算法

在kube-scheduler中，有一种接口叫做ScheduleAlgorithm，就是负责在Node中为Pod选择一个最合适的Node

1. 只有同时满足FilterPlugin和Extender的过滤条件的Node才是可行Node，调度算法优先用FilterPlugin过滤，然后在用Extender过滤，这样可以尽量减少传给Extender的Node数量；
2. 调度算法为待调度的Pod对每个可行Node(过滤通过)进行评分，评分方法是`$\sum^n_0f(ScorePlugin_i)*w_i+\sum^m_0g(Extender_j)*w_j$`，其中`f(x)`和`g(x)`是标准化分数函数，w为权重；
3. 分数最高的Node为最优候选Node，当有多个Node都为最高分数时，每个Node有1/n的概率成最优Node，其中n是可行Node列表中第几个最高分数的Node，例如：第一个分数最高的Node成为最优的Node的概率是1/1，第二个是1/2，第三个是1/3...，以此类推；
4. 调度算法并不是对`调度框架`和`调度插件`再抽象和封装，只是对调度周期从PreFilter到Score的过程的一种抽象，其中PostFilter不在调度算法抽象范围内。因为PostFilter与过滤无关，是用来实现抢占的扩展点；

```go
// ScheduleAlgorithm抽象了调度一个Pod的接口。
type ScheduleAlgorithm interface {
  // 利用指定的调度框架调度Pod，为什么没有Node？因为Node是通过Cache获取的，所以在创建ScheduleAlgorithm对象的时候会传入Cache。
	// 来看看核心接口参数都有哪些：
	// 1. context.Context: 这个很好理解，当kube-scheduler退出时，调度算法也要及时退出，而不是完成复杂的调度算法再退出，否则可能无法实现优雅退出；
	// 2. framework.Framework: 调度一个Pod必须使用Pod.Spec.SchedulerName指定的调度框架，如果未指定使用默认的调度框架；
	// 3. *framework.CycleState: CycleState在调度插件的文章中有详细说明，此处简单说明一下，调度一个Pod分为调度周期和绑定周期，
	//							 CycleState用于存储调度一个Pod的整个周期的状态，调度一个Pod就是一轮调度。
	// 							 CycleState就是存储这一轮调度的各种状态，因为调度插件本身无状态，所以由调用者创建并在调度插件中共享使用。
	// 4. *v1.Pod: 这个很好理解，毕竟调度的就是一个Pod。
	// 为什么没有Node参数？否则调度算法怎么为Pod选择最优的Node？了解调度缓存的读者应该知道，Node相关信息是存储在调度缓存中。
	// 调度算法接口参数没有Node，唯一的解释就是调度算法对象有调度缓存亦或是调度缓存的快照，这样就不用每次调用都携带相同的参数了。
	// 还有，只有调度框架，为什么没有调度扩展程序(Extender)？因为调度扩展程序也是参与到调度Pod过程中的，唯一的解释与Node相同，
	// 即存储在调度算法对象中，这样不用每次都传递了，毕竟当前的设计调度扩展程序配置是相对静态的，一旦kube-scheduler启动就不会变化。
	Schedule(context.Context, framework.Framework, *framework.CycleState, *v1.Pod) (scheduleResult ScheduleResult, err error)
	// 这个接口只用来测试，所以不用关心
	Extenders() []framework.Extender
}

// ScheduleResult是调度一个Pod的结果，它包含最终选择的Node以及一些过程结果。
type ScheduleResult struct {
	// 最终选择的Node名字，也即是Pod即将调度到的Node
	SuggestedHost string
	// 调度过程中评估的Node数量
	EvaluatedNodes int
	// 需要引入一个概念：可行的(Feasible)Node，就是通过所有FilterPlguin的Node。FeasibleNodes就是可行Node的数量
	FeasibleNodes int
}
```

## genericScheduler

genericScheduler是ScheduleAlgorithm的一个实现(当前来看应该是唯一实现)，实现了调度一个Pod的普遍流程

```go
type genericScheduler struct {
  // 调度缓存，有了cache就有了所有的Node，这一点证明了笔者在解析ScheduleAlgorithm.Schedule()接口是的推断。
	cache                    internalcache.Cache
  // 所有调度扩展程序，这一点证明了笔者在解析ScheduleAlgorithm.Schedule()接口是的推断。
	extenders                []framework.Extender
  // 是否了解Cache.UpdateSnapshot(), 如果不了解请阅读https://github.com/jindezgm/k8s-src-analysis/blob/master/kube-scheduler/Cache.md
  // 每调度一个Pod，都需要调用Cache.UpdateSnapshot()来更新本地的nodeInfoSnapshot。
	nodeInfoSnapshot         *internalcache.Snapshot
  // 这个变量用来提升调度性能的，percentageOfNodesToScore是一个百分比，PN/TN*100>=percentageOfNodesToScore就可以进入评分阶段。
  // 其中PN是可行Node数量，TN是总共Node数量。percentageOfNodesToScore在Node数量比较多的情况下减少评分阶段的计算量。
  // 比如一共有1000个Node，如果1000个都是可行Node，那么就需要计算1000个Node的分数，这可能是一个比较大的计算量。
  // 真的需要1000个Node都参与评分么？不一定，可能100个就够了，有的时候不需要找到全局的最优解，局部最优也是不错的，只要这个局部数量别太少。
	percentageOfNodesToScore int32
  // 就是因为percentageOfNodesToScore的存在，可能只要一部分Node参与调度。
  // 那么下次调度的时候希望不要一直在同一批Node上调度，直到有Node过滤失败再有新的Node加入进来。
  // nextStartNodeIndex就是用来记录下一次调度的时候从哪个Node开始遍历，每次调度完成后nextStartNodeIndex需要加上此次调度遍历Node的数量。
  // 这样调度下一个Pod的时候就从下一批Node开始，当然每次更新nextStartNodeIndex的时候需要考虑超出范围归零的问题。
	nextStartNodeIndex       int
}
```

## Schedule接口实现

```go
// Schedule()实现了ScheduleAlgorithm.Schedule()接口.
func (g *genericScheduler) Schedule(ctx context.Context, fwk framework.Framework, state *framework.CycleState, pod *v1.Pod) (result ScheduleResult, err error) {
	trace := utiltrace.New("Scheduling", utiltrace.Field{Key: "namespace", Value: pod.Namespace}, utiltrace.Field{Key: "name", Value: pod.Name})
	defer trace.LogIfLong(100 * time.Millisecond)

  // snapshot()函数就是调用了g.cache.UpdateSnapshot(g.nodeInfoSnapshot)，更新Node快照，获取最新的Node状态
	if err := g.snapshot(); err != nil {
		return result, err
	}
	trace.Step("Snapshotting scheduler cache and node infos done")

  // 如果没有Node，只能返回错误了。比如，当前集群还没有Node，此时创建的Pod在调度过程中产生的错误就是这里返回的。
	if g.nodeInfoSnapshot.NumNodes() == 0 {
		return result, ErrNoNodesAvailable
	}

    // findNodesThatFitPod()找到Pod的可行Node，就是通过FilterPlugin(包括前处理)实现的。
	// findNodesThatFitPod()函数后续章节会详细解析。
	feasibleNodes, diagnosis, err := g.findNodesThatFitPod(ctx, fwk, state, pod)
	if err != nil {
		return result, err
	}
	trace.Step("Computing predicates done")

  // 如果没有可行Node，返回错误，错误中包含了Node的数量以及每个Node过滤失败的结果，这些错误信息是为抢占调度提供参考的。
	// 一句话表达: 没有Node满足Pod的需求，唯一的可行性就是抢占。
	if len(feasibleNodes) == 0 {
		return result, &framework.FitError{
			Pod:         pod,
			NumAllNodes: g.nodeInfoSnapshot.NumNodes(),
			Diagnosis:   diagnosis,
		}
	}

	// 如果只有一个Node满足要求，那么该Node就是最优选，没有必要再通过评分寻找最优Node
	if len(feasibleNodes) == 1 {
		return ScheduleResult{
			SuggestedHost:  feasibleNodes[0].Name,
			EvaluatedNodes: 1 + len(diagnosis.NodeToStatusMap),
			FeasibleNodes:  1,
		}, nil
	}

  // 'prioritize'是以前版本对于评分(Score)的称呼，就是给Node评分的过程。
	// 后续章节会有prioritizeNodes()详细解析。
	priorityList, err := g.prioritizeNodes(ctx, fwk, state, pod, feasibleNodes)
	if err != nil {
		return result, err
	}

  // 寻找最合适的Node，详情参看后续章节关于selectHost()的解析。
	host, err := g.selectHost(priorityList)
	trace.Step("Prioritizing done")

  // 返回调度结果。
	return ScheduleResult{
		SuggestedHost:  host,
		EvaluatedNodes: len(feasibleNodes) + len(diagnosis.NodeToStatusMap),
		FeasibleNodes:  len(feasibleNodes),
	}, err
}
```

### 过滤

又分成过滤前、过滤、过滤后

```go
// findNodesThatFitPod()找到所有满足Pod需求的Node，返回值framework.Diagnosis用来记录中间结果，感兴趣的读者可以看看类型定义。
// 当然，可以从本函数的代码可以猜出framework.Diagnosis的大部分功能。
func (g *genericScheduler) findNodesThatFitPod(ctx context.Context, fwk framework.Framework, state *framework.CycleState, pod *v1.Pod) ([]*v1.Node, framework.Diagnosis, error) {
	// framework.Diagnosis包含了过滤每个Node的结果(NodeToStatusMap)以及哪些插件返回了不可调度。
	// 调度框架、调度插件、调度算法等模块称结果为状态，也对，状态包括成功、错误等。
	diagnosis := framework.Diagnosis{
		NodeToStatusMap:      make(framework.NodeToStatusMap),
		UnschedulablePlugins: sets.NewString(),
	}

	// 敲黑板！从现在开始就会感受到调度框架的魅力，因为有了调度框架，所以的代码感觉如此的顺其自然。
    // 运行所有的过滤前处理插件，为过滤准备数据。
	s := fwk.RunPreFilterPlugins(ctx, state, pod)
	// 获取所有的Node，此处是从Cache的快照中获取，此时是不是对快照的价值有一些认识了呢？
	// 正是因为有快照的存在，才使得每次获取的Node都是相同的，如果直接从Cache中列举每次可能都有不同。
	allNodes, err := g.nodeInfoSnapshot.NodeInfos().List()
	if err != nil {
		return nil, diagnosis, err
	}
	if !s.IsSuccess() {
		// 如果不是不可调度的状态，那么按照错误返回。
        // 因为有些插件需要访问一些外部资源(即插件不可控的对象、函数等)，就会有返回错误的可能。
		if !s.IsUnschedulable() {
			return nil, diagnosis, s.AsError()
		}
		// 程序运行到这里说明Pod不可调度，那么问题来了，不是过滤的前处理么？怎么也有类似过滤的功能？
        // 笔者在此解释一下，过滤前处理是笔者对PreFilter的定义，主要功能是为Filter准备数据。
        // 但是不排除有些情况在因为无法获取准备的数据造成Pod暂时无法被调度，比如Pod声明的PVC不存在。
        // PreFilterPlugin实现的功能主要是为FilterPlugin准备数据而不是过滤，所有笔者更习惯称之为前处理。
		// 由于过滤前处理返回了不可调度状态，那么对于所有的Node来说，该Pod都是不可调度的。
		for _, n := range allNodes {
			diagnosis.NodeToStatusMap[n.Node().Name] = s
		}
		// 记录哪些返回不可调度状态
		diagnosis.UnschedulablePlugins.Insert(s.FailedPlugin())
		return nil, diagnosis, nil
	}

	// 由于抢占调度有可能在先前的调度周期中设置Pod.Status.NominatedNodeName。
	// 该Node可能是唯一适合该Pod的候选Node，因此，在遍历所有Node之前先尝试提名的Node。
	// 这个特性当前还不是发布状态，如果使用需要使能该特性。
	if len(pod.Status.NominatedNodeName) > 0 && feature.DefaultFeatureGate.Enabled(features.PreferNominatedNode) {
		// 评估提名的Node，至于怎么评估的，下面有evaluateNominatedNode()函数注释。
		feasibleNodes, err := g.evaluateNominatedNode(ctx, pod, fwk, state, diagnosis)
		if err != nil {
			klog.ErrorS(err, "Evaluation failed on nominated node", "pod", klog.KObj(pod), "node", pod.Status.NominatedNodeName)
		}
		// 提名的Node评估通过，应该将提名Node分配给Pod
		if len(feasibleNodes) != 0 {
			return feasibleNodes, diagnosis, nil
		}
	}
	// 利用FilterPlugin过滤Node，代码见下面注释
	feasibleNodes, err := g.findNodesThatPassFilters(ctx, fwk, state, pod, diagnosis, allNodes)
	if err != nil {
		return nil, diagnosis, err
	}

    // 对已经过滤的Node再用Extender过滤一遍，FilterPlugin.Filter()和Extender.Filter()是串联调用的.
    // 说明Pod必须同时满足FilterPlugin.Filter()和Extender.Filter()的过滤条件，两个过滤条件是'与'的关系。
    // 先用插件过滤再用Extender过滤，这样可以提前过滤掉不符合插件要求的Node，避免不必要的Node传输给Extender，这也是一种优化。
	// findNodesThatPassExtenders()代码下面有注释。
	feasibleNodes, err = g.findNodesThatPassExtenders(pod, feasibleNodes, diagnosis.NodeToStatusMap)
	if err != nil {
		return nil, diagnosis, err
	}
	return feasibleNodes, diagnosis, nil
}

// evaluateNominatedNode()评估提名Node是否满足Pod的需求。
func (g *genericScheduler) evaluateNominatedNode(ctx context.Context, pod *v1.Pod, fwk framework.Framework, state *framework.CycleState, diagnosis framework.Diagnosis) ([]*v1.Node, error) {
	// 根据提名Node的名字找到Node
	nnn := pod.Status.NominatedNodeName
	nodeInfo, err := g.nodeInfoSnapshot.Get(nnn)
	if err != nil {
		return nil, err
	}
	// 只将提名Node作为候选利用FilterPlugin过滤一下
	node := []*framework.NodeInfo{nodeInfo}
	feasibleNodes, err := g.findNodesThatPassFilters(ctx, fwk, state, pod, diagnosis, node)
	if err != nil {
		return nil, err
	}

	// 再利用调度扩展程序过滤一下
	feasibleNodes, err = g.findNodesThatPassExtenders(pod, feasibleNodes, diagnosis.NodeToStatusMap)
	if err != nil {
		return nil, err
	}

	// 评估提名Node其实就是看看提名Node是否满足过滤条件，感觉'评估'这个词有点大了，希望以后能扩展评估的范围吧...
	return feasibleNodes, nil
}

// findNodesThatPassFilters()利用FilterPlugin过滤Node
func (g *genericScheduler) findNodesThatPassFilters(
	ctx context.Context,
	fwk framework.Framework,
	state *framework.CycleState,
	pod *v1.Pod,
	diagnosis framework.Diagnosis,
	nodes []*framework.NodeInfo) ([]*v1.Node, error) {
	// numFeasibleNodesToFind()函数用来计算有多少Node通过过滤就可以了，还记得percentageOfNodesToScore这个成员变量么？
    // 换句话说就是如果通过过滤的Node太多会对后面的评分带来压力，但是太多并不会带来明显的提升，只要通过过滤的一部分就可以了。
    // numFeasibleNodesToFind()函数后面有注释。
	numNodesToFind := g.numFeasibleNodesToFind(int32(len(nodes)))

	// 创建过滤通过Node缓存，最多numNodesToFind个
	feasibleNodes := make([]*v1.Node, numNodesToFind)

    // 没有过滤插件等同于每个Node都能通过，所以直接返回连续的numNodesToFind的Node就可以了。
	if !fwk.HasFilterPlugins() {
		length := len(nodes)
		for i := range feasibleNodes {
			feasibleNodes[i] = nodes[(g.nextStartNodeIndex+i)%length].Node()
		}
		g.nextStartNodeIndex = (g.nextStartNodeIndex + len(feasibleNodes)) % length
		return feasibleNodes, nil
	}

	// 因为过滤每个Node彼此是无耦合的，所以可以用多协程对每个Node执行过滤操作，此处为并行操作做准备
    // 当并行运行的协程处理函数遇到错误可以将错误写入errCh，注意只有第一个写入成功，因为chan缓冲大小是1
	// 为什么不把chan缓冲设置为numNodesToFind？很简单，一个错误就可以终止Pod的调度，多了也没用。
	errCh := parallelize.NewErrorChannel()
	// 因为存在多协程并发写入framework.Diagnosis的可能，所以需要一个锁保护一下。
	// 需要注意的是framework.Diagnosis是1.21版本才引入的，之前是framework.NodeToStatusMap类型，所以锁的名字一直没有变。
	var statusesLock sync.Mutex
    // feasibleNodesLen用来记录当前已经过滤通过的Node数量，多协程之间共享，因为是原子操作它，所以不用加锁	
	var feasibleNodesLen int32
    // 为并发准备Context
	ctx, cancel := context.WithCancel(ctx)
	// checkNode是并行的协程函数，用来检测一个Node是否能够通过过滤，参数i是Node的索引
	checkNode := func(i int) {
		// 第i个协程过滤第i个Node，g.nextStartNodeIndex是本次调度周期的起始索引，这种操作应该比较常见不足为奇了吧
		nodeInfo := nodes[(g.nextStartNodeIndex+i)%len(nodes)]
		// 两个不同的地方会调用RunFilterPluginsWithNominatedPods函数：Schedule和Preempt(抢占)。
		// 当Schedule调用它时，要检测该Pod是否可以调度到该节Node上与已经存在的Pod加上提名到该节点的更高优先级或同等优先级的Pod一起运行。
		// 也就是说，不只是简单的过滤，同时还要考虑已经提名到该节点的Pod对该Pod的影响。
		status := fwk.RunFilterPluginsWithNominatedPods(ctx, state, pod, nodeInfo)
		// 如果过滤出错，则输出错误并返回
		if status.Code() == framework.Error {
			errCh.SendErrorWithCancel(status.AsError(), cancel)
			return
		}

		// 判断过滤是否通过
		if status.IsSuccess() {
			// Node过滤通过，则在feasibleNodesLen总数上加一
			length := atomic.AddInt32(&feasibleNodesLen, 1)
			if length > numNodesToFind {
				// 数量已经超过numNodesToFind，取消其他正在运行的协程
				cancel()
				// 总数减一，因为并没有输出到feasibleNodes中，这是很常规的原子操作使用方法
				atomic.AddInt32(&feasibleNodesLen, -1)
			} else {
				// 记录过滤通过的Node
				feasibleNodes[length-1] = nodeInfo.Node()
			}
		} else {
			// 过滤未通过，则记录该Node的状态，并且记录哪些插件过滤失败，过滤失败其实就是Pod不可调度。
			statusesLock.Lock()
			diagnosis.NodeToStatusMap[nodeInfo.Node().Name] = status
			diagnosis.UnschedulablePlugins.Insert(status.FailedPlugin())
			statusesLock.Unlock()
		}
	}

	beginCheckNode := time.Now()
	statusCode := framework.Success
	defer func() {
		metrics.FrameworkExtensionPointDuration.WithLabelValues(runtime.Filter, statusCode.String(), fwk.ProfileName()).Observe(metrics.SinceInSeconds(beginCheckNode))
	}()

	// 启动多个协程并行过滤Node，此处需要注意的是并不是启动了len(allNodes)协程，parallelize包有一个并行度，默认值是16.
    // 如果并行度是16，那么最多启动16个协程，每个协程负责过滤一部分Node，就是典型的MapReduce模型。
    // 举个例子，如果总共有64个Node，那么每个协程负责过滤4个连续的Node。
	fwk.Parallelizer().Until(ctx, len(nodes), checkNode)
	// 总共过滤的Node数量是过滤失败的Node与成功的Node总和
	processedNodes := int(feasibleNodesLen) + len(diagnosis.NodeToStatusMap)
	// 计算下一个调度周期过滤Node的起始索引，就是当前的起始索引+过滤Node的总数，因为这些Node参与了当前Pod的过滤计算。
    // 看似没什么问题，其实这可能不是最好的方法：
	// 1. 过滤的Node里面有一些不满足Pod需求未通过，但是这些Node可能会满足下一个Pod的需求；
	// 2. 即便过滤通过的Node，最终只有一个Node会被选中，其他Node陪玩一轮，并且不会进入下一轮；
	// 3. 对于资源均匀分配的策略来说还好，但是对于资源需要集中分配的策略来说可能是个灾难；
	g.nextStartNodeIndex = (g.nextStartNodeIndex + processedNodes) % len(nodes)

    // feasibleNodes是按照最大量(numNodesToFind)申请的内存，实际可能没有那么多，所以需要调整到实际过滤通过的Node数量
	feasibleNodes = feasibleNodes[:feasibleNodesLen]
	// 如果过滤过程中有错误，则返回错误
	if err := errCh.ReceiveError(); err != nil {
		statusCode = framework.Error
		return nil, err
	}
	// 返回过滤成功的Node
	return feasibleNodes, nil
}

// numFeasibleNodesToFind()用来计算通过过滤Node的数量阈值，kube-scheduler认为通过过滤Node数量超过该阈值对调度效果影响不大，但是计算量会增大。
// 所以genericScheduler.percentageOfNodesToScore就是用来设置一个比例值，只选择Node总数一定比例的Node就可以了。
// 因为这个比例是配置的，有些情况可能不合理(比如Node总数不多)，所以需要需要结合实际情况计算。
func (g *genericScheduler) numFeasibleNodesToFind(numAllNodes int32) (numNodes int32) {
	// minFeasibleNodesToFind是一个常量(100)，也就是说无论比例是多少，最少也要100个Node，除非Node总数就不足100个。
    // g.percentageOfNodesToScore >= 100等于比例无效，无论多少Node都可以。
    // 这两种情况就不用计算，有多少来多少
	if numAllNodes < minFeasibleNodesToFind || g.percentageOfNodesToScore >= 100 {
		return numAllNodes
	}

    // 如果配置的比例<=0，需要计算自适应的比例
	adaptivePercentage := g.percentageOfNodesToScore
	if adaptivePercentage <= 0 {
		// 基础比例是50%
		basePercentageOfNodesToScore := int32(50)
		// Node总数满125个比例减一，可以持续累加，这个跟双十一满减一个原理，至于为什么是125笔者还没弄清楚。
        // 这可以理解，单纯的比例并不合理，尤其是Node总量非常大的情况下，比例应该适当调低。
        // 所以可以用函数y=(50x-0.008x^2)/100表示最终Node数量。初中数据告诉我们这是一个开口朝下的抛物线。
        // 这个函数有一个最大值，是当x=-b/2a，即Node总量为3125个时，应该选择782个Node。
        // 也就是说，当Node总量在[100(前面过滤了小于100的情况), 3125]时，选择Node的数量是在[49，782]范围单调递增的；
        // 当Node数量[3125, 7250]时，选择Node的数量是在[782, 0]单点递减的，超过7250比例就会变为负数。
		adaptivePercentage = basePercentageOfNodesToScore - numAllNodes/125
		// 比例总不能<=0吧？所以就有了minFeasibleNodesPercentageToFind(5)，即最小比例也应该是5%
		if adaptivePercentage < minFeasibleNodesPercentageToFind {
			adaptivePercentage = minFeasibleNodesPercentageToFind
		}
	}

	// 按照比例计算Node数量，adaptivePercentage = 50 - 0.008*numAllNodes
	// 所以numNodes = (50*numAllNodes - 0.008*numAllNodes^2) / 100，就是上面y=(50x-0.008x^2)/100的公式
	numNodes = numAllNodes * adaptivePercentage / 100
	// 最终还要限制在>=100范围，因为太少会影响调度效果
	if numNodes < minFeasibleNodesToFind {
		return minFeasibleNodesToFind
	}

	return numNodes
}

// findNodesThatPassExtenders()利用Extender过滤Node
func (g *genericScheduler) findNodesThatPassExtenders(pod *v1.Pod, feasibleNodes []*v1.Node, statuses framework.NodeToStatusMap) ([]*v1.Node, error) {
	// 遍历Extender, Extender是顺序调用的，而不是并发调用的。在一个Extender中排除一些可行Node，然后以递减的方式传递给下一个Extender。
	// 这种设计虽然可能降低Extender的传递Node的数据量，但是串行执行远程调用效率确实不可观。
	for _, extender := range g.extenders {
		// 可行的Node数量为0就没必要过滤了
		if len(feasibleNodes) == 0 {
			break
		}
		// Pod申请的资源不是Extender管理的也就不需要用它来过滤，这也是一种有效的提升效率的方法，尽量避免无效的远程调用。
		if !extender.IsInterested(pod) {
			continue
		}

		// 调用Exender过滤Node，了解Extender过滤详情参看笔者关于Extender的文章。
		feasibleList, failedMap, failedAndUnresolvableMap, err := extender.Filter(pod, feasibleNodes)
		if err != nil {
			// 如果Extender是可以忽略的，就会忽略Extender报错，也就是说调度Pod有没有该Extender参与都行，有自然更好。
			if extender.IsIgnorable() {
				klog.InfoS("Skipping extender as it returned error and has ignorable flag set", "extender", extender, "err", err)
				continue
			}
			return nil, err
		}

		// failAndUnresolvableMap中失败节点的状态会添加或覆盖到statuses。
		// 以便调度框架可以知悉特定节点的UnschedulableAndUnresolvable状态，这最终可以提高抢占效率。
		for failedNodeName, failedMsg := range failedAndUnresolvableMap {
			var aggregatedReasons []string
			if _, found := statuses[failedNodeName]; found {
				aggregatedReasons = statuses[failedNodeName].Reasons()
			}
			aggregatedReasons = append(aggregatedReasons, failedMsg)
			statuses[failedNodeName] = framework.NewStatus(framework.UnschedulableAndUnresolvable, aggregatedReasons...)
		}

		// 追加Extender过滤失败的原因
		for failedNodeName, failedMsg := range failedMap {
			if _, found := failedAndUnresolvableMap[failedNodeName]; found {
				// failedAndUnresolvableMap优先于failedMap，请注意，仅当Extender在两个map中都返回Node时，才会发生这种情况
				continue
			}
			if _, found := statuses[failedNodeName]; !found {
				statuses[failedNodeName] = framework.NewStatus(framework.Unschedulable, failedMsg)
			} else {
				statuses[failedNodeName].AppendReason(failedMsg)
			}
		}

		// Extender过滤通过的Node作为下一个Extender过滤的候选
		feasibleNodes = feasibleList
	}
	return feasibleNodes, nil
}
```

### 评分

过滤通过的Node又称之为可行Node，将作为评分阶段的候选Node列表，调度算法将利用ScorePlugin与Extender对每个可行Node进行评分。

```go
// prioritizeNodes()为待调度的Pod给没给候选Node进行评分
func (g *genericScheduler) prioritizeNodes(
	ctx context.Context,
	fwk framework.Framework,
	state *framework.CycleState,
	pod *v1.Pod,
	nodes []*v1.Node,
) (framework.NodeScoreList, error) {
	// 既没有Extender也没有ScorePlugin，则所有节点的得分均为1。
	if len(g.extenders) == 0 && !fwk.HasScorePlugins() {
		result := make(framework.NodeScoreList, 0, len(nodes))
		for i := range nodes {
			result = append(result, framework.NodeScore{
				Name:  nodes[i].Name,
				Score: 1,
			})
		}
		return result, nil
	}

	// 运行所有PreScorePlugin准备评分数据
	preScoreStatus := fwk.RunPreScorePlugins(ctx, state, pod, nodes)
	if !preScoreStatus.IsSuccess() {
		return nil, preScoreStatus.AsError()
	}

	// 运行所有ScorePlugin对Node进行评分，此处需要知道的是scoresMap的类型是map[string][]NodeScore。
    // scoresMap的key是插件名字，value是该插件对所有Node的评分
	scoresMap, scoreStatus := fwk.RunScorePlugins(ctx, state, pod, nodes)
	if !scoreStatus.IsSuccess() {
		return nil, scoreStatus.AsError()
	}

	if klog.V(10).Enabled() {
		for plugin, nodeScoreList := range scoresMap {
			for _, nodeScore := range nodeScoreList {
				klog.InfoS("Plugin scored node for pod", "pod", klog.KObj(pod), "plugin", plugin, "node", nodeScore.Name, "score", nodeScore.Score)
			}
		}
	}

	// result用于汇总所有分数。
	result := make(framework.NodeScoreList, 0, len(nodes))
	// 累加所有插件对于Node的评分
	for i := range nodes {
		result = append(result, framework.NodeScore{Name: nodes[i].Name, Score: 0})
		for j := range scoresMap {
			// 还记得在构建Framework的时候对配置权重的校验么？如果不做校验，那么此处就可能会溢出。
			// 需要注意的是，此处的分数是标准化分数并乘以插件对应的权重，相关的计算在调度框架中实现
			result[i].Score += scoresMap[j][i].Score
		}
	}

	// 如果配置了Extender，还要调用Extender对Node评分并累加到result中。
	if len(g.extenders) != 0 && nodes != nil {
		// 因为要多协程并发调用Extender并统计分数，所以需要锁来互斥写入Node分数
		var mu sync.Mutex
		var wg sync.WaitGroup
		// combinedScores的key是Node名字，value是Node评分
		combinedScores := make(map[string]int64, len(nodes))
		for i := range g.extenders {
			// 如果Extender不管理Pod申请的资源则跳过
			if !g.extenders[i].IsInterested(pod) {
				continue
			}
			// 启动协程调用Extender对所有Node评分。
			wg.Add(1)
			go func(extIndex int) {
				metrics.SchedulerGoroutines.WithLabelValues(metrics.PrioritizingExtender).Inc()
				defer func() {
					metrics.SchedulerGoroutines.WithLabelValues(metrics.PrioritizingExtender).Dec()
					wg.Done()
				}()
				// 调用Extender对Node进行评分
				prioritizedList, weight, err := g.extenders[extIndex].Prioritize(pod, nodes)
				if err != nil {
					// 此处有点意思，并没有返回错误，因为评分不同于过滤，对错误不敏感。过滤如果失败是要返回错误的(如果不能忽略)，因为Node可能无法满足Pod需求；
                    // 而评分无非是选择最优的节点，评分错误只会对选择最优有一点影响，但是不会造成故障。
					return
				}
				// 与其他插件对Node的评分进行合并
				mu.Lock()
				for i := range *prioritizedList {
					host, score := (*prioritizedList)[i].Host, (*prioritizedList)[i].Score
					if klog.V(10).Enabled() {
						klog.InfoS("Extender scored node for pod", "pod", klog.KObj(pod), "extender", g.extenders[extIndex].Name(), "node", host, "score", score)
					}
					// Extender的权重是通过Prioritize()返回的，其实该权重是人工配置的，只是通过Prioritize()返回使用上更方便。
					// 合并后的评分是每个Extender对Node评分乘以权重的累加和
					combinedScores[host] += score * weight
				}
				mu.Unlock()
			}(i)
		}
		// 等待所有协程退出，即便是多协程并发调用Extender，也会存在木桶效应，即调用时间取决于最慢的Extender。
        // 虽然Extender可能都很快，但是网络延时是一个比较常见的事情，更严重的是如果Extender异常造成调度超时，那么就拖累了整个kube-scheduler的调度效率。
		wg.Wait()
		for i := range result {
			// 最终Node的评分是所有ScorePlugin分数总和+所有Extender分数总和
			// 此处标准化了Extender的评分，使其范围与ScorePlugin一致，否则二者没法累加在一起。
			result[i].Score += combinedScores[result[i].Name] * (framework.MaxNodeScore / extenderv1.MaxExtenderPriority)
		}
	}

	if klog.V(10).Enabled() {
		for i := range result {
			klog.InfoS("Calculated node's final score for pod", "pod", klog.KObj(pod), "node", result[i].Name, "score", result[i].Score)
		}
	}
	return result, nil
}
```

### 选择

对Node评分不是目的，只是一种手段，最终目的是选出最优的Node。如何选择最优，我们立刻能想到的就是分最高的Node，那么问题来了，如果最高分数相同怎么办？

```go
// selectHost()根据所有可行Node的评分找到最优的Node
func (g *genericScheduler) selectHost(nodeScoreList framework.NodeScoreList) (string, error) {
	// 没有可行Node的评分，返回错误
	if len(nodeScoreList) == 0 {
		return "", fmt.Errorf("empty priorityList")
	}
	// 在nodeScoreList中找到分数最高的Node，初始化第0个Node分数最高
	maxScore := nodeScoreList[0].Score
	selected := nodeScoreList[0].Name
	// 如果做高分数相同怎么办？genericScheduler的解决方法是先统计数量(cntOfMaxScore)
	cntOfMaxScore := 1
	for _, ns := range nodeScoreList[1:] {
		// 如果分数比候选的分数高则成为候选
		if ns.Score > maxScore {
			maxScore = ns.Score
			selected = ns.Name
			cntOfMaxScore = 1
		} else if ns.Score == maxScore {
			// 分数相同就累计数量
			cntOfMaxScore++
			// 有1/cntOfMaxScore的概率成为最优Node，毕竟大家分数都相同，选中谁就看命运了，当然越靠前概率越高。
			// 为什么越靠前概率越高，因为越往后数量越多，概率自然就越小，比如第二个同分的Node就有50%的概率会成为最优Node
			if rand.Intn(cntOfMaxScore) == 0 {
				selected = ns.Name
			}
		}
	}
	return selected, nil
}
```

# 提名

需要了解PodNominator到底是干啥的。看其源码注释

```go
    // nominatedNodeName is set only when this pod preempts other pods on the node, but it cannot be
    // scheduled right away as preemption victims receive their graceful termination periods.
    // This field does not guarantee that the pod will be scheduled on this node. Scheduler may decide
    // to place the pod elsewhere if other nodes become available sooner. Scheduler may also decide to
    // give the resources on this node to a higher priority pod that is created after preemption.
    // As a result, this field may be different than PodSpec.nodeName when the pod is
    // scheduled.
    // +optional
    NominatedNodeName string `json:"nominatedNodeName,omitempty" protobuf:"bytes,11,opt,name=nominatedNodeName"`
```

简单总结源码注释就是当Pod抢占Node上其他Pod的时候会设置Pod.Status.NominatedNodeName为Node的名字。调度器不会立刻将Pod调度到Node上，因为需要等到被抢占的Pod优雅退出。