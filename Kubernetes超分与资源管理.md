# 超分

为什么Kubernetes需要关注超分? 这个不是VM该关心的的事情吗?

如果我们的Kubernetes集群跑到VM上, 是不需要特意关心这个数据的, 但是如果当我们Kubernetes直接跑到物理机上的时候, 48C512G配置的服务器, 不可能让我只跑48个1C1G的容器, 这个时候对CPU资源进行超分也就变得必须了

一般来说, 对于云主机类产品, CPU超分1:5,内存不超分 的都算得上业界良心了(可以想想为什么CPU超分内存不超分, 下面会介绍)

对于Openstack和CloudStack这种IaaS平台类产品, 配置CPU和内存超分非常简单, OS可以在nova.conf里面配置, 有2个参数, 一个是CPU的超分比, 一个是内存的超分比, CS也可以方便的通过2个参数进行配置.

那对于PaaS类的产品Kubernetes呢? 我们先来看看Kubernetes怎么管理资源的

# Kubernetes资源管理

## 查看节点可分配资源

可以使用`kubectl describe nodes [节点名]`来查看

```
[root@localhost ~]# kubectl get nodes
NAME                    STATUS   ROLES    AGE    VERSION
localhost.localdomain   Ready    master   111d   v1.14.2
```

然后

```
[root@localhost ~]# kubectl describe nodes localhost.localdomain
Name:               localhost.localdomain
Roles:              master
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=localhost.localdomain
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/master=
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
                    node.alpha.kubernetes.io/ttl: 0
                    projectcalico.org/IPv4Address: 172.16.104.171/24
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Thu, 06 Jun 2019 18:18:13 +0800
Taints:             <none>
Unschedulable:      false
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Wed, 25 Sep 2019 19:42:38 +0800   Wed, 19 Jun 2019 05:02:29 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Wed, 25 Sep 2019 19:42:38 +0800   Wed, 19 Jun 2019 05:02:29 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Wed, 25 Sep 2019 19:42:38 +0800   Wed, 19 Jun 2019 05:02:29 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Wed, 25 Sep 2019 19:42:38 +0800   Wed, 19 Jun 2019 05:02:29 +0800   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  172.16.104.171
  Hostname:    localhost.localdomain
Capacity:
 cpu:                2
 ephemeral-storage:  36805060Ki
 hugepages-1Gi:      0
 hugepages-2Mi:      0
 memory:             3863576Ki
 pods:               110
Allocatable:
 cpu:                2
 ephemeral-storage:  33919543240
 hugepages-1Gi:      0
 hugepages-2Mi:      0
 memory:             3761176Ki
 pods:               110
System Info:
 Machine ID:                 785220a876814f0f8de7841b3bc81277
 System UUID:                ACCC4D56-DC90-05D0-8FB6-28A16A321D5A
 Boot ID:                    a6ed1354-4fcd-4ba2-8d71-6b5ff50088ba
 Kernel Version:             3.10.0-862.el7.x86_64
 OS Image:                   CentOS Linux 7 (Core)
 Operating System:           linux
 Architecture:               amd64
 Container Runtime Version:  docker://18.9.6
 Kubelet Version:            v1.14.2
 Kube-Proxy Version:         v1.14.2
PodCIDR:                     192.168.0.0/24
Non-terminated Pods:         (10 in total)
  Namespace                  Name                                             CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
  ---------                  ----                                             ------------  ----------  ---------------  -------------  ---
  default                    nginx-deployment-6b87cd7bcd-6db5p                500m (25%)    500m (25%)  200Mi (5%)       200Mi (5%)     172m
  default                    nginx-deployment-6b87cd7bcd-qxpql                500m (25%)    500m (25%)  200Mi (5%)       200Mi (5%)     172m

Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests     Limits
  --------           --------     ------
  cpu                2 (100%)     1 (50%)
  memory             540Mi (14%)  740Mi (20%)
  ephemeral-storage  0 (0%)       0 (0%)
```

## 资源限制参数

Kubernetes怎么对资源进行限制, 是不是立刻想到2个参数, Requests和limits, 没错, Kubernetes就是围绕这两个参数来进行资源的调度和QoS(资源服务质量管理)的.

大多数时候, 我们在定义Pod的时候可以不定义这两个参数, 这样Kubernetes会认为该Pod需要的资源很少, 就可以调度到任意节点上.

- `Requests`: Pod需要的最小的资源, 注意不代表实际占用的最小的资源, 可以理解为最小预分配资源
- `Limits`: Pod所能占的最大的资源

注意:
1、**在繁忙时期，当Pod试图请求超过`Limits`限制的资源时，此进程会被Kubernetes杀掉**;
2、再次强调`Requests`不代表实际资源, 比如PodA设置的`Requests`的MEM为500M, 但是实际可能启动时候只占100M内存, `Requests`只是预分配最小资源, 比如宿主机可以给容器用的内存有8G, 建`Requests`为1G内存的容器, 只能创建8个, 即 **Requests <= 节点可用资源**
3、`Requests` <= `Limits`

如果缺少资源导致Pod不能进入Running态, 可以通过`kubectl describe`命名查看Pod, 可以看到事件中有如下报错信息

```
Events:
  Type     Reason            Age               From               Message
  ----     ------            ----              ----               -------
  Warning  FailedScheduling  1s (x2 over 68s)  default-scheduler  0/1 nodes are available: 1 Insufficient cpu.
```

## 资源调度策略

- Kubernetes的`kubelet`通过计算Pod中所有容器的`Requests`的总和来决定对Pod的调度。

- 不管是CPU还是内存， Kubernetes调度器和 kubelet都会确保节点上所有Pod的`Requests`的总和不会超过在该节点上可分配给容器使用的资源容量上限。

## 资源服务质量管理QoS

当资源紧张时, Pod负载的波动可能导致部分容器被杀掉, 这个时候我们肯定希望不太重要的容器被杀掉, 那么如何衡量重要程度呢? 

Kubernetes引入三个QoS等级

- `Guaranteed`: 如果Pod中所有Container的所有Resource的limit和request都相等且不为0
- `Burstable`: 不符合`Guaranteed`和`Best-Effort`的Pod
- `Best-Effort`: 如果Pod中所有容器的所有Resource的request和limit都没有赋值

这三个等级的重要程度依次降低, 即`Best-Effort`优先级最低, 资源不足时候最先杀掉, 而尽量保证`Guaranteed`, 

这个值可以通过`kubectl describe pod [Pod名]`来查看, 里面对应的参数如下: 

```
......
QoS Class:       Guaranteed
Node-Selectors:  <none>
```

注意:

- CPU是可压缩资源, 因为CPU是按时间片来轮流使用的, 而内存是不可压缩资源, 所以一般内存不超分, CPU如果使用超过`Limits`, cgroup会对其限流, 如果内存超了会被杀掉.
- 如果定义了`Limits`而没有定义`Requests`, **Requests值在未定义时会被默认为Limits想等**, 这时候QoS为`Guaranteed`
- 空闲CPU资源按照Requests值比例分配, 比如PodA配置 Requests 1 Limits10, PodB配置Requests 2 Limits8, 如果A和B同时需要更多CPU资源,而这个时候空闲只有1.5CPU, 那么这1.5CPU将按照1:2的比例分配给A和B, 即A最终再得到0.5CPU, B再得到1CPU

## LimitRange 和 Resource Quotas

这两个概念简单介绍下 

资源配置范围管理`LimitRange`是由于如果我们需要对集群内每一个Pod都配置`Requests`和`Limits`, 非常繁琐, 所以对其来一个全局限制, 可以创建一个`LimitRange`资源然后把其应用的某一个namespace里面
`kubectl create -f limitsrange.yaml --namespace=limittest`

资源配额管理`Resource Quotas`是为了解决多个团队或者用户共享集群的时候, 公平分配资源的问题. 需要在`APIServer`中的`--admission-control`中添加`ResourceQuota`, 然后用法和`LimitRange`类似, 绑定在`namespace`中

# 超分设置

了解了Kubernetes对资源的一些知识, 再来看超分设置, 不难得出Kubernetes就是通过 `Requests`和`Limits`参数来设置超分的, 具体怎么设置呢?

那一个48C512G的服务器为work node来举例, 预留小部分系统资源后可分配容器的资源预估为 46C460G, 假设我们系统最小支持的容器规格为1C1G, 按照内存不超分设计, 我们可以最大跑460个容器, 然后根据上面的知识推算CPU的`Requests`值, 46C/460 = 0.1 , 即CPU的 `Requests`值设为100ms, 而CPU的`Limits`设为用户购买的CPU规格, 如果是1C1G的, `Limits`设为1

# 资源预留

上面只是Kubernetes对Pod的资源管理, 算是集群内部的资源管理, 但是生产里面的work节点里面, 可能还存在kubelete等其他Kubernetes和操作系统的组件, 这个时候就会有资源抢占的问题, 如果Pod优先级设置不当, 在资源紧缺的时候可能还会导致节点挂掉, 所以Kubernetes提供了资源预留这么一个功能, 来为系统守护进程和k8s组件预留计算资源, 目前支持对CPU, memory, ephemeral-storage三种资源进行预留

有多个部分

- Node Capacity：Node的硬件资源总量
- kube-reserved：给k8s系统进程预留的资源(包括kubelet、container runtime、node problem detector等，但不会给以pod形式起的k8s系统进程预留资源)
- system-reserved：给linux系统守护进程预留的资源
- eviction-threshold：通过--eviction-hard参数为节点预留内存，当节点可用内存值低于此值时，kubelet会进行pod的驱逐
- allocatable：是真正可供节点上Pod使用的容量，kube-scheduler调度Pod时的参考此值(kubectl describe node可以看到，Node上所有Pods的request量不超过Allocatable)

对应关系为 

```
allocatable = NodeCapacity - [kube-reserved] - [system-reserved] - [eviction-threshold]
```

对应到`kubelet`的启动参数, 有如下:

kubelet的启动参数中涉及资源预留的主要有如下几个：(其中前2个最常用,即为Kubernetes组件和操作系统保留资源)

- `--kube-reserved=[cpu=100m][,][memory=100Mi][,][ephemeral-storage=1Gi]`
- `--system-reserved=[cpu=100mi][,][memory=100Mi][,][ephemeral-storage=1Gi]`
- `--cgroups-per-qos`

> 可选，默认开启。开启这个参数后，kubelet会将所有的pod创建在kubelet管理的cgroup层次结构下（这样才有了限制所有Pod使用资源总量的基础）。要想启用Node Allocatable特性，这个参数必须开启

- `--cgroup-driver`

> 可选。指定kubelet使用的cgroup driver。默认为cgroupfs，还可以是systemd，但是这个值需要和docker runtime所使用的cgroup driver保持一致。

- `--cgroup-root`

> 可选。指定给pod使用的根cgroup，容器运行时会尽量将pod的资源限制在这个根cgroup下面。默认为空，即使用容器运行时作为根cgroup

- `--enforce-node-allocatable=pods[,][system-reserved][,][kube-reserved]`

> 指定kubelet为哪些进程使用cgroup来做硬限制，可选的值有：pods,kube-reserved,system-reserve

- `--kube-reserved-cgroup`
- `--system-reserved-cgroup`

> 上面2个参数是指定Kubernetes和系统组件使用的cgroup, 这个必须先创建好

- `--eviction-hard`

> 设置进行pod驱逐的阈值，这个参数只支持内存和磁盘

注意: 如果对k8s组件和系统进程也做了cgroup硬限制，当k8s组件和系统组件资源使用量超过这个限制时，会出现这些进程被杀掉的情况


