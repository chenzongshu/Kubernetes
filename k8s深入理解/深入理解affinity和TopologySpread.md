# 总结

Kubernetes在调度节点的时候，是已Pod为最小调度单元的，但是在用户视角，操作的往往是deployment。而且，有时候因为高可用或者其他需求，需要对Pod直接的调度和亲和性，做出一定设置，Kubernetes 给出了2个方法：`affinity`和`TopologySpread` ，其中affinity包括antiaffinity，pod和node

podAffinity 可以将无数个 Pod 调度到特定的某一个拓扑域，这是堆叠；podAntiAffinity 则可以控制一个拓扑域只存在一个 Pod，这是打散。但这两种情况都太极端了，在不少场景下都无法达到理想的效果，例如为了实现容灾和高可用，将业务 Pod 尽可能均匀得分布在不同可用区就很难实现

`PodTopologySpread` 特性的提出正是为了对 Pod 的调度分布提供更精细的控制，以提高服务可用性以及资源利用率，1.19进入stable版本

# Affinity

具体用法省略，可以参见官网文档，不复杂，需要注意的就是官网不推荐大规模使用podAffinity/podAntiAffinity，这会导致调度器性能问题



# PodTopologySpread

具体用法省略，可以参见官网文档。其中就是**maxSkew** 参数比较难理解

**maxSkew** 描述了 Pod 在不同拓扑域中不均匀分布的最大程度，第i个拓扑域skew值见下面公式

```
skew[i] = 拓扑域[i]中匹配的 Pod 个数 - min{拓扑域[*]中匹配的 Pod 个数}
```

