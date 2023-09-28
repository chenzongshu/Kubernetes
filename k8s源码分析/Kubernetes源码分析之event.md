# 介绍

Event 是集群中某个事件的报告。它一般表示系统的某些状态变化。存储在etcd里面。

从kubelet里面， `RunKubelet` 中会初始化 `EventBroadcaster` 和 `Recorder`

涉及到下面组件

● `EventRecorder`：事件（Event）生产者，也称为事件记录器Kubernetes系统组件通过EventRecorder记录关键性事件。
● `EventBroadcaster`：事件（Event）消费者，也称为事件广播器。EventBroadcaster消费EventRecorder记录的事件并将其分发给目前所有已连接的broadcasterWatcher。分发过程有两种机制，分别是非阻塞（Non-Blocking）分发机制和阻塞（Blocking）分发机制。
● `broadcasterWatcher`：观察者（Watcher），用于定义事件的处理方式，例如上报事件至Kubernetes API Server。

# Event写入流程

```bash
                                                     ┌──────────────┐
                                                     │eventHandler: │
                                              ┌─────▶│     logf     │
                                              │      └──────────────┘
┌─────────────┐   Event()   ┌───────────┐     │                      
│EventRecorder│────────────▶│  channel  │─────┤                      
└─────────────┘             └───────────┘     │      ┌──────────────┐
                                              │      │eventHandler: │
                                              └─────▶│ recordToSink │
                                    StartEventWatcher ──────────────┘
                                                             │       
                                                             │       
                                                             ▼       
┌────────┐      ┌───────────┐     ┌───────────┐      ┌──────────────┐
│  Etcd  │◀──── │ APIServer │◀────│recordEvent│◀─────│EventCorrelate│
└────────┘      └───────────┘     └───────────┘      └──────────────┘
```

# EventRecorder 和 EventBroadcaster

`cmd/kubelet/app/server.go`

```go
func makeEventRecorder(kubeDeps *kubelet.Dependencies, nodeName types.NodeName) {
    if kubeDeps.Recorder != nil {
        return
    }
    eventBroadcaster := record.NewBroadcaster()
    kubeDeps.Recorder = eventBroadcaster.NewRecorder(legacyscheme.Scheme, v1.EventSource{Component: componentKubelet, Host: string(nodeName)})
    eventBroadcaster.StartStructuredLogging(3)
    if kubeDeps.EventClient != nil {
        klog.V(4).InfoS("Sending events to api server")
        eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: kubeDeps.EventClient.Events("")})
    } else {
        klog.InfoS("No api server defined - no events will be sent to API server")
    }
}
```

`eventBroadcaster` 是个事件广播器，`StartLogging` 和 `StartRecordingToSink` 创建了两个不同的事件处理函数，分别把事件记录到日志和发送给 apiserver。

而 `NewRecorder` 新建了一个 `Recoder` 对象，通过它的 `Event`、`Eventf` 和 `PastEventf` 方法，用户可以往里面发送事件，`eventBroadcaster` 会把接收到的事件发送个多个处理函数，比如这里提到的写日志和发送到 apiserver。

## EventBroadcaster

### 初始化

**EventBroadcaster** 本身的定义

```go
type Broadcaster struct {
    watchers     map[int64]*broadcasterWatcher
    nextWatcher  int64
    distributing sync.WaitGroup

    incomingBlock sync.Mutex
    incoming      chan Event
    stopped       chan struct{}

    watchQueueLength int

    fullChannelBehavior FullChannelBehavior
}
```

有下面如下方法：

```go
type EventBroadcaster interface {
    // 将从 EventBroadcaster 接收到的 events 丢给 eventHandler
    StartEventWatcher(eventHandler func(*v1.Event)) watch.Interface

    // 将从 EventBroadcaster 接收到的 events 丢给 EventSink
    StartRecordingToSink(sink EventSink) watch.Interface

    // 将从 EventBroadcaster 接收到的 events 丢给指定的日志函数    StartLogging(logf func(format string, args ...interface{})) watch.Interface
    StartStructuredLogging(verbosity klog.Level) watch.Interface

    // 用于获取 EventRecorder，EventRecorder 可以发送 events 给 EventBroadcaster
    NewRecorder(scheme *runtime.Scheme, source v1.EventSource) EventRecorder

    // Shutdown shuts down the broadcaster
    Shutdown()
}
```

**EventBroadcaster** 对应的实现是 **EventBroadcasterImpl**

```go
func NewBroadcaster() EventBroadcaster {
    return &eventBroadcasterImpl{
        Broadcaster:   watch.NewLongQueueBroadcaster(maxQueuedEvents, watch.DropIfChannelFull),
        sleepDuration: defaultSleepDuration,
    }
}
```

其中` NewLongQueueBroadcaster()` 会初始化一个 `Broadcaster` 

然后通过`m.loop()`启动goroutine来监控 `m.incoming`，里面有调用 `distributing(event)`函数分发给所有已经连接的broadcasterWatcher，分发过程有两种机制，分别是非阻塞分发机制和阻塞分发机制。在非阻塞分发机制下使用DropIfChannelFull标识，在阻塞分发机制下使用WaitIfChannelFull标识，默认为DropIfChannelFull标识

```go
func NewLongQueueBroadcaster(queueLength int, fullChannelBehavior FullChannelBehavior) *Broadcaster {
    m := &Broadcaster{
······
    }
    m.distributing.Add(1)
    go m.loop()
    return m
}
```

### StartRecordingToSink

`StartRecordingToSink` 和 `StartStructuredLogging` 都是一个Interface，最后都调用到`StartEventWatcher(eventHandler)`

`StartEventWatcher()`运行了一个goroutine，用于不断监控EventBroadcaster来发现事件并调用相关函数对事件进行处理

```go
func (e *eventBroadcasterImpl) StartEventWatcher(eventHandler func(event runtime.Object)) func() {
    watcher := e.Watch()
    go func() {
        defer utilruntime.HandleCrash()
        for {
            //通过上看可以看出ResultChan 是拿到了f.result 也就是说当在这个管道写入的时候这里就会获取到
            watchEvent, ok := <-watcher.ResultChan()
            if !ok {
                return
            }
            eventHandler(watchEvent.Object)
        }
    }()
    return watcher.Stop
}
```

## EventRecorder
