> 遇到过多次Load Average高，导致容器里应用很慢，但是CPU利用率很低的故障，重启kubelet和Docker都没有用，最后重启节点才解决，下来我们来研究下为什么有这种情况

# 什么是Load Average

使用top命令，可以看到 Load Avg

```bash
Processes: 519 total, 2 running, 517 sleeping, 2797 threads                                                                                                                                      16:59:12
Load Avg: 2.45, 2.28, 2.42  CPU usage: 6.81% user, 6.81% sys, 86.37% idle  SharedLibs: 248M resident, 43M data, 22M linkedit. MemRegions: 288853 total, 4402M resident, 161M private, 2673M shared.
PhysMem: 15G used (3406M wired), 550M unused
```

Load Avg后面为什么有三个数字，看说明就会指定，三个数值分别代表过去 1 分钟，5 分钟，15 分钟在这个节点上的 Load Average

Load Average其实是CPU的另外一种度量，举个例子，对于一个单个 CPU 的系统，如果在 1 分钟的时间里，处理器上始终有一个进程 在运行，同时操作系统的进程可运行队列中始终都有 9 个进程在等待获取 CPU 资源。那么 对于这 1 分钟的时间来说，系统的"load average"就是 1+9=10，这个定义对绝大部分的 Unix 系统都适用。

所以，对于单核CPU系统来说，最大只有100%，但是 Load Average有几千