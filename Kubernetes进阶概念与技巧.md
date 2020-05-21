# Kubernetes日志



```
--v=0	通常，这对操作者来说总是可见的。
--v=1	当您不想要很详细的输出时，这个是一个合理的默认日志级别。
--v=2	有关服务和重要日志消息的有用稳定状态信息，这些信息可能与系统中的重大更改相关。这是大多数系统推荐的默认日志级别。
--v=3	关于更改的扩展信息。
--v=4	调试级别信息。
--v=6	显示请求资源。
--v=7	显示 HTTP 请求头。
--v=8	显示 HTTP 请求内容。
--v=9	显示 HTTP 请求内容，并且不截断内容。
```

# 查看Pod信息

如果一些Pod信息闪创闪退, 每次敲命令看不到, 可以加上 "-w"参数, 一直输出

```
kubectl get po -o wide -w
```


# Kubernetes service

- service的域名, **只能在同一个namespace中解析**, 如果跨域名必须使用 `域名.Namespace名`的形式来访问

## Pod访问apiserver

pod中进程访问apiserver, 是因为apiserver本身也是一个service, 他的名称就是`kubernetes`

## 负载分发策略

有两种: 

- `RoundRobin`：轮询模式，即轮询将请求转发到后端的各个pod上（默认模式）；
- `SessionAffinity`：基于客户端IP地址进行会话保持的模式，第一次客户端访问后端某个pod，之后的请求都转发到这个pod上。

# RBAC

RBAC三个核心概念: 

1. Role：角色，它其实是一组规则，定义了一组对 Kubernetes API 对象的操作权限。
2. Subject：被作用者，既可以是“人”，也可以是“机器”，也可以使你在 Kubernetes 里定义的“用户”。
3. RoleBinding：定义了“被作用者”和“角色”的绑定关系。

Role + RoleBinding + ServiceAccount 的权限分配方式是编写和安装各种插件的时候，经常用到的组合

## role

- role只能对命名空间内的资源进行授权

## RoleBinding

- 通过roleRef字段来引用对Role的引用, 但是注意`RoleBinding`也是有namespace限制的, 所以roleRef也只能引用本ns的role
- User只能通过外部认证系统来完成
- 如果要访问非namespace范围的对象, 只能通过 ClusterRole 和 ClusterRoleBinding 这两个组合了

# Service Account

Service Account 的授权信息和文件，实际上保存在它所绑定的一个特殊的 Secret 对象里的。这个特殊的 Secret 对象，就叫作**ServiceAccountToken**。任何运行在 Kubernetes 集群上的应用，都必须使用这个 ServiceAccountToken 里保存的授权信息，也就是 Token，才可以合法地访问 API Server

Kubernetes 已经为你提供了一个的默认“服务账户”（default Service Account）。并且，任何一个运行在 Kubernetes 里的 Pod，都可以直接使用这个默认的 Service Account，而无需显示地声明挂载它

如果你通过`kubectl describe po xxx`查看一下任意一个运行在 Kubernetes 集群里的 Pod，就会发现，每一个 Pod，都已经自动声明一个类型是 Secret、名为 default-token-xxxx 的 Volume，然后 自动挂载在每个容器的一个固定目录上。

```
Volumes:
default-token-s8rbq:
Type:       Secret (a volume populated by a Secret)
SecretName:  default-token-s8rbq
Optional:    false
```

一旦 Pod 创建完成，容器里的应用就可以直接从这个默认 ServiceAccountToken 的挂载目录里访问到授权信息和文件。这个容器内的路径在 Kubernetes 里是固定的，即：/var/run/secrets/kubernetes.io/serviceaccount

你的应用程序只要直接加载这些授权文件，就可以访问并操作 Kubernetes API 了。而且，如果你使用的是 Kubernetes 官方的 Client 包（`k8s.io/client-go`）的话，它还可以自动加载这个目录下的文件，你不需要做任何配置或者编码操作。



# StatuefulSet

- 按照标号来扩容或者缩容

- 对statuefulset的访问，必须使用 DNS 记录或者 hostname 的方式，而绝不应该直接访问这些 Pod 的 IP 地址, 因为IP地址会变

- StatuefulSet 对应Headless Service创建的DNS记录格式为

  ```
  <pod-name>.<svc-name>.<namespace>.svc.cluster.local
  ```



# nodeAffinity

怎么对Pod按照标签进行特定节点的调度, 大家一定脱口而出`nodeSelector`

`nodeSelector`是一种简单粗暴的方式, 使用方式也很简单, 分两步: 1. 节点打标签; 2. yaml文件指定需要调度的节点

**`nodeSelector`是一种快要被淘汰的参数**, 替代它的正是nodeAffinity(节点亲和性), 因为其语法更加丰富, 适应不同的场景. 而且还有`Anti-Affinity`(反亲和性), 可以进行简单的逻辑组合了

调度可以分成软策略和硬策略两种方式，软策略就是如果你没有满足调度要求的节点的话，POD 就会忽略这条规则，继续完成调度过程；而硬策略就比较强硬了，如果没有满足条件的节点的话，就不断重试直到满足条件为止

`nodeAffinity`就有两上面两种策略：`preferredDuringSchedulingIgnoredDuringExecution`和`requiredDuringSchedulingIgnoredDuringExecution`，前面的就是软策略，后面的就是硬策略。

示例:

```
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
  labels:
    app: node-affinity-pod
spec:
  containers:
  - name: with-node-affinity
    image: nginx
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: NotIn
            values:
            - 192.168.1.140
            - 192.168.1.161
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: source
            operator: In
            values:
            - qikqiak
```

- 上面这个 POD 首先是要求 POD 不能运行在140和161两个节点上，如果有个节点满足`source=qikqiak`的话就优先调度到这个节点上

- 操作符如下:

  > - In：label 的值在某个列表中
  > - NotIn：label 的值不在某个列表中
  > - Gt：label 的值大于某个值
  > - Lt：label 的值小于某个值
  > - Exists：某个 label 存在
  > - DoesNotExist：某个 label 不存在

如果`nodeSelectorTerms`下面有多个选项的话，满足任何一个条件就可以了；如果`matchExpressions`有多个选项的话，则必须同时满足这些条件才能正常调度 POD



# StorageClass

为什么需要StorageClass?

试想一下, 一个k8s集群有上万个PVC, 要运维手动去创建上万个PV么? 更麻烦的是，随着新的 PVC 不断被提交，运维人员就不得不继续添加新的、能满足条件的 PV，否则新的 Pod 就会因为 PVC 绑定不到 PV 而失败。在实际操作中，这几乎没办法靠人工做到。

我们就可以使用StorageClass, **而 StorageClass 对象的作用，其实就是创建 PV 的模板**

具体地说，StorageClass 对象会定义如下两个部分内容：

- 第一，PV 的属性。比如，存储类型、Volume 的大小等等。
- 第二，创建这种 PV 需要用到的存储插件。比如，Ceph 等等。这个通过`provisioner`字段来实现
- parameter字段就是PV的参数

# 本地持久化卷

Local Persistent Volume是不是等同于HostPath加上NodeAffinity吗？

**绝不应该把一个宿主机上的目录当作 PV 使用**。这是因为，这种本地目录的存储行为完全不可控，它所在的磁盘随时都可能被应用写满，甚至造成整个宿主机宕机。而且，不同的本地目录之间也缺乏哪怕最基础的 I/O 隔离机制。





那kubernetes怎么把Pod调度到对应的节点上呢



