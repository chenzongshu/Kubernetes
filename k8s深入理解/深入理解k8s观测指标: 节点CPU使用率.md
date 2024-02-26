# 前言

Kubernetes的一个固有问题就是节点的实际使用率不均衡的问题，为了防止节点过热带来的对服务的影响，对节点的资源（CPU/MEM）实际使用率的监控就是构建一套k8s体系必备的功能了。

通常来说，Kubernetes 节点CPU使用率可以来自2个地方，

- `kubectl top` 命令： 比如执行kubectl top node命令得到的节点cpu

> 当然，top命令的监控数据来自于metric server

- `node-exporter`： 从Promethus里面得到的cpu

但是实际使用过程中，发现这两个地方得到的数据不太一样，甚至相差了10%

比如说，kubectl top node {nodeid} 得到的节点cpu使用率为70%的时候，node-exporter得到的节点cpu使用率已经到了80%

这个是为什么呢？ 下面我们来分析这这两组指标的差异性

# node-exporter

## node_cpu_seconds_total

在网上的很多文章和具体实践中，节点的cpu使用率是来自node-exporter的指标 `node_cpu_seconds_total`，这个指标根据里面的mode可以来标记每种CPU的时间，比如mode="idle"代表CPU 的空闲时间

所以，往往我们使用node-exporter计算节点CPU使用率的时候，都是算出空闲的时间占比，再以总数减去该值 ，便可知道CPU的使用率，此处使用irate方法

```
100-avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance)* 100
```

> 现有的服务器一般为多核，所以加上avg求出所有cpu的平均值

mode这个值还有很多，比如user，system，softirq，idle等，和cpu类型一致

那这个值怎么获取的呢？我们从node-exporter源码来寻找答案（node-exporter代码略）

### CPU

node_cpu_seconds_total的源码位于 `collector/cpu_linux.go`

- node-exporter 使用了一个名为 procfs 的第三方库，用于从 /proc 文件系统中读取 CPU 的相关信息。
- node-exporter 在 collector/cpu_linux.go 文件中定义了一个名为 CPUCollector 的结构体，用于实现 Collector 接口，以便注册到 node-exporter 中。
- CPUCollector 结构体包含了一个名为 cpu 的字段，它是一个 prometheus.CounterVec 类型的变量，用于存储 node_cpu_seconds_total 指标的数据。
- CPUCollector 结构体的 Update 方法负责更新 node_cpu_seconds_total 指标的数据，它首先调用 procfs 包的 NewStat 方法，获取当前主机的 CPU 统计信息，然后遍历每个 CPU 的每个模式，将其转换为秒数，并使用 cpu.WithLabelValues 方法为每个标签组合设置相应的值。

可以看到，CPU的user，system，softirq，idle等指标都来自于 `/proc/stat`  文件， 会显示类似

```
cpu  2255 34 2290 22625563 6290 127 456 0 0 0
cpu0 1132 34 1441 11311718 3675 127 438 0 0 0
cpu1 1123 0 849 11313845 2614 0 18 0 0 0
intr 114930548 113199788 3 0 5 263 0 4 0 1 0 0 0 12340 0 9211947 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
ctxt 11546
btime 1628426614
processes 63
procs_running 1
```

其中cpu核后面的数据就对应了CPU各个态的时间，单位是1/100秒，注意这个值是系统启动以来的统计值

```
cpu  2255 34 2290 22625563 6290 127 456 0 0 0
```



# Metric Server

metric server是Kubernetes原生的测量汇聚的组件，我们常用的kubectl top命令就必须安装metric-server

不过metric server本身并不直接读取底层数据，metric-server 是从kubelet中读取的数据并转换为k8s api格式，所以整个数据链路是：

`Kubectl` -> `metric-server` -> `APIServer` -> `kubelet(cadvisor)` -> `cgroup`

- kubectl top其实是发出一个http请求，比如下面的

```go
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes/xxxx |jq .             
kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/xxxx/pods/xxxx |jq .
```

- Metric-server每分钟从kubelet获取node和pod数据，并转换为k8s API格式，暴露给metric API

metric-server用一个结构体来存储，并定期调用scrape和store接口来刷新

```go
//metrics-server/pkg/storage/storage.go

type storage struct {
	mu    sync.RWMutex
	pods  podStorage
	nodes nodeStorage
}
```

Metric-server又是通过kubelet接口 `curl 127.0.0.1:10255/metrics/resource` 来获取数据

- kubelet获取的node和pod数据分为2部分，pod数据来自于cadvisor，node数据来自kubelet本身代码本身的计算

cadvisor取到各个cgroup的数据，注意的是cgroup包含的进程总和是在 `/sys/fs/cgroup/cpu/tasks` 中，即这些进程对应的cpu和mem用量才会被计算

源码中会通过函数 `GetProcessList()` 计算得到进程，然后利用inotify去watch cgroupPath，监控该目录下的变更，拿到该目录下的增删改查事件，也就知道container的变更，从而动态更新cache中的数据





# 两者差异

回归到前面的问题，为什么两者有差异性呢？

从OS和Cgroup的角度，就很好的得到了答案

node-exporter的CPU使用率，是用的`1-idle`，即相当于`usr+sys+softirq+hardirq+iowait`

Metric-server的CPU使用率，来自cgroup，但是cgroup只包含了 `usr+sys `，从`cpuacct.stat`文件可以看到

```
#cat cpuacct.stat
user 3558269764
system 590838942
```

这样，随着系统IO和中断时间增加，可能这差值会进一步变大

