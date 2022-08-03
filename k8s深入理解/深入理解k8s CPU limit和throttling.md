# 前言

k8s的基础知识不在此赘述，大家都知道在pod内，如果cpu使用率达到limit限制，就会产生cpu限流。但是在实际生产中，有时候即使cpu使用率离limit还比较远的时候，也会产生cpu限流，从而导致服务rt升高，服务无法响应，健康检查受影响等

下面来分析下原理

# 基础知识

## Cgroup

cgroup是对指定进程做计算资源限制的。CPU Cgroup是Cgroups的子系统，专门限制进程的CPU， CPU Cgroup相关信息都在` /sys/fs/cgroup/cpu `子文件夹下面

比如，我们进入 `kubepods` 这个控制组，将会看到

```bash
ls cpu.*
cpu.cfs_period_us  cpu.cfs_quota_us  cpu.rt_period_us  cpu.rt_runtime_us  cpu.shares  cpu.stat
```

- **cpu.cfs_period_us**： CFS 算法的一个调度周期，一般它的值是 100000，单位us，也就 100ms

- **cpu.cfs_quota_us**：它“表示 CFS 算法中，在一个调度周期里这个控制组 被允许的运行时间，比如这个值为 50000 时，就是 50ms
  
  如果用这个值去除以调度周期（也就是 cpu.cfs_period_us），50ms/100ms = 0.5，这样 这个控制组被允许使用的 CPU 最大配额就是 0.5 个 CPU。如果超过了 1 个 CPU，这就意味着这时控制组需要 2 个 CPU 的资源配额。

- **cpu.shares**：这个值是 CPU Cgroup 对于控制组之间的 CPU 分配比例，它的缺省值是 1024

> 意味着：
> 
> - cpu.cfs_quota_us 和 cpu.cfs_period_us 这两个值决定了**每个控制组中所有进程的可使用 CPU资源的最大值**
> 
> - cpu.shares 这个值决定了 **CPU** **Cgroup** **子系统下控制组可用** **CPU** **的相对比例**， 不过只有当系统上 CPU 完全被占满的时候，这个比例才会在各个控制组间起作用

## 容器中CPU的开销

### 容器中的top命令靠谱吗？

top命令中"%Cpu(s)"那一行中显示的数值，并不是这个容器的 CPU 整体使用率，而是**容器 宿主机的 CPU 使用率**。进程CPU利用率是准确的

为什么这样呢？这还要从Linux的CPU利用率计算说起

#### 进程CPU利用率

如果我们要计算某个进程的CPU利用率，可以查看proc 文件系统中每个 进程对应的 stat 文件 `/proc/[pid]/stat` ，这里面是一堆数字，实时输出了进程的状态信息，比如进程的运行态（Running 还是 Sleeping）、父进程 PID、进程优先级、进程使用的内存等等总共 50 多项。具体含义见《Linux programmer’s manual》手册

```
1 (systemd) S 0 1 1 0 -1 4202752 86841827 651644134 55 1213 1316706 590418 8281850 4103654 20 0 1 0 6 46731264 1463 18446744073709551615 94140720173056 94140721628802 140735870004784 140735870001416 139723897876403 0 671173123 4096 1260 18446744072416252254 0 0 17 2 0 0 203 0 0 94140723728696 94140723873336 94140740952064 140735870013340 140735870013407 140735870013407 140735870013407 0
```

我们需要关注的是stat 文件中的第 14 项 utime 和第 15 项 stime。utime 是表示进程的用户态部分在 Linux 调度中获得 CPU 的 ticks，stime 是表示进程的内核态部分在 Linux 调度中获得 CPU 的 ticks

> - 这个 ticks 就是 Linux 操作 系统中的一个时间单位，可以理解成类似秒，毫秒的概念。在 Linux 中有个自己的时钟，它会周期性地产生中断。每次中断都会触发 Linux 内核去做 一次进程调度，而这一次中断就是一个 tick。因为是周期性的中断，比如 1 秒钟 100 次中 断，那么一个 tick 作为一个时间单位看的话，也就是 1/100 秒。
> 
> - **utime 和 stime 都是一个累计值，也就是说从进程启动开始，这两个值 就是一直在累积增长的**

所以进程在一段时间获取的总CPU tick就是 `(utime_2 – utime_1) + (stime_2 – stime_1)`

**进程的 CPU 使用率 =((utime_2 – utime_1) + (stime_2 – stime_1)) * 100.0 / (HZ * et * 1 )**

> - HZ：ticks 是按照固定频率发生的，在 我们的 Linux 系统里 1 秒钟是 100 次，那么 HZ 就是 1 秒钟里 ticks 的次数，这里值是 100。
> 
> - et：是我们刚才说的那个“瞬时”的时间，也就是得到 utime_1 和 utime_2 这两个值的时间间隔。
> 
> - “1”, 就是 1 个 CPU。那么这三个值相乘，就是在这“瞬时”的时间（et）里，1 个 CPU 所包含的 ticks 数目
> 
> 进程的 CPU 使用率 =（进程的 ticks/ 单个 CPU 总 ticks）*100.0

#### 系统CPU利用率

整个系统的 CPU 使用率，可以从 proc 文件系统里得到，文件是`/proc/stat`，但是这个文 件中的各项 CPU ticks 是反映整个节点的，并且这个 /proc/stat 文件也不包含在任意一个 Namespace 里

> /proc/stat 数值和进程的不同，可以查看手册

### 怎么获取容器中的CPU利用率

容器的cgroup下面还有一个 `cpuacct.stat`文件

```bash
cat cpuacct.stat
user 38782674
system 12075281
```

这两个值就是**这个cgroup里所有进程的内核态 ticks 和用户态 的 ticks**，所以就可以套上前面的公式来计算容器的CPU利用率了

## Load Average

Load Average其实是CPU的另外一种度量，Load Average后面三个数值分别代表过去 1 分钟，5 分钟，15 分钟在这个节点上的 Load Average

```
Load Average = 可运行队列进程平均数 + D状态进程数
//D状态进程：TASK_UNINTERRUPTIBLE进程，表示进程为了等待某个系统资源而进入睡眠状态
```

> TASK_UNINTERRUPTIBLE 状态的进程不一定是做I/O, 比如等待信号量的进程也会进 入到这个状态

# Kubernetes CPU Limit

Kubernetes使用cgroup，内核的throttling来做CPU限流的。以一个4C的宿主机为例

## Cpu Request

root的cgroup在`/sys/fs/cgroup/cpu,cpuacct/*` ，而k8s用 `cpu.share`来分配CPU资源。

在一个典型的k8s节点，下面分了3个cgroup：

- `system.slice`：系统负载

- `user.slice`：非k8s的用户态程序

- `kubepods`：给k8s的pod的资源

实际看一个节点，一个4C的节点总CPU share为4096(1 core=1024)，上面的`system.slice`, `user.slice`都是1024，`kubepod` 有 **4096**，是不是迷糊了，这加起来6144，超过4096了?

实际上，CFS是根据比例来计算的。即前面2个cgroup每个得到了680（4096的16.6%，就是1024/6144），而`kubepod` 得到2736。

但在空闲情况下，前两个 cgroup 不会使用所有分配的资源。调度程序有一种机制来避免浪费未使用的 CPU 份额。调度程序将未使用的 CPU 释放到全局池，以便它可以分配给需要更多 CPU 功率的 cgroup

**k8s的Request对应cpu_shares**，一般来说，cpu.shares==1024表示1个CPU比例，那么Request CPU的值是n，给cpu.shares 就是 n*1024

## Cpu Limit

limit和Request完全不同，k8s使用CFS配额机制来限制limit。配置在 `cfs_period_us` and `cfs_quota_us`文件里面。

和cpu.share不同的是，配额是基于周期而不是基于CPU能力。

- `cfs_period_us`用于定义时间段，始终为 100000us（100ms）。
- `cfs_quota_us`用于表示配额期内允许的配额。通常Pod的`cfs_quota_us`对应容器的CPU Limit，比如一个CPU Limit=2的容器，cfs_quota_us 会被设置为200ms，表示该容器在每 100ms 的时间周期内最多使用 200ms 的 CPU 时间片，即 2 个 CPU 核心

# 检查Pod CPU限流

只需登录到 pod 并运行`cat /sys/fs/cgroup/cpu/cpu.stat`

- `nr_periods`— 总调度周期
- `nr_throttled`— nr_periods 中的总限制期
- `throttled_time`— 以 ns 为单位的总限流时间

# CPU Throttling 原因

主要可能有2个原因，1个是因为在一个CPU配额周期内CPU耗尽，2是因为受CPU拓扑结构影响，部分应用对CPU上下文切换比较敏感，尤其是跨NUMA访问时

## cfs_period

这个比较好理解，比如上面说的 cfs_period 的100ms内，某个pod的limit设置为2C，的cfs_quota有200ms，但是来了突发性的CPU资源需求，比如流量突增。

假如是个web服务，假设每个请求要60ms，即使容器最近整体CPU利用率很低，但是在100ms内来了4个请求，需要花240ms来处理，已经超过预算200ms了，就需求等到下个周期才能处理，这样就导致RT上升了。

我们只能将容器的 CPU Limit 值调大。然而，若想彻底解决 CPU Throttle，通常需要将 CPU Limit 调大两三倍，有时甚至五到十倍，问题才会得到明显缓解。而为了降低 CPU Limit 超卖过多的风险，还需降低容器的部署密度，进而导致整体资源成本上升

## CPU拓扑结构影响

在 NUMA 架构下，节点中的 CPU 和内存会被切分成了两部分甚至更多，CPU 被允许以不同的速度访问内存的不同部分，当 CPU 跨 Socket 访问另一端内存时，其访存时延相对更高。盲目地在节点为容器分配物理资源可能会降低延迟敏感应用的性能，因此我们需要避免将 CPU 分散绑定到多个 Socket 上，提升内存访问时的本地性。

Kubelet 提供的 CPU 管理策略 “static policy”、以及拓扑管理策略 “single-numa-node”，会将容器与 CPU 绑定，可以提升应用负载与 CPU Cache，以及 NUMA 之间的亲和性，但是损失了多核下的CPU资源弹性。
