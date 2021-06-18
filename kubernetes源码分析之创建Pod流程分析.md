# kubernetes源码分析: 创建Pod流程分析

敲下`kubectl run nginx --image=nginx --replicas=3` 命令后，K8s 中发生了哪些事情？

# 调用栈

```
NewDefaultKubectlCommand
NewKubectlCommand
```







