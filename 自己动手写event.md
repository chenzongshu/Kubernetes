# 基础知识

## 什么是event

Event 是集群中某个事件的报告。它一般表示系统的某些状态变化。存储在etcd里面。

下面是个简单的例子，随便写了一个不存在的镜像，pod event报错

```bash
kubectl -n moelove describe pods test-d9ddbdd84-tnrhd
...
Events:
  Type     Reason     Age                    From               Message
  ----     ------     ----                   ----               -------
  Normal   Scheduled  4m                     default-scheduler  Successfully assigned moelove/non-exist-d9ddbdd84-tnrhd to kind-worker3
  Normal   Pulling    2m22s (x4 over 3m59s)  kubelet            Pulling image "test::latest"
  Warning  Failed     2m21s (x4 over 3m59s)  kubelet            Failed to pull image "test::latest": rpc error: code = Unknown desc = failed to pull and unpack image "test:latest": failed to resolve reference "test::latest": failed to authorize: failed to fetch anonymous token: unexpected status: 403 Forbidden
  Warning  Failed     2m21s (x4 over 3m59s)  kubelet            Error: ErrImagePull
  Warning  Failed     2m9s (x6 over 3m58s)   kubelet            Error: ImagePullBackOff
  Normal   BackOff    115s (x7 over 3m58s)   kubelet            Back-off pulling image "test:lastest"
```

Reson是事件的原因，而Message是事件的具体内容，是用户可以理解的语句

其他都很好理解，但是类似 `115s (x7 over 3m58s)` 这样的输出是什么意思呢？

这说明 **该类型的 event 在 3m58s 中已经发生了 7 次，最近的一次发生在 115s 之前**，

但是当我们去直接 `kubectl get events` 的时候，我们并没有看到有 7 次重复的 event 。这说明 **Kubernetes 会自动将重复的 events 进行合并**

## 关键特性

### 保存时间

一般来说，只保存1小时，这是因为集群内事件太多

### 命名空间相关性

event是和namespace相关的

### 资源相关性

下面是我们经常查看event的操作

```bash
## 对deployment
kubectl -n moelove describe deploy/redis                
Name:                   redis
Namespace:              moelove
...
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  15m   deployment-controller  Scaled up replica set redis-687967dbc5 to 1

## 对pod
kubectl -n moelove describe pods redis-687967dbc5-27vmr
Name:         redis-687967dbc5-27vmr                                                                 
Namespace:    moelove
Priority:     0
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  18m   default-scheduler  Successfully assigned moelove/redis-687967dbc5-27vmr to kind-worker3
  Normal  Pulling    18m   kubelet            Pulling image "ghcr.io/moelove/redis:alpine"
  Normal  Pulled     17m   kubelet            Successfully pulled image "ghcr.io/moelove/redis:alpine" in 6.814310968s
  Normal  Created    17m   kubelet            Created container redis
  Normal  Started    17m   kubelet            Started container redis
```

注意到没有，对deployment和下面的pod，看到的event是不同的

所以，event是和资源对象相关的，describe的时候，只能看到和自己相关的

## 查看event的高级用法

### 按时间排序

默认情况下 `kubectl get event` 并没有按照events发生的时间进行排序

```bash
kubectl -n test get events --sort-by='{.metadata.creationTimestamp}'
LAST SEEN   TYPE     REASON              OBJECT                        MESSAGE
2m12s       Normal   Scheduled           pod/redis-687967dbc5-27vmr    Successfully assigned redis-687967dbc5-27vmr to kind-worker3
2m13s       Normal   SuccessfulCreate    replicaset/redis-687967dbc5   Created pod: redis-687967dbc5-27vmr
2m13s       Normal   ScalingReplicaSet   deployment/redis              Scaled up replica set redis-687967dbc5 to 1
2m12s       Normal   Pulling             pod/redis-687967dbc5-27vmr    Pulling image "redis:alpine"
2m6s        Normal   Pulled              pod/redis-687967dbc5-27vmr    Successfully pulled image "redis:alpine" in 6.814310968s
2m5s        Normal   Created             pod/redis-687967dbc5-27vmr    Created container redis
2m5s        Normal   Started             pod/redis-687967dbc5-27vmr    Started container redis
```

### 查看一条event详细内容

正常情况下是看不到event的ID的，但是event也是k8s的资源，也有metadata.Name

```bash
kubectl get events --sort-by='{.metadata.creationTimestamp}' -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
redis-687967dbc5-27vmr.16c4fb7bde8c69d2
redis-687967dbc5.16c4fb7bde6b54c4
redis.16c4fb7bde1bf769
redis-687967dbc5-27vmr.16c4fb7bf8a0ab35
redis-687967dbc5-27vmr.16c4fb7d8ecaeff8
redis-687967dbc5-27vmr.16c4fb7d99709da9
redis-687967dbc5-27vmr.16c4fb7d9be30c06
```

选择其中一条

```bash
kubectl get events redis-687967dbc5-27vmr.16c4fb7bde8c69d2 -o yaml
action: Binding
apiVersion: v1
eventTime: "2021-12-28T19:31:13.702987Z"
firstTimestamp: null
involvedObject:
  apiVersion: v1
  kind: Pod
  name: redis-687967dbc5-27vmr
  namespace: default
  resourceVersion: "330230"
  uid: 71b97182-5593-47b2-88cc-b3f59618c7aa
kind: Event
lastTimestamp: null
message: Successfully assigned redis-687967dbc5-27vmr to kind-worker3
metadata:
  creationTimestamp: "2021-12-28T19:31:13Z"
  name: redis-687967dbc5-27vmr.16c4fb7bde8c69d2
  namespace: moelove
  resourceVersion: "330235"
  uid: e5c03126-33b9-4559-9585-5e82adcd96b0
reason: Scheduled
reportingComponent: default-scheduler
reportingInstance: default-scheduler-kind-control-plane
source: {}
type: Normal

```

- involvedObject： 触发event的资源类型
- lastTimestamp：最后一次触发的时间
- message：事件说明
- metadata :event的元信息，name，namespace等
- reason：event的原因
- source：上报事件的来源，比如kubelet中的某个节点
- type:事件类型，Normal或Warning

# 怎么写event

我们在代码中，如果需要往k8s写入event事件，该怎么来写？ 下面基于client-go，来做一个示例， 其实很简单，核心就是client-go的record库

```go
import {
    clientcorev1 "k8s.io/client-go/kubernetes/typed/core/v1"
    "k8s.io/client-go/kubernetes/scheme"
    "k8s.io/client-go/tools/record"

	eventBroadcaster := record.NewBroadcaster()
	eventBroadcaster.StartStructuredLogging(3)
	eventBroadcaster.StartRecordingToSink(&clientcorev1.EventSinkImpl{Interface: client.CoreV1().Events(pod.Namespace)})
	r := eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "descheduler"})
	r.Event(pod, v1.EventTypeNormal, "Descheduled", fmt.Sprintf("pod evicted by descheduler%s", "test reason for czs"))
}
```

-  `eventBroadcaster` 是个事件广播器，

- `StartLogging` 和 `StartRecordingToSink` 创建了两个不同的事件处理函数，分别把事件记录到日志和发送给 apiserver。

-  `NewRecorder` 新建了一个 `Recoder` 对象，通过它的 `Event`、`Eventf` 和 `PastEventf` 方法，用户可以往里面发送事件，`eventBroadcaster` 会把接收到的事件发送个多个处理函数，比如这里提到的写日志和发送到 apiserver

需要注意的点：

- `StartRecordingToSink()`输入的namespace要存在，不然会抛错退出

- `Event()` 的入参是我们用的最多的：
  
  - 第一个参数是资源类型，比如这里的`pod` 是Kubernetes的数据类型，这里是 `*v1.Pod` 
  
  - 第二个是事件的类型，注意k8s事件只有2种类型，Normal和Warning，对应的参数是`v1.EventTypeNormal` 和 `v1.EventTypeWarning`
  
  - 第三个参数是Reason，字符串类型，含义在开头有说
  
  - 第四个参数是Message，字符串类型，含义在开头有说

    
