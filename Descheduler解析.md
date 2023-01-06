# Descheduler介绍

## What

辅助kube-scheduler，对POD进行定期的二次打散整理，确保集群尽量调度均衡。

## When

三种模式，Job，CronJob，Deployment

Job/CronJob 很好理解，执行的时间也可控

但是Deployment的执行时间？

## How

通过设置策略来控制做哪些驱逐逻辑，但是可以过滤namespace、节点和类型

目前有10种策略，最有用就下面几种

### RemoveDuplicates

该策略确保只有一个和 Pod 关联的 RS、Deployment 或者 Job 资源对象运行在同一节点上。如果还有更多的 Pod 则将这些重复的 Pod 进行驱逐，以便更好地在集群中分散 Pod。

### LowNodeUtilization

如果node上的cpu/memory/pods的request超过targetThresholds阈值，则认为该node负载太高，会尝试将其上的pod进行迁移，迁移的目标node需要低于thresholds阈值，这些node被认为是负载太低。

注意：

- thresholds的判断是必须cpu+mem+pod三个条件（都是百分比）**同时满足**才算作”低负载node”，其cpu/memory依据是request值。

- targetThresholds判断依据是cpu or mem or pod任意一个超过阈值才算作”高负载node”。

- pod会从”高负载node” 迁移至 “低负载node”，不满足2个条件的node不会做均衡处理。

- 如果“高负载node”或者“低负载node”任一个数量为0，这个策略立即被终止

### HighNodeUtilization

找到未充分利用的节点并将 Pod 从节点中逐出，以期将这些 Pod 紧凑地调度到更少的节点中。与节点自动缩放结合使用，此策略旨在帮助触发未充分利用的节点的缩减。此策略**必须**与调度程序评分策略`MostAllocated`一起使用。

其他和上面的LowNodeUtilization相同，只是驱逐的对象不同，该策略驱逐“低负载”节点

## Notice

- 关键性 Pod 不会被驱逐，比如 priorityClassName 设置为 system-cluster-critical 或 system-node-critical 的 Pod
- 不属于 RS、Deployment 或 Job 管理的 Pods 不会被驱逐
- DaemonSet 创建的 Pods 不会被驱逐
- 使用 LocalStorage 的 Pod 不会被驱逐，除非设置 evictLocalStoragePods: true
- 具有 PVC 的 Pods 不会被驱逐，除非设置 ignorePvcPods: true
- 在 LowNodeUtilization 和 RemovePodsViolatingInterPodAntiAffinity 策略下，Pods 按优先级从低到高进行驱逐，如果优先级相同，Besteffort 类型的 Pod 要先于 Burstable 和 Guaranteed 类型被驱逐
- annotations 中带有 descheduler.alpha.kubernetes.io/evict 字段的 Pod 都可以被驱逐，该注释用于覆盖阻止驱逐的检查，用户可以选择驱逐哪个Pods
- 如果 Pods 驱逐失败，可以设置 --v=4 从 descheduler 日志中查找原因，如果驱逐违反 PDB 约束，则不会驱逐这类 Pods

## Question

Deployment部署时的执行时间

- 在启动参数里面指定执行间隔

```
          command:
            - "/bin/descheduler"
          args:
            - "--policy-config-file"
            - "/policy-dir/policy.yaml"
            - "--descheduling-interval"
            - "5m"
            - "--v"
            - "3"
```

LowNodeUtilization能保证一定从高负载到低负载？

> 这应该就是为什么使用Request作为依据，这样就能和kube-scheduler规则保持一致，保证到“低负载”节点，如果我们换成实际CPU利用率，得考虑这部分偏差

# 源码分析

## 核心结构体

### 进程相关结构体

```go
type DeschedulerServer struct {
    componentconfig.DeschedulerConfiguration

    Client         clientset.Interface
    Logs           *logs.Options
    SecureServing  *apiserveroptions.SecureServingOptionsWithLoopback
    DisableMetrics bool
}
```

```go
type DeschedulerConfiguration struct {
    metav1.TypeMeta

    // deployment部署时候的执行周期
    DeschedulingInterval time.Duration

    KubeconfigFile string

    // PolicyConfigFile is the filepath to the descheduler policy configuration.
    PolicyConfigFile string

    // Dry run
    DryRun bool

    // Node selectors
    NodeSelector string

    // MaxNoOfPodsToEvictPerNode restricts maximum of pods to be evicted per node.
    MaxNoOfPodsToEvictPerNode int

    // EvictLocalStoragePods allows pods using local storage to be evicted.
    EvictLocalStoragePods bool

    // IgnorePVCPods sets whether PVC pods should be allowed to be evicted
    IgnorePVCPods bool

    // Logging specifies the options of logging.
    // Refer [Logs Options](https://github.com/kubernetes/component-base/blob/master/logs/options.go) for more information.
    Logging componentbaseconfig.LoggingConfiguration
}
```

### 策略

```go
type DeschedulerStrategy struct {
    // Enabled or disabled
    Enabled bool

    // Weight
    Weight int

    Params *StrategyParameters
}

//策略参数，基本这些参数对应了可以配置的各种策略
type StrategyParameters struct {
    NodeResourceUtilizationThresholds *NodeResourceUtilizationThresholds
    NodeAffinityType                  []string
    PodsHavingTooManyRestarts         *PodsHavingTooManyRestarts
    PodLifeTime                       *PodLifeTime
    RemoveDuplicates                  *RemoveDuplicates
    FailedPods                        *FailedPods
    IncludeSoftConstraints            bool
    Namespaces                        *Namespaces
    ThresholdPriority                 *int32
    ThresholdPriorityClassName        string
    LabelSelector                     *metav1.LabelSelector
    NodeFit                           bool
}
```

## 启动

常规的 `cmd/descheduler/descheduler.go`中的 `main` 函数

`NewDeschedulerCommand()` 做cobra初始化参数接收，走 `run()` -> `RunDeschedulerStrategies()`

### Run

`Run()`里面就初始化了clientset和策略

```go
func Run(rs *options.DeschedulerServer) error {
······
    rsclient, err := client.CreateClient(rs.KubeconfigFile)
······
    deschedulerPolicy, err := LoadPolicyConfig(rs.PolicyConfigFile)
······
    stopChannel := make(chan struct{})
    return RunDeschedulerStrategies(ctx, rs, deschedulerPolicy, evictionPolicyGroupVersion, stopChannel)
}
```

### RunDeschedulerStrategies

`RunDeschedulerStrategies()`就是流程的主要函数，核心就是一个循环函数，周期就是启动命令里面配置的周期

```go
wait.Until(func() {
······
}, rs.DeschedulingInterval, stopChannel)
```

最主要的循环，就是去自定义策略列表里面循环，然后去 `strategyFuncs` 里面去匹配，`strategyFuncs`里面注册了回调函数，然后如果策略是enable就直接调用该回调函数

```go
        for name, strategy := range deschedulerPolicy.Strategies {
            if f, ok := strategyFuncs[name]; ok {
                if strategy.Enabled {
                    f(ctx, rs.Client, strategy, nodes, podEvictor)
                }
            } else {
                klog.ErrorS(fmt.Errorf("unknown strategy name"), "skipping strategy", "strategy", name)
            }
        }
```

下面以某一个规则举例，来看看里面的规则处理流程

### LowNodeUtilization()

```go
func LowNodeUtilization(ctx context.Context, client clientset.Interface, strategy api.DeschedulerStrategy, nodes []*v1.Node, podEvictor *evictions.PodEvictor) {
    // 判断参数是否合理
    if err := validateNodeUtilizationParams(strategy.Params); err != nil {
······
    thresholdPriority, err := utils.GetPriorityFromStrategyParams(ctx, client, strategy.Params)
······
    // 和获取门限值，下面还有一些初始化赋值
    thresholds := strategy.Params.NodeResourceUtilizationThresholds.Thresholds
    targetThresholds := strategy.Params.NodeResourceUtilizationThresholds.TargetThresholds
······

    resourceNames := getResourceNames(thresholds)

    // 这个函数是整个策略的重点
    lowNodes, sourceNodes := classifyNodes(
        getNodeUsage(ctx, client, nodes, thresholds, targetThresholds, resourceNames),
        // 该函数即lowThresholdFilter
        func(node *v1.Node, usage NodeUsage) bool {
            if nodeutil.IsNodeUnschedulable(node) {
                klog.V(2).InfoS("Node is unschedulable, thus not considered as underutilized", "node", klog.KObj(node))
                return false
            }
            return isNodeWithLowUtilization(usage)
        },
        // 该函数即highThresholdFilter
        func(node *v1.Node, usage NodeUsage) bool {
            return isNodeAboveTargetUtilization(usage)
        },
    )

    // log message in one line
    keysAndValues := []interface{}{
        "CPU", thresholds[v1.ResourceCPU],
        "Mem", thresholds[v1.ResourceMemory],
        "Pods", thresholds[v1.ResourcePods],
    }
    for name := range thresholds {
        if !isBasicResource(name) {
            keysAndValues = append(keysAndValues, string(name), int64(thresholds[name]))
        }
    }

    // log message in one line
    keysAndValues = []interface{}{
        "CPU", targetThresholds[v1.ResourceCPU],
        "Mem", targetThresholds[v1.ResourceMemory],
        "Pods", targetThresholds[v1.ResourcePods],
    }
    for name := range targetThresholds {
        if !isBasicResource(name) {
            keysAndValues = append(keysAndValues, string(name), int64(targetThresholds[name]))
        }
    }

    if len(lowNodes) == 0 {
        klog.V(1).InfoS("No node is underutilized, nothing to do here, you might tune your thresholds further")
        return
    }

    if len(lowNodes) <= strategy.Params.NodeResourceUtilizationThresholds.NumberOfNodes {
        klog.V(1).InfoS("Number of nodes underutilized is less or equal than NumberOfNodes, nothing to do here", "underutilizedNodes", len(lowNodes), "numberOfNodes", strategy.Params.NodeResourceUtilizationThresholds.NumberOfNodes)
        return
    }

    if len(lowNodes) == len(nodes) {
        klog.V(1).InfoS("All nodes are underutilized, nothing to do here")
        return
    }

    if len(sourceNodes) == 0 {
        klog.V(1).InfoS("All nodes are under target utilization, nothing to do here")
        return
    }

    evictable := podEvictor.Evictable(evictions.WithPriorityThreshold(thresholdPriority), evictions.WithNodeFit(nodeFit))

    // stop if node utilization drops below target threshold or any of required capacity (cpu, memory, pods) is moved
    continueEvictionCond := func(nodeUsage NodeUsage, totalAvailableUsage map[v1.ResourceName]*resource.Quantity) bool {
        if !isNodeAboveTargetUtilization(nodeUsage) {
            return false
        }
        for name := range totalAvailableUsage {
            if totalAvailableUsage[name].CmpInt64(0) < 1 {
                return false
            }
        }

        return true
    }

    evictPodsFromSourceNodes(
        ctx,
        sourceNodes,
        lowNodes,
        podEvictor,
        evictable.IsEvictable,
        resourceNames,
        "LowNodeUtilization",
        continueEvictionCond)

}
```

`classifyNodes()` 函数是重点函数，该函数区分了高低利用率的节点，返回的是节点组列表，入参传入了2个函数指针，里面区分高低利用率节点就是利用传入的函数指针，返回了2个`NodeUsage`结构体数组
