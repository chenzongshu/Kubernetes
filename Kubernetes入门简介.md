# 简介

Kubernetes作为时下最火的容器编排工具，不用多说。架构图如下：

![](https://images2015.cnblogs.com/blog/704717/201703/704717-20170304103633345-155330022.png)

# 节点

k8s集群中，存在两种性质的物理节点，一种是master，一种是node节点

- master：简单的说，master节点就是管理节点，负责整个集群的资源调度；
- node： 工作节点，执行启动容器的节点；


## Master

　　Master节点上面主要由四个模块组成：APIServer、scheduler、controller manager、etcd。

-  **APIServer**：APIServer负责对外提供RESTful的Kubernetes API服务，它是系统管理指令的统一入口，任何对资源进行增删改查的操作都要交给APIServer处理后再提交给etcd。如架构图中所示，kubectl（Kubernetes提供的客户端工具，该工具内部就是对Kubernetes API的调用）是直接和APIServer交互的。

- **scheduler**：scheduler的职责很明确，就是负责调度pod到合适的Node上。如果把scheduler看成一个黑匣子，那么它的输入是pod和由多个Node组成的列表，输出是Pod和一个Node的绑定，即将这个pod部署到这个Node上。Kubernetes目前提供了调度算法，但是同样也保留了接口，用户可以根据自己的需求定义自己的调度算法。

- **controller manager**：如果说APIServer做的是“前台”的工作的话，那controller manager就是负责“后台”的。每个资源一般都对应有一个控制器，而controller manager就是负责管理这些控制器的。比如我们通过APIServer创建一个pod，当这个pod创建成功后，APIServer的任务就算完成了。而后面保证Pod的状态始终和我们预期的一样的重任就由controller manager去保证了。

- **etcd**：etcd是一个高可用的键值存储系统，Kubernetes使用它来存储各个资源的状态，从而实现了Restful的API。

## Node

每个Node节点主要由三个模块组成：kubelet、kube-proxy、runtime。

- **runtime**：runtime指的是容器运行环境，目前Kubernetes支持docker和rkt两种容器。

- **kube-proxy**：该模块实现了Kubernetes中的服务发现和反向代理功能。反向代理方面：kube-proxy支持TCP和UDP连接转发，默认基于Round Robin算法将客户端流量转发到与service对应的一组后端pod。服务发现方面，kube-proxy使用etcd的watch机制，监控集群中service和endpoint对象数据的动态变化，并且维护一个service到endpoint的映射关系，从而保证了后端pod的IP变化不会对访问者造成影响。另外kube-proxy还支持session affinity。

- **kubelet**：Kubelet是Master在每个Node节点上面的agent，是Node节点上面最重要的模块，它负责维护和管理该Node上面的所有容器，但是如果容器不是通过Kubernetes创建的，它并不会管理。本质上，它负责使Pod得运行状态与期望的状态一致。


# Pod

Pod 是Kubernetes的基本操作单元，也是应用运行的载体。整个Kubernetes系统都是围绕着Pod展开的，比如如何部署运行Pod、如何保证Pod的数量、如何访问Pod等。另外，Pod是一个或多个机关容器的集合，这可以说是一大创新点，提供了一种容器的组合的模型。

## Pod与容器

　　在Docker中，容器是最小的处理单元，增删改查的对象是容器，容器是一种虚拟化技术，容器之间是隔离的，隔离是基于Linux Namespace实现的。而在Kubernetes中，Pod包含一个或者多个相关的容器，Pod可以认为是容器的一种延伸扩展，一个Pod也是一个隔离体，而Pod内部包含的一组容器又是共享的（包括PID、Network、IPC、UTS）。除此之外，Pod中的容器可以访问共同的数据卷来实现文件系统的共享。
	
## 镜像
	
在kubernetes中，镜像的下载策略为：

- Always：每次都下载最新的镜像
- Never：只使用本地镜像，从不下载
- IfNotPresent：只有当本地没有的时候才下载镜像

Pod被分配到Node之后会根据镜像下载策略进行镜像下载，可以根据自身集群的特点来决定采用何种下载策略。无论何种策略，都要确保Node上有正确的镜像可用。

## Pod的一些设置

可以在yaml文件中设置

```
启动命令，如：spec-->containers-->command；

环境变量，如：spec-->containers-->env-->name/value；

端口桥接，如：spec-->containers-->ports-->containerPort/protocol/hostIP/hostPort（使用hostPort时需要注意端口冲突的问题，不过Kubernetes在调度Pod的时候会检查宿主机端口是否冲突，比如当两个Pod均要求绑定宿主机的80端口，Kubernetes将会将这两个Pod分别调度到不同的机器上）;

Host网络，一些特殊场景下，容器必须要以host方式进行网络设置（如接收物理机网络才能够接收到的组播流），在Pod中也支持host网络的设置，如：spec-->hostNetwork=true；

数据持久化，如：spec-->containers-->volumeMounts-->mountPath;

重启策略，当Pod中的容器终止退出后，重启容器的策略。这里的所谓Pod的重启，实际上的做法是容器的重建，之前容器中的数据将会丢失，如果需要持久化数据，那么需要使用数据卷进行持久化设置。Pod支持三种重启策略：Always（默认策略，当容器终止退出后，总是重启容器）、OnFailure（当容器终止且异常退出时，重启）、Never（从不重启）；
```

## Pod生命周期

Pod被分配到一个Node上之后，就不会离开这个Node，直到被删除。当某个Pod失败，首先会被Kubernetes清理掉，之后ReplicationController将会在其它机器上（或本机）重建Pod，重建之后Pod的ID发生了变化，那将会是一个新的Pod。所以，Kubernetes中Pod的迁移，实际指的是在新Node上重建Pod。以下给出Pod的生命周期图。

![](https://images2015.cnblogs.com/blog/704717/201703/704717-20170304103856954-1091445106.png)


**生命周期回调函数**：PostStart（容器创建成功后调研该回调函数）、PreStop（在容器被终止前调用该回调函数）。以下示例中，定义了一个Pod，包含一个JAVA的web应用容器，其中设置了PostStart和PreStop回调函数。即在容器创建成功后，复制/sample.war到/app文件夹中。而在容器终止之前，发送HTTP请求到`http://monitor.com:8080/waring`，即向监控系统发送警告。具体示例如下：

```
………..
containers:
- image: sample:v2  
     name: war
     lifecycle：
      posrStart:
       exec:
         command:
          - “cp”
          - “/sample.war”
          - “/app”
      prestop:
       httpGet:
        host: monitor.com
        psth: /waring
        port: 8080
        scheme: HTTP
```

# Replication Controller

Replication Controller（RC）是Kubernetes中的另一个核心概念，应用托管在Kubernetes之后，Kubernetes需要保证应用能够持续运行，这是RC的工作内容，它会确保任何时间Kubernetes中都有指定数量的Pod在运行。在此基础上，RC还提供了一些更高级的特性，比如滚动升级、升级回滚等。

## Label

RC与Pod的关联是通过Label来实现的。Label机制是Kubernetes中的一个重要设计，通过Label进行对象的弱关联，可以灵活地进行分类和选择。对于Pod，需要设置其自身的Label来进行标识，Label是一系列的Key/value对，在Pod-->metadata-->labeks中进行设置。

　　Label的定义是任一的，但是Label必须具有可标识性，比如设置Pod的应用名称和版本号等。另外Lable是不具有唯一性的，为了更准确的标识一个Pod，应该为Pod设置多个维度的label。如下：
	
```
　　　　"release" : "stable", "release" : "canary"

　　　　"environment" : "dev", "environment" : "qa", "environment" : "production"

　　　　"tier" : "frontend", "tier" : "backend", "tier" : "cache"

　　　　"partition" : "customerA", "partition" : "customerB"

　　　　"track" : "daily", "track" : "weekly"
```

举例，当你在RC的yaml文件中定义了该RC的selector中的label为app:my-web，那么这个RC就会去关注Pod-->metadata-->labeks中label为app:my-web的Pod。修改了对应Pod的Label，就会使Pod脱离RC的控制。同样，在RC运行正常的时候，若试图继续创建同样Label的Pod，是创建不出来的。因为RC认为副本数已经正常了，再多起的话会被RC删掉的。
	
## 弹性伸缩

弹性伸缩是指适应负载变化，以弹性可伸缩的方式提供资源。反映到Kubernetes中，指的是可根据负载的高低动态调整Pod的副本数量。调整Pod的副本数是通过修改RC中Pod的副本是来实现的，示例命令如下：

扩容Pod的副本数目到10

```
$ kubectl scale relicationcontroller yourRcName --replicas=10
```

缩容Pod的副本数目到1

```
$ kubectl scale relicationcontroller yourRcName --replicas=1
```

## 滚动升级

滚动升级是一种平滑过渡的升级方式，通过逐步替换的策略，保证整体系统的稳定，在初始升级的时候就可以及时发现、调整问题，以保证问题影响度不会扩大。Kubernetes中滚动升级的命令如下：

```
$ kubectl rolling-update my-rcName-v1 -f my-rcName-v2-rc.yaml --update-period=10s
```

升级开始后，首先依据提供的定义文件创建V2版本的RC，然后每隔10s（--update-period=10s）逐步的增加V2版本的Pod副本数，逐步减少V1版本Pod的副本数。升级完成之后，删除V1版本的RC，保留V2版本的RC，及实现滚动升级。

　　升级过程中，发生了错误中途退出时，可以选择继续升级。Kubernetes能够智能的判断升级中断之前的状态，然后紧接着继续执行升级。当然，也可以进行回退，命令如下：
	
```
$ kubectl rolling-update my-rcName-v1 -f my-rcName-v2-rc.yaml --update-period=10s --rollback	
```

回退的方式实际就是升级的逆操作，逐步增加V1.0版本Pod的副本数，逐步减少V2版本Pod的副本数。

# Job

　　从程序的运行形态上来区分，我们可以将Pod分为两类：长时运行服务（jboss、mysql等）和一次性任务（数据计算、测试）。RC创建的Pod都是长时运行的服务，而Job创建的Pod都是一次性任务。

　　在Job的定义中，restartPolicy（重启策略）只能是Never和OnFailure。Job可以控制一次性任务的Pod的完成次数（Job-->spec-->completions）和并发执行数（Job-->spec-->parallelism），当Pod成功执行指定次数后，即认为Job执行完毕。
	

# Service

如果Pods是短暂的，那么重启时IP地址可能会改变，怎么才能从前端容器正确可靠地指向后台容器呢？

Service是定义一系列Pod以及访问这些Pod的策略的一层抽象。Service通过Label找到Pod组.

在Kubernetes中，在受到RC调控的时候，Pod副本是变化的，对于的虚拟IP也是变化的，比如发生迁移或者伸缩的时候。这对于Pod的访问者来说是不可接受的。Kubernetes中的Service是一种抽象概念，它定义了一个Pod逻辑集合以及访问它们的策略，Service同Pod的关联同样是居于Label来完成的。Service的目标是提供一种桥梁， 它会为访问者提供一个固定访问地址，用于在访问时重定向到相应的后端，这使得非 Kubernetes原生应用程序，在无须为Kubemces编写特定代码的前提下，轻松访问后端。

Service同RC一样，都是通过Label来关联Pod的。当你在Service的yaml文件中定义了该Service的selector中的label为app:my-web，那么这个Service会将Pod-->metadata-->labeks中label为app:my-web的Pod作为分发请求的后端。当Pod发生变化时（增加、减少、重建等），Service会及时更新。这样一来，Service就可以作为Pod的访问入口，起到代理服务器的作用，而对于访问者来说，通过Service进行访问，无需直接感知Pod。

需要注意的是，Kubernetes分配给Service的固定IP是一个虚拟IP，并不是一个真实的IP，在外部是无法寻址的。真实的系统实现上，Kubernetes是通过Kube-proxy组件来实现的虚拟IP路由及转发。所以在之前集群部署的环节上，我们在每个Node上均部署了Proxy这个组件，从而实现了Kubernetes层级的虚拟转发网络。

## 外部服务service

应用需要用到外部服务, 比如外部数据库, 外部短信系统等的时候, 可以使用外部服务service

使用方式也比较简单, 创建一个**不带选择器**的service, 再创建一个**同名**的endpoint, 即可

```
apiVersion: v1
kind: Service
metadata:
  name: mysql-test
spec:
  ports:
  - port: 80
    targetPort: 81
    protocol: TCP
```

将外部服务器的172.17.241.47、59.107.26.221的80端口映射到内部服务

```
apiVersion: v1
kind: Endpoints
metadata:
  name: mysql-test
subsets:
  - addresses:
    - ip: 172.17.241.47
    - ip: 59.107.26.221
    ports:
    - port: 80
      protocol: TCP
```

如果外部服务是个URI, 还可以用ExternalName, 这里不详细讲解

```
kind: Service
apiVersion: v1
metadata:
  name: mongo
  spec:
  type: ExternalName
  externalName: ds149763.mlab.com
```

## headless service

- 希望自己控制负载均衡策略
- 应用程序系统知道属于同组服务的其他实例, 然后自己决定如何处理这个Pod列表

使用方式为不为service设置ClusterIP

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx-demo
  ports:
  - port: 80
    name: nginx
  clusterIP: None
```

# liveness和readness

两种健康检测方式到底有什么区别? 简明扼要的归纳

- liveness检测容器的存活性, readness检测容器里面的应用是否处于就绪状态;
- liveness检测失败会根据重启策略来决定是否重启Pod, readness检测失败会把pod 改为 not ready, 这个时候不会导流量到该pod;
- 两者功能不同, 如果只配置了liveness, 容器启动成功但是应用还没成功的时候, 会导流量到该pod, 会产生一些错误;

# RBAC

## role

- role只能对命名空间内的资源进行授权

