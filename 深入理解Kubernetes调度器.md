> https://www.infoq.cn/article/or7crphtdlx1ivhsfngk
>
> https://www.cnblogs.com/luozhiyun/p/13619296.html
>
> https://www.qikqiak.com/post/custom-kube-scheduler/



# 调度器

k8s最资源的调度能力是它有别于IaaS平台的能, 它的调度就是通过kube-scheduler来实现的, kube-scheduler是k8s关键组件之一



## 简单调度流程

```
=> 创建Pod
-> APIServer创建Pod(nodeName 是空的，并且它的 phase 是 Pending 状态)
-> kube-Scheduler 以及 kubelet 都能 watch 到这个 pod 的生成事件，kube-Scheduler发现这个 pod 的 nodeName 是空的之后，会认为这个 pod 是处于未调度状态
-> 经过"预选","优选"两大阶段, 找到一个得分最高的节点, 然后把Node绑定到pod 的 spec
-> kubelet watch 到这个 pod 是属于自己节点上的一个 pod
<= kubelet创建Pod, 包括storage以及network，最后等所有的资源都完成，kubelet会把状态更新为Running
```

## Predicates预选

### 节点预选

这一步主要是为了做性能优化. 早期k8s把所有可用节点都加载到预选调度过程中. 但是节点太多了就非常吃资源.比如如果集群有5000节点, 为了调度一个Pod就会在5000节点做计算.

所以scheduler后期就通过PercentageOfNodesToScore机制优化了预选调度过程中的性能。该机制的原理为：一旦发现一定数量的可用节点（占所有节点的百分比），调度器就停止寻找更多的可用节点，这样可以提升大型Kubernetes集群中调度器的性能。

PercentageOfNodesToScore参数值是一个集群中所有节点的百分比，范围在1和100之间。默认值为50%



# 自定义调度器

一般来说，我们有4种扩展 Kubernetes 调度器的方法。

- 一种方法就是直接 clone 官方的 kube-scheduler 源代码，在合适的位置直接修改代码，然后重新编译运行修改后的程序，当然这种方法是最不建议使用的，也不实用，因为需要花费大量额外的精力来和上游的调度程序更改保持一致。
- 第二种方法就是和默认的调度程序一起运行独立的调度程序，默认的调度器和我们自定义的调度器可以通过 Pod 的 `spec.schedulerName` 来覆盖各自的 Pod，默认是使用 default 默认的调度器，但是多个调度程序共存的情况下也比较麻烦，比如当多个调度器将 Pod 调度到同一个节点的时候，可能会遇到一些问题，因为很有可能两个调度器都同时将两个 Pod 调度到同一个节点上去，但是很有可能其中一个 Pod 运行后其实资源就消耗完了，并且维护一个高质量的自定义调度程序也不是很容易的，因为我们需要全面了解默认的调度程序，整体 Kubernetes 的架构知识以及各种 Kubernetes API 对象的各种关系或限制。
- 第三种方法是[调度器扩展程序](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/scheduler_extender.md)，这个方案目前是一个可行的方案，可以和上游调度程序兼容，所谓的调度器扩展程序其实就是一个可配置的 Webhook 而已，里面包含 `过滤器` 和 `优先级` 两个端点，分别对应调度周期中的两个主要阶段（过滤和打分）。
- 第四种方法是通过调度框架（Scheduling Framework），Kubernetes v1.15 版本中引入了可插拔架构的调度框架，使得定制调度器这个任务变得更加的容易。调库框架向现有的调度器中添加了一组插件化的 API，该 API 在保持调度程序“核心”简单且易于维护的同时，使得大部分的调度功能以插件的形式存在，而且在 v1.16 版本中上面的 `调度器扩展程序` 也已经被废弃了，所以以后调度框架才是自定义调度器的核心方式。

