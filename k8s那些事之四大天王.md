> 本文不想对官方文档内容做过多的搬运，主要介绍一些自己的理解的总结和容易忽略的地方

# Deployment

Deployment无状态应用, 作为Kubernetes应用的最早, 最广泛的资源类型, 为Kubernetes闻名天下立下了汗马功劳. 容器最早的推广就是无状态应用。

所谓无状态应用，就是每个实例都是等幂而且有同样的上下文，可以任意销毁创建。

但是Deployment并不直接操作Pod，中间还抽象了一层ReplicaSet，形成了 `应用` -> `版本` -> `副本` 的抽象层次

```
         ┌─────────────┐         
         │ Deployment  │         -> 在RS上面封装了生命周期
         └─────────────┘         
                │                
                ▼                
         ┌─────────────┐         
         │ ReplicaSet  │         -> 在Pod上面封装了个数等信息
         └─────────────┘         
                │                
    ┌───────────┼───────────┐    
    ▼           ▼           ▼    
┌───────┐   ┌───────┐   ┌───────┐
│  Pod  │   │  Pod  │   │  Pod  │
└───────┘   └───────┘   └───────┘
```

`ReplicaSet`负责通过“控制器模式”，保证系统中 Pod 的个数永远等于指定的个数

`ReplicaSet`其实是Deployment中”版本“这个概念的实现，在RollingUpdate的时候，会多生成一个RS，然后新的RS慢慢增加Pod，老的RS慢慢减少Pod来完成滚动升级的。

# Statefulset

应用模型千差万别，特别是分布式系统中，有状态的应用非常之多，它多个实例之间，往往有依赖关系，比如:主从关系、主备关系。还有就是数据存储类应用，它的多个实例，往往都会在本地磁盘上保存一份数据。而这些实例一 旦被杀掉，即便重建出来，实例与数据之间的对应关系也已经丢失，从而导致应用失败。

所以，这种实例之间有不对等关系，以及实例对外部数据有依赖关系的应用，就被称为“有状态 应用”

statefulset, 从Kubernetes 1.9版本GA开始, 极大扩展了容器应用的边界。

StatefulSet 抽象了2种情况：

- 拓扑状态：即不同实例之间有先后顺序启动，删除也必须按照顺序来，删除掉再创建的实例必须要有一样的网络标识，这样原先的访问者才能使用同样的方法，访问到 这个新 Pod
- 存储状态：多个实例分别绑定了不同的存储数据。对于这些应用实例来说，Pod A 第一次读取到的数据，和隔了十分钟之后再次读取到的数据，应该是同一 份，哪怕在此期间Pod A 被重新创建过。

那么statefulset怎么解决上面两种问题呢？

## 拓扑状态

比较简单，statefulset会给Pod名字编号，从0开始。而且**这些 Pod 的创建也是严格按照编号顺序进行的**。比如web-0创建不成功的时候，web-1不会创建，销毁的时候则是相反。

此外，Kubernetes 还为每一个 Pod 提供了一个固定并且唯一的访问入口，即:这个 Pod 对应的 DNS 记录。格式是 `${PodName}.${HeadlessServiceName}`，不过，虽然DNS不变，但是Pod IP还是改变的，所以应该通过DNS来访问其Pod

## 存储状态

我们来看看官网的例子

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # has to match .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 # by default is 1
  template:
    metadata:
      labels:
        app: nginx # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi
```

StatefulSet 额外添加了一个 volumeClaimTemplates 字段。从名字就可以 看出来，它跟 Deployment 里 Pod 模板(PodTemplate)的作用类似。也就是说，凡是被这 个 StatefulSet 管理的 Pod，都会声明一个对应的 PVC;而这个 PVC 的定义，就来自于 volumeClaimTemplates 这个模板字段。更重要的是，这个 PVC 的名字，会被分配一个与这个 Pod 完全一致的编号。格式为`${PVC 名字}-${StatefulSet 名字}-${编号}`

# Daemonset

Daemonset守护进程，就是每个Node给启动有且只有一个Pod，并且新节点加入集群之后也会自动创建该Pod。

它可以在没有网络插件的情况下部署，意味着它本身就具备了和静态Pod差不多的特性。比如Flanneld，就可以通过Daemonset部署

那么Kubernetes怎么保证每个节点上只有一个Pod呢？大家当然会脱口而出Controller，宾果。

Controller首先从 Etcd 里获取所有的 Node 列表，然后遍历所有的Node。这时，它就可以很容易地去检查每个节点是否只有一个name=xxxx的Pod在运行，多的就干掉，少的就给新建。

但是Controller给Daemonset Pod多干了2件事大家一定要注意：

1、增加了一个nodeAffinity，通过节点亲和性，指定了这个Pod必须运行在某个节点上

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchFields:
          - key: metadata.name
            operator: In
            values:
            - node3
```

2、还有就是给Pod打了一堆容忍，以便初始化的时候跑到各个节点上

```yaml
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
  - effect: NoSchedule
    key: node.kubernetes.io/disk-pressure
    operator: Exists
  - effect: NoSchedule
    key: node.kubernetes.io/memory-pressure
    operator: Exists
  - effect: NoSchedule
    key: node.kubernetes.io/pid-pressure
    operator: Exists
  - effect: NoSchedule
    key: node.kubernetes.io/unschedulable
    operator: Exists
```

你也可以手动给他增加容忍，官方给的例子就加了一个容忍让该Pod跑到master上

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
······
spec:
······
  template:    
    spec:
      tolerations:
      # this toleration is to have the daemonset runnable on master nodes
      # remove it if your masters can't run pods
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
```

# Job/Cronjob

Job/Cronjob都是Kubernetes里面针对离线业务的资源对象，简单来说，就是只需要运行一次或多次的任务。

## Job

Job其实就是一种Pod（里面是Pod模板），但是和普通的Pod有所不同：

- restartPolicy只能被设置为 Never 和 OnFailure，这项是针对Pod的，注意
- Job Controller 重新创建 Pod 的间隔是呈指数增加的，即下一次重新创建 Pod 的动作会分别发生在 10 s、20 s、40 s ...后
- 里面有多个参数可以控制Job中Pod的并行行为：
  - `completions`：完成多少Pod数之后，代表Job完成；
  - `parallelism`： 最多可以同时运行的Pod数；
  - `activeDeadlineSeconds` ：Pod运行的最长时间，即如果Pod一直不Completed，过了该时间会被终止，可以在Pod状态里面看到终止原因是DeadlineExceeded
  - `ttlSecondsAfterFinished`：自动清理Complete或 Failed状态的Job的时间阈值

## Cronjob

cronjob实际就是一个job的控制器，看官方示例：

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            imagePullPolicy: IfNotPresent
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

这就是一个jobTemplate加上一个标准的Unix Cron格式的表达式嘛，每隔一段时间，就会创建一个Job

需要注意的点就是**cronjob的时间基于kube-controller-manager所在的时间**

附一个Cron时间表语法：

```
# ┌───────────── 分钟 (0 - 59)
# │ ┌───────────── 小时 (0 - 23)
# │ │ ┌───────────── 月的某天 (1 - 31)
# │ │ │ ┌───────────── 月份 (1 - 12)
# │ │ │ │ ┌───────────── 周的某天 (0 - 6) （周日到周一；在某些系统上，7 也是星期日）
# │ │ │ │ │                                   
# │ │ │ │ │
# │ │ │ │ │
# * * * * *
```

> 要生成 CronJob 时间表表达式，你还可以使用 [crontab.guru](https://crontab.guru/) 之类的 Web 工具

当然由于定时任务的特殊性，很可能某个 Job 还没有执行完，另外一个新 Job 就产生了。这时候，你可以通过 `spec.concurrencyPolicy` 字段来定义具体的处理策略

- Allow：这也是默认情况，这意味着这些 Job 可以同时存在;
- Forbid：这意味着不会创建新的 Pod，该创建周期被跳过;
- Replace：这意味着新产生的 Job 会替换旧的、没有执行完的 Job。

从1.20版本，Kubernetes将会有cronjob v2新的资源类型进入alpha特性

