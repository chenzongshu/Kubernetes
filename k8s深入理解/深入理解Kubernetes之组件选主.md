# 简介

Kubernetes的APIServer是无状态多主的模式，而Controller Manager和Scheduler都是主从模式，那Kubernetes是怎么选主的呢，我们今天来分析一下，也可以指导下我们Controller的编写

## 锁

### 悲观锁

悲观并发控制（又名“悲观锁”，Pessimistic Concurrency Control，缩写“PCC”）是一种并发控制的方法。它可以阻止一个事务以影响其他用户的方式来修改数据。如果一个事务执行的操作读某行数据应用了锁，那只有当这个事务把锁释放，其他事务才能够执行与该锁冲突的操作。

在悲观锁的场景下，假设用户 A 和 B 要修改同一个文件，A 在锁定文件并且修改的过程中，B 是无法修改这个文件的，只有等到 A 修改完成，并且释放锁以后，B 才可以获取锁，然后修改文件。由此可以看出，悲观锁对并发的控制持悲观态度，它在进行任何修改前，首先会为其加锁，确保整个修改过程中不会出现冲突，从而有效的保证数据一致性。但这样的机制同时降低了系统的并发性，尤其是两个同时修改的对象本身不存在冲突的情况。同时也可能在竞争锁的时候出现死锁，所以现在很多的系统例如 Kubernetes 采用了乐观并发的控制方法。

### 乐观锁

乐观并发控制（又名“乐观锁”，Optimistic Concurrency Control，缩写“OCC”）是一种并发控制的方法。它假设多用户并发的事务在处理时不会彼此影响，各事务能够在不请求锁的情况下处理各自的数据。在提交数据更新之前，每个事务会先检查在该事务读取数据后，有没有其他事务又修改了该数据。如果其他事务有更新的话，正在提交的事务会进行回滚。

相对于悲观锁对锁的提前控制，乐观锁相信请求之间出现冲突的概率是比较小的，在读取及更改的过程中都是不加锁的，只有在最后提交更新时才会检测冲突，因此在高并发量的系统中占有绝对优势。同样假设用户A和B要修改同一个文件，A和B会先将文件获取到本地，然后进行修改。如果A已经修改好并且将数据提交，此时B再提交，服务器端会告知B文件已经被修改，返回冲突错误。此时冲突必须由B来解决，可以将文件重新获取回来，再一次修改后提交。

乐观锁通常通过增加一个资源版本字段，来判断请求是否冲突。初始化时指定一个版本值，每次读取数据时将版本号一同读出，每次更新数据，同时也对版本号进行更新。当服务器端收到数据时，将数据中的版本号与服务器端的做对比，如果不一致，则说明数据已经被修改，返回冲突错误

## k8s资源锁

分布式锁实现很多，也可以基于中间件比如zookeeper或者etcd这种。

k8s都没用，用了自己的资源来实现锁，目前有三种：

- configmap
- endpoint
- lease

> kubernetes从1.15推荐使用lease，前两种弃用

我们以Controller Manager为例子看看

可以首先看看哪个是主服务

```yaml
# kubectl -n kube-system get ep kube-controller-manager -o yaml
apiVersion: v1
kind: Endpoints
metadata:
  annotations:
    control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"node3_8593e385-c447-40da-853b-859fe3875971","leaseDurationSeconds":15,"acquireTime":"2021-01-28T02:45:08Z","renewTime":"2021-04-25T09:33:15Z","leaderTransitions":2}'
  creationTimestamp: "2020-12-03T07:14:15Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:control-plane.alpha.kubernetes.io/leader: {}
    manager: kube-controller-manager
    operation: Update
    time: "2021-04-25T09:33:15Z"
  name: kube-controller-manager
  namespace: kube-system
  resourceVersion: "54689011"
  selfLink: /api/v1/namespaces/kube-system/endpoints/kube-controller-manager
  uid: 2aeca749-c482-408b-a03c-6b5d82c82d99
```

`annotations`里面的 `control-plane.alpha.kubernetes.io/leader`里面有存储了选主的状态

- `holderIdentity`：用于表示当前拥有者，
- `acquireTime`为拥有者取得持有权的时间，
- `renewTime`为当前拥有者上一次活跃时间。而更换Leader条件是当renewTime与自己当下时间计算超过`leaseDurationSeconds`时进行。

查看lease资源

```yaml
# kubectl get lease -n kube-system
NAMESPACE         NAME                      HOLDER                                       AGE
kube-system       kube-controller-manager   node3_8593e385-c447-40da-853b-859fe3875971   143d
kube-system       kube-scheduler            node3_f8fb887e-9df1-404f-a3b3-24bb7d6a9961   143d

# kubectl -n kube-system get lease kube-controller-manager -o yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  creationTimestamp: "2020-12-03T07:14:15Z"
······
spec:
  acquireTime: "2021-01-28T02:45:08.000000Z"
  holderIdentity: node3_8593e385-c447-40da-853b-859fe3875971
  leaseDurationSeconds: 15
  leaseTransitions: 2
  renewTime: "2021-04-25T09:42:13.266234Z"
```

### 原理

原理很简单，Kubernetes 提供了一种带 “租期” 的类型 Lease，然后，通过创建 Name 唯一的 LeaseLock 资源，多个参与者中只有一个人能创建成功，那么谁创建成功（拥有）这个资源，谁就是 Leader。这里带 ”租期“，所以要求拥有这个资源的 Leader 必须定时续租，如果断租了，那么别的参与者就可以抢占这个 Lock。

因为有多个参与者参与抢占这个资源，所以会出现竞争，而解决竞争的方式 Kubernetes 是通过版本号的乐观锁解决。虽然在使用的时候，我们对这个版本号没有感觉，但是，通过查看 Kubernetes Client Go 的代码可以发现，这个事情其实是 SDK 帮助我们完成了

> 资源版本 (ResourceVersion)字段包含在 Kubernetes 对象的元数据 (Metadata)中。这个字符串格式的字段标识了对象的内部版本号，其取值来自 etcd 的 modifiedindex，且当对象被修改时，该字段将随之被修改。值得注意的是该字段由服务端维护，不建议在客户端进行修改。



# Demo

client-go写一个类似的Controller，（k8s源码中就有demo：`staging/src/k8s.io/client-go/examples/leader-election/main.go`）

核心代码如下：

```go
	if leaderElect {
    // 先建一个lease的资源锁
		lock := &resourcelock.LeaseLock{
			LeaseMeta: metav1.ObjectMeta{
				Name:      leaseLockName,
				Namespace: leaseLockNamespace,
			},
			Client: k8sclientset.CoordinationV1(),
			LockConfig: resourcelock.ResourceLockConfig{
				Identity: id,
			},
		}
    
    // 这只是demo，需要关注当你成为Leader或者失去Leader的时候，你需要执行什么业务代码
		go leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
			Lock:            lock,
			ReleaseOnCancel: true,
			LeaseDuration:   60 * time.Second,
			RenewDeadline:   15 * time.Second,
			RetryPeriod:     5 * time.Second,
			Callbacks: leaderelection.LeaderCallbacks{
				OnStartedLeading: func(ctx context.Context) {
					if err := controller.Run(ctx, threads); err != nil {
						klog.Fatalf("Error to run the controller instance: %s.", err)
					}
					klog.Infof("%s: leading", id)
				},
				OnStoppedLeading: func() {
					controller.Stop()
					klog.Infof("%s: lost", id)
				},
			},
		})
	} else {
		if err := controller.Run(ctx, threads); err != nil {
			klog.Fatalf("Error to run the controller instance: %s.", err)
		}
	}
```

注意上面的Leader选举过程回调函数有下面3种，上面只用了2种

```go
  type LeaderCallbacks struct {
      // OnStartedLeading is called when a LeaderElector client starts leading
      OnStartedLeading func(context.Context)
      // OnStoppedLeading is called when a LeaderElector client stops leading
      OnStoppedLeading func()
      // OnNewLeader is called when the client observes a leader that is
      // not the previously observed leader. This includes the first observed
      // leader when the client starts.
      OnNewLeader func(identity string)
  }
```

# 源码分析

按照上一节的demo，我们来分析下背后的源码流程。

## 结构体

### LeaseLock

demo开始就先实例了一个`LeaseLock`对象，源码位于 `staging/src/k8s.io/client-go/tools/leaderelection/resourcelock/leaselock.go`

```go
type LeaseLock struct {
	// LeaseMeta should contain a Name and a Namespace of a
	// LeaseMeta object that the LeaderElector will attempt to lead.
	LeaseMeta  metav1.ObjectMeta
	Client     coordinationv1client.LeasesGetter
	LockConfig ResourceLockConfig
	lease      *coordinationv1.Lease
}
```

该文件里面还实现了`LeaseLock`的CRUD操作

### LeaderElectionConfig

`RunOrDie()`函数的主要入参就是`LeaderElectionConfig`

```go
type LeaderElectionConfig struct {
	// 资源锁的实现对象
	Lock rl.Interface

	// 非leader 在获取锁之前需要检查 leader 过期的时间，核心的默认是15s
	LeaseDuration time.Duration
  
	// 当前 leader 尝试更新锁状态的期限，Core默认是10s
	RenewDeadline time.Duration
	
	// 抢锁时尝试间隔，Core默认是2s
	RetryPeriod time.Duration

	// 回调函数上面有说明
	Callbacks LeaderCallbacks

	WatchDog *HealthzAdaptor

	// 退出时是否release lock
	ReleaseOnCancel bool

	// Name is the name of the resource lock for debugging
	Name string
}
```



## 流程源码

```go
func RunOrDie(ctx context.Context, lec LeaderElectionConfig) {
	le, err := NewLeaderElector(lec)
	if err != nil {
		panic(err)
	}
	if lec.WatchDog != nil {
		lec.WatchDog.SetLeaderElection(le)
	}
	le.Run(ctx)
}
```

先根据`LeaderElectionConfig`创建了一个叫le的实例，返回了`LeaderElector`结构体

```go
type LeaderElector struct {
	config LeaderElectionConfig
	observedRecord    rl.LeaderElectionRecord //这个结构体保存了选主的信息，重要，k8s里面看到的
	observedRawRecord []byte
	observedTime      time.Time
	reportedLeader string
	clock clock.Clock
	metrics leaderMetricsAdapter
	name string
}
```

然后执行了`Run()`方法

```go
func (le *LeaderElector) Run(ctx context.Context) {
	defer runtime.HandleCrash()
	defer func() {
		le.config.Callbacks.OnStoppedLeading()
	}()

	if !le.acquire(ctx) {
		return // ctx signalled done
	}
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	go le.config.Callbacks.OnStartedLeading(ctx)
	le.renew(ctx)
}
```

里面主要调用了两个方法： `acquire` 和 `renew`。

 `acquire` 和 `renew` 里面核心都是执行`tryAcquireOrRenew`函数，

但是不同的是，`acquire`会通过 `wait.JitterUntil` 来定期调用`tryAcquireOrRenew`，获取锁，直到成功为止，如果获取不到锁，则会以 `RetryPeriod` 为间隔不断尝试。如果获取到锁，就会关闭 stop 通道（ `close(stop)` ），通知 `wait.JitterUntil` 停止尝试；

`renew` 只有在获取锁之后才会调用，它会通过持续更新资源锁的数据，来确保继续持有已获得的锁，保持自己的 leader 状态。这里还是用到了很多 `wait` 包里的方法

最后来看下核心的函数`tryAcquireOrRenew`

```go
func (le *LeaderElector) tryAcquireOrRenew(ctx context.Context) bool {
	now := metav1.Now()
  // 上面提到的锁记录，保存在annotation 中的值，每个节点都将 HolderIdentity 设置为自己
	leaderElectionRecord := rl.LeaderElectionRecord{
  ·····
	}

	//1. 获取或者创建 ElectionRecord
	oldLeaderElectionRecord, oldLeaderElectionRawRecord, err := le.config.Lock.Get(ctx)
	// 获取记录出错，有可能是记录不存在，这种错误需要处理。
  if err != nil {
		if !errors.IsNotFound(err) {
		······
		}
    // 记录不存在的话，则创建一条新的记录
		if err = le.config.Lock.Create(ctx, leaderElectionRecord); err != nil {
    ······
		}
    // 创建记录成功，同时表示获得了锁，返回true
		le.observedRecord = leaderElectionRecord
		le.observedTime = le.clock.Now()
		return true
	}

	// 2. 正常获取了锁资源的记录，检查锁持有者和更新时间。
	if !bytes.Equal(le.observedRawRecord, oldLeaderElectionRawRecord) {
    // 记录之前的锁持有者，其实有可能就是自己。
		le.observedRecord = *oldLeaderElectionRecord
		le.observedRawRecord = oldLeaderElectionRawRecord
		le.observedTime = le.clock.Now()
	}
  // 在满足以下所有的条件下，认为锁由他人持有，并且还没有过期，返回 false
  // a. 当前锁持有者的并非自己
  // b. 上一次观察时间 + 观测检查间隔大于现在时间，即距离上次观测的间隔，小于 `LeaseDuration` 的设置值。  
	if len(oldLeaderElectionRecord.HolderIdentity) > 0 &&
		le.observedTime.Add(le.config.LeaseDuration).After(now.Time) &&
		!le.IsLeader() {
		klog.V(4).Infof("lock is held by %v and has not yet expired", oldLeaderElectionRecord.HolderIdentity)
		return false
	}

  // 3. 更新资源的 annotation 内容。
  // 在本函数开头 leaderElectionRecord 有一些字段被设置成了默认值，这里来设置正确的值。
	if le.IsLeader() {
    // 如果自己持有锁，则继承之前的获取时间和 leader 切换次数
		leaderElectionRecord.AcquireTime = oldLeaderElectionRecord.AcquireTime
		leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions
	} else {
    // 发生 leader 切换，所以 LeaderTransitions + 1
		leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions + 1
	}

	// 更新锁资源对象
	if err = le.config.Lock.Update(ctx, leaderElectionRecord); err != nil {
		klog.Errorf("Failed to update lock: %v", err)
		return false
	}

	le.observedRecord = leaderElectionRecord
	le.observedTime = le.clock.Now()
	return true
}
```

然后，更新本身资源的时候，利用了k8s本身的乐观锁的特性，通过校验`ResourceVersion`的值来保证操作的原子性。

# 附

> ResourceVersion 字段在 Kubernetes 中除了用在上述并发控制机制外，还用在 Kubernetes 的 list-watch 机制中。Client 端的 list-watch 分为两个步骤，先 list 取回所有对象，再以增量的方式 watch 后续对象。Client 端在list取回所有对象后，将会把最新对象的 ResourceVersion 作为下一步 watch 操作的起点参数，也即 Kube-Apiserver 以收到的 ResourceVersion 为起始点返回后续数据，保证了 list-watch 中数据的连续性与完整性。