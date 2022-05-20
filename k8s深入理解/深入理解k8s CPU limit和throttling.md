# 前言

k8s的基础知识不在此赘述，大家都知道在pod内，如果cpu使用率达到limit限制，就会产生cpu限流。但是在实际生产中，有时候即使cpu使用率离limit还比较远的时候，也会产生cpu限流，从而导致服务rt升高，服务无法响应，健康检查受影响等

下面来分析下原理

# Kubernetes CPU Limit

Kubernetes使用cgroup，内核的throttling来做CPU限流的。以一个4C的宿主机为例

## Cpu Request

root的cgroup在`/sys/fs/cgroup/cpu,cpuacct/*` ，而k8s用 `cpu.share`来分配CPU资源。

在一个典型的k8s节点，下面分了3个cgroup：

- `system.slice`：系统负载

-  `user.slice`：非k8s的用户态程序

- `kubepods`：给k8s的pod的资源

实际看一个节点，一个4C的节点总CPU share为4096(1 core=1024)，上面的`system.slice`, `user.slice`都是1024，`kubepod` 有 **4096**，是不是迷糊了，这加起来6144，超过4096了?

实际上，CFS是根据比例来计算的。即前面2个cgroup每个得到了680（4096的16.6%，就是1024/6144），而`kubepod` 得到2736。

但在空闲情况下，前两个 cgroup 不会使用所有分配的资源。调度程序有一种机制来避免浪费未使用的 CPU 份额。调度程序将未使用的 CPU 释放到全局池，以便它可以分配给需要更多 CPU 功率的 cgroup

**k8s的Request对应cpu_shares**

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

