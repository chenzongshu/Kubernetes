## 最大规模

- 节点数不超过 5000
- Pod 总数不超过 150000
- 容器总数不超过 300000
- 每个节点的 pod 数量不超过 100

# 资源需求

管理节点

| 节点规模       | 资源需求 | 备注 |
| -------------- | -------- | ---- |
| 1-5 个节点     | 1C4G     |      |
| 6-10 个节点    | 2C8G     |      |
| 11-100 个节点  | 4C16G    |      |
| 101-250 个节点 | 8C32G    |      |
| 251-500 个节点 | 16C64G   |      |
| 超过 500 节点  | 32C120G  |      |

# 稳定性自查项

## 组件挂掉/拥塞

任意组件挂掉/拥塞, 会影响已经运行的容器么?

> 举例, 如果APIServer异常,拥塞, 此时可能若干节点的 kubelet 均无法正常上报心跳，在 controller 处会将其置为 not ready。而后可能会导致由 rs 触发重启的重建，service 摘除相对应节点上容器的流量等等。而实际上，这些节点的容器其实原本是正常运行的。出现问题的，仅仅是 apiserver 的拥塞而已。又比如，采用的网络方案，如果网络元数据的存储故障了，是否会导致现有容器的网络受到影响，是否会导致网络瘫痪。

## 异常告警

任意组件异常, 都会有告警和对应的处理方式吗?

> 这里以 etcd 为例。毫无疑问，etcd 是 Kubernetes 的核心中枢。但是 etcd 并不是想象中那么万无一失。在实际生产实践中，遇到过多次莫名其妙的故障，导致整个 etcd 集群无法写入数据，甚至完全故障。最后不得不启动先前的故障预案。在之前的故障预案中，我们做了多种计划和多次演练，包括上策：从 etcd 集群原节点恢复，中策：将数据迁移到新的节点进行恢复，以及下策：从定时备份的数据中恢复。如果从定时备份的数据中恢复，则难免会有一部分数据丢失。在实际中，我们遇到过的最坏情况是将数据迁移到新节点进行集群重建恢复。但是三种情况的演练仍然是必要的，以确保在最坏情况下，仍然能最大限度的保障集群的稳定和安全。

## 组件损坏

任意组件损坏, 集群都能恢复么?

组件的故障异常既然不可避免，那么如何对组件进行监控告警，如何定义告警的规则和和方式，以及告警后如何接入自动化的处理，就需要进行认真的思考。比如，各个组件所在的容器 / 物理机需要配合相应的资源监控告警，组件本身也需要进行健康检查等方式的监控告警，甚而，还需要对组件本身的性能数据进行监控告警。



# 容器网络



## 统一SDN

整个数据中心构建统一SDN网络， 这样当有了容器平台的时候，接入就容易的多，而且下层的VXLAN对于容器内的应用不可见，对于容器来讲，他就像和Vmware虚拟机里面的应用以及KVM里面的应用在同一个二层网络里面一样，十分平滑。

如果我们有了容器平台，无论是物理机部署，还是虚拟机部署，都可以通过桥接的方式，连接到外面的网络中，不同的租户可以使用不同的VLAN，通过硬件SDN的VXLAN封装实现互通。

当然，使用统一的软件或者硬件SDN，需要对于整个运维部和机房有统一的规划，这在很多企业是没有做到的

## 机房网络BGP

规划使用Calico的BGP模式，需要底层支持

## 云平台内部自行搭建K8S集群下网络

很多企业担心被云厂商绑定，因而要跨云进行部署，并且不愿意使用云平台提供的托管K8S，而是通过类似多容器部署管理平台，自行在多个云上搭建K8S集群。  

这就面临一个问题，就是云内的VPC是不支持BGP网络的，因而在云内搭建K8S集群，就需要使用Calico的IPIP模式，也即在云的网络虚拟化之上，又多了一层虚拟化，这样使得性能有所损耗。  

另一个问题就是，Calico的IPIP模式的网络是Overlay网络，和其他虚拟机上的应用天然不通，必须要经过代理或者NAT才能通信，要求业务做一定的适配，影响比较大。  

这个时候，有两种替代方案。 

-  第一，在大多数的云平台上，一个虚拟机是可以创建多个网卡的，我们可以将一个网卡用于管控通信，其他的网卡在创建容器的时候，用命令行将其中一个虚拟机网卡塞到容器网络的namespace里面，这样容器就能够使用VPC网络和其他虚拟机上的应用进行平滑通信了，而且没有了二次虚拟化。

 当然云平台对于一个虚拟机上的网卡数目有限制，这就要求我们创建的虚拟机规格不能太大，因为上面能够跑的容器的数量受到网卡数目限制。 另外我们在调度容器的时候，需要考虑一个节点还有没有剩余的网卡。Kubernetes有一个概念叫Extended Resources，官方文档举了这样一个例子。

```
curl --header "Content-Type: application/json-patch+json" \
--request PATCH \
--data '[{"op": "add", "path": "/status/capacity/example.com~1dongle", "value": "4"}]' \
http://localhost:8001/api/v1/nodes/<your-node-name>/status
```

同理可以将网卡作为Extended Resources放到每个Node上，然后创建Pod的时候，指定网卡资源。

```
apiVersion: v1
kind: Pod
metadata:  
	name: extended-resource-demo
spec:  
	containers:  
	- name: extended-resource-demo-ctr    
	  image: nginx    
	  resources:      
	  	requests:      example.com/dongle: 3      
	  	limits:        example.com/dongle: 3
```

同时要配合开发CNI插件，通过nsenter，ip netns等命令将网卡放入namespace中。

 

- 第二，就是使用hostnetwork，也即容器共享虚拟机网络，这样使得每个容器没有单独的网络命名空间，因而端口会冲突，需要随机分配应用的端口，由于应用的端口事先不可知，需要通过服务发现机制，给注册中心注册随机的端口。

当然还可以使用Calico网络,然后使用Ingress进行容器内外的访问



# Etcd

ETCD作为Kubernetes唯一的持久化存储组件，不但承载了对象的存储功能，还有对象的变化通知功能，压力是非常大的。

因而这里对于ETCD的配置，一定要用SSD盘，这对整个集群的响应时延非常关键，另外ETCD对于存储量有限制，可以配置quota-backend-bytes。

目前存储在ETCD里面的数据比较多，除了主流的Pod，Service这些，event和ConfigMap也会占用很大的量，Event主要用来保存各个组件的运行状态，可以通过他发现异常，Event数据写入比较频繁，带来压力比较大，又不是特别关键的数据，可以分ETCD集群存储。按照官方文档，可以如下配置。

```
--etcd-servers-overrides=/events#https://0.example.com:2381;https://1.example.com:2381;https://2.example.com:2381
```

另外ConfigMap如果存储的配置项比较多，也会比较大，建议使用Apollo配置中心，而不要直接使用容器的配置中心。

## 备份和恢复

备份

```
etcdctl snapshot --endpoints=$ETCD_SERVERS --cacert=/var/lib/etcd/cert/ca.pem --cert=/var/lib/etcd/cert/etcd-client.pem --key=/var/lib/etcd/cert/etcd-client-key.pem save /var/lib/etcd_backup/backup_$(date "+%Y%m%d%H%M%S").db
```

还原

```
set -x
export ETCD_NAME=$(cat /usr/lib/systemd/system/etcd.service|grep ExecStart|grep -Eo "name.*-name-[0-9].*--client"|awk '{print $2}')
export ETCD_CLUSTER=$(cat /usr/lib/systemd/system/etcd.service|grep ExecStart|grep -Eo "initial-cluster.*--initial"|awk '{print $2}')
export ETCD_INITIAL_CLUSTER_TOKEN=$(cat /usr/lib/systemd/system/etcd.service|grep ExecStart|grep -Eo "initial-cluster-token.*"|awk '{print $2}')
export ETCD_INITIAL_ADVERTISE_PEER_URLS=$(cat /usr/lib/systemd/system/etcd.service|grep ExecStart|grep -Eo "initial-advertise-peer-urls.*--listen-peer"|awk '{print $2}')
ETCDCTL_API=3 etcdctl snapshot --cacert=/var/lib/etcd/cert/ca.pem --cert=/var/lib/etcd/cert/etcd-client.pem --key=/var/lib/etcd/cert/etcd-client-key.pem  restore /var/lib/etcd_backup/backup_xxx.db \
  --name $ETCD_NAME \
  --data-dir /var/lib/etcd/data.etcd \
  --initial-cluster $ETCD_CLUSTER \
  --initial-cluster-token $ETCD_INITIAL_CLUSTER_TOKEN \
  --initial-advertise-peer-urls $ETCD_INITIAL_ADVERTISE_PEER_URLS
chown -R etcd:etcd /var/lib/etcd/data.etcd
```



# Controller Manager

controller-manager里面集合了太多的controller在一个进程里面，如果我们创建和删除某种资源特别的频繁，就会使得这种controller占用大量的资源，因而我们可以选择将controller-manager里面的一些核心的controller独立进程进行部署。

# Scheduler

不同的租户是不共享虚拟机的，这样不同的租户是可以并行调度的。因为不同的租户即便进行并行调度，也不会出现冲突的现象，每个租户不是在所有节点中进行调度，而仅仅在属于这个租户的有限的节点中进行调度，大大提高了调度策略



# K8S集群跨DC分布应用分布

当一个Kubernetes跨多个AZ的时候，为了实现应用的高可用，需要将容器在多个AZ里面合理分布。

## Pod Topology Spread Constraints

拓扑约束依赖于节点标签来标识每个节点所在的拓扑域。

假设拥有一个具有以下标签的 4 节点集群：

```
NAME    STATUS   ROLES    AGE     VERSION   LABELS
node1   Ready    <none>   4m26s   v1.16.0   node=node1,zone=zoneA
node2   Ready    <none>   3m58s   v1.16.0   node=node2,zone=zoneA
node3   Ready    <none>   3m17s   v1.16.0   node=node3,zone=zoneB
node4   Ready    <none>   2m43s   v1.16.0   node=node4,zone=zoneB
```

可以定义一个或多个 topologySpreadConstraint 来指示 kube-scheduler 如何将每个传入的 Pod 根据与现有的 Pod 的关联关系在集群中部署。字段包括：

- maxSkew 描述 pod 分布不均的程度。这是给定拓扑类型中任意两个拓扑域中匹配的 pod 之间的最大允许差值。它必须大于零。

- topologyKey 是节点标签的键。如果两个节点使用此键标记并且具有相同的标签值，则调度器会将这两个节点视为处于同一拓扑中。调度器试图在每个拓扑域中放置数量均衡的 pod。

- whenUnsatisfiable 指示如果 pod 不满足扩展约束时如何处理：

- - DoNotSchedule（默认）告诉调度器不用进行调度。
  - ScheduleAnyway 告诉调度器在对最小化倾斜的节点进行优先级排序时仍对其进行调度。

- labelSelector 用于查找匹配的 pod。匹配此标签的 pod 将被统计，以确定相应拓扑域中 pod 的数量。


假设我们将Pod声明如下

```
kind: Pod
apiVersion: v1
metadata:
  name: mypod
  labels:
    foo: bar
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        foo: bar
  - maxSkew: 1
    topologyKey: node
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        foo: bar
  containers:
  - name: pause
    image: k8s.gcr.io/pause:3.1
```

## Service Topology

Service 拓扑可以让一个服务基于集群的 Node 拓扑进行流量路由。例如，一个服务可以指定流量是被优先路由到一个和客户端在同一个 Node 或者在同一可用区域的端点。

如果集群启用了 Service 拓扑功能后，就可以在 Service 配置中指定 topologyKeys 字段，从而控制 Service 的流量路由。此字段是 Node 标签的优先顺序字段，将用于在访问这个 Service 时对端点进行排序。流量会被定向到第一个标签值和源 Node 标签值相匹配的 Node。如果这个 Service 没有匹配的后端 Node，那么第二个标签会被使用做匹配，以此类推，直到没有标签。

如果没有匹配到，流量会被拒绝，就如同这个 Service 根本没有后端。

```
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
  topologyKeys:
    - "kubernetes.io/hostname"
    - "topology.kubernetes.io/zone"
    - "topology.kubernetes.io/region"
    - "*"
```

可以设置 Service 的 topologyKeys 的值，像下面的做法一样定向流量了。

- 只定向到同一个 Node 上的端点，Node 上没有端点存在时就失败：配置 ["kubernetes.io/hostname"]。
- 偏向定向到同一个 Node 上的端点，回退同一区域的端点上，然后是同一地区，其它情况下就失败：配置 ["kubernetes.io/hostname", "topology.kubernetes.io/zone", "topology.kubernetes.io/region"]。这或许很有用，例如，数据局部性很重要的情况下。
- 偏向于同一区域，但如果此区域中没有可用的终结点，则回退到任何可用的终结点：配置 ["topology.kubernetes.io/zone", "*"]

## nodeSelector & node Affinity and anti-affinity

nodeSelector：为Node规划标签，然后在创建部署的时候，通过使用nodeSelector标签来指定Pod运行在哪些节点上。

Node affinity：指定一些Pod在Node间调度的约束。支持两种形式：

requiredDuringSchedulingIgnoredDuringExecution和preferredDuringSchedulingIgnoredDuringExecution，可以认为前一种是必须满足，如果不满足则不进行调度，后一种是倾向满足，不满足的情况下会调度的不符合条件的Node上。

IgnoreDuringExecution表示如果在Pod运行期间Node的标签发生变化，导致亲和性策略不能满足，则继续运行当前的Pod。

## Inter-pod affinity and anti-affinity

允许用户通过已经运行的Pod上的标签来决定调度策略，如果Node X上运行了一个或多个满足Y条件的Pod，那么这个Pod在Node应该运行在Pod X。 

有两种类型

- requiredDuringSchedulingIgnoredDuringExecution，刚性要求，必须精确匹配
- preferredDuringSchedulingIgnoredDuringExecution，软性要求



# 遵循集群限制与解决方案

为保证大规模集群的可用性、稳定性和各项性能，请关注下表列出的限制及对应的推荐解决方案。

|      |      |      |
| ---- | ---- | ---- |
|      |      |      |

| **限制项**                     | **说明**                                                     | **推荐解决方案**                                             |
| ------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| etcd数据库大小（DB Size）      | etcd数据库大小上限为8 GB。当etcd数据库过大时，会影响其性能，包括数据读取和写入延迟、系统资源占用、选举延时等，也会导致服务和数据恢复过程更加困难和耗时。 | 控制etcd DB Size的总大小在8 GB以下。控制集群资源总量，及时清理无需使用的资源。对于高频修改类资源，建议将单个资源大小控制在100KB以下。在etcd中，每次对键值对的更新都会生成一个新的历史版本。在高频更新大数据对象的场景下，etcd保存其数据的历史版本时会消耗更多的资源。 |
| etcd中每种类型资源的数据总大小 | 如果资源对象总量过大，客户端全量访问该资源时会导致大量的系统资源消耗，严重时可能会导致API Server或自定义控制器（Controller）无法初始化。 | 控制每种资源类型的对象总大小在800 MB以下。定义新类型的CRD时，请提前规划最终期望存在的CR数，确保每种CRD资源的总大小可控。使用Helm部署Chart时，Helm会创建版本（Release）用于跟踪部署状态。默认情况下，Helm使用Secret来存储版本信息。在大规模集群中，将大量的版本信息存储在Secret中可能超出Kubernetes对Secret总大小的限制。请改为使用[Helm的SQL存储后端](https://helm.sh/docs/topics/advanced/#sql-storage-backend)。 |
| 每个命名空间的Service数量      | kubelet会把集群中定义的Service的相关信息以环境变量的形式注入到运行在该节点上的Pod中，让Pod能够通过环境变量来发现Service，并与之通信。每个命名空间中的服务数量过多会导致为Pod注入的环境变量过多，继而可能导致Pod启动缓慢甚至失败 | 将每个命名空间的Service数量控制在5,000以下。您可以选择不填充这些环境变量，将`podSpec`中的`enableServiceLinks`设置为`false`。更多信息，请参见[Accessing the Service](https://kubernetes.io/docs/tutorials/services/connect-applications-service/#accessing-the-service)。 |
| 集群的总Service数量            | Service数量过多会导致kube-proxy需要处理的网络规则增多，继而影响kube-proxy的性能。对于LoadBalancer类型的Service，Service数量过多会导致Service同步到SLB的时延增加，延迟可能达到分钟级别。 | 将所有Service的总数量控制在10,000以下。对LoadBalancer类型的Service，建议将Service总数控制在500以下。 |
| 单个Service的Endpoint最大数量  | 每个节点上运行着kube-proxy组件，用于监视（Watch）Service的相关更新，以便及时更新节点上的网络规则。当某个Service存在很多Endpoint时，Service相应的Endpoints资源也会很大，每次对Endpoints对象的更新都会导致控制面kube-apiserver和节点kube-proxy之间传递大量流量。集群规模越大，需要更新的相关数据越多，风暴效应越明显。**说明**为了解决此问题，kube-proxy在v1.19以上的集群中默认使用[EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)来提高性能。 | 将单个Service的Endpoints的后端Pod数量控制在3,000以下。在大规模集群中，应使用[EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)而非Endpoints进行网络端点的拆分和管理。拆分后，每次资源变更时传输的数据量会有效减少。如果您有自定义Controller依赖Endpoints资源进行路由决策，您仍可以使用Endpoints对象，但请确保单个Endpoints的后端Pod数量不超过1,000。超过时，Pod的Endpoints对象会自动被截断。更多信息，请参见[Over-capacity endpoints](https://kubernetes.io/docs/concepts/services-networking/service/#over-capacity-endpoints)。 |
| 所有Service Endpoint的总数量   | 集群中Endpoint数量过多可能会造成API Server负载压力过大，并导致网络性能降低。 | 将所有Service关联的Endpoint的总数量控制在64,000以下。        |
| Pending Pod的数量              | Pending Pod数量过多时，新提交的Pod可能会长时间处于等待状态，无法被调度到合适的节点上。在此过程中，如果Pod无法被调度，调度器会周期性地产生事件（event），继而可能导致事件泛滥（event storm）。 | 将Pending Pod的总数量控制在10,000以下。                      |

# **合理设置管控组件参数**

## kube-apiserver

为了避免大量并发请求导致控制面超载，kube-apiserver限制了它在特定时间内可以处理的并发请求数。一旦超出此限制，API Server将开始限流请求，向客户端返回429 HTTP响应码，即请求过多，并让客户端稍后再试。如果服务端没有任何限流措施，可能导致控制面因处理超出其承载能力的请求而过载，严重影响整个服务或集群的稳定性以及可用性。因此，推荐您在服务端配置限流机制，避免因控制面崩溃而带来更加广泛的问题。

#### **限流分类**

kube-apiserver的限流分为两种。

- v1.18以下：kube-apiserver仅支持最大并发度限流，将请求区分为读类型和写类型，通过启动参数`--max-requests-inflight`和`--max-mutating-requests-inflight`限制读写请求的最大并发度。该方式不区分请求优先级。某些低优先级的慢请求可能占用大量资源，造成API Server请求堆积，导致一些更高优先级或更紧急的请求无法及时处理。

  ACK集群Pro版支持自定义kube-apiserver的max-requests-inflight和max-mutating-requests-inflight的参数配置。更多信息，请参见[自定义Pro版集群的控制面组件参数](https://help.aliyun.com/zh/ack/ack-managed-and-ack-dedicated/user-guide/customize-ack-pro-control-plane-component-parameters)。

- v1.18及以上：引入[APF(API优先级和公平性）](https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/flow-control/)机制来进行更细粒度的流量管理，支持根据预设的规则和优先级来分类和隔离请求，从而确保更重要和紧急的请求优先处理，同时遵循一定的公平性策略，保证不同类型的请求都能够得到合理的处理机会。该特性于v1.20进入Beta阶段，默认开启。

#### **限流监测与推荐解决方案**

客户端可以通过返回状态码429或通过监控指标`apiserver_flowcontrol_rejected_requests_total`来判断服务端是否有限流行为。当观测到有限流行为时，可以通过以下方式解决。

- 监控API Server资源水位：当水位较低时，可调整`max-requests-inflight`和`max-mutating-requests-inflight`参数总和提高总的限流值。

  对于500个节点以上的集群，建议参数总和配置为2000～3000之间；对于3000个节点以上的集群，建议参数总和配置在3000～5000之间。

- 重新配置PriorityLevelConfiguration：

  - 高优先级请求：为不期望限流的请求划分新的FlowSchema，匹配高优先级的PriorityLevelConfiguration，例如`workload-high`或`exempt`。但`exempt`的请求不会被APF限流，请谨慎配置。您也可以为高优先级的请求配置新的PriorityLevelConfiguration，给予更高的并发度。
  - 低优先级请求：当的确存在某些客户端慢请求导致API Server水位高或响应慢时，可以为该类请求新增一个FlowSchema，并为该FlowSchema匹配低并发度的PriorityLevelConfiguration。

## **kube-controller-manager和kube-scheduler**

kube-controller-manager和kube-scheduler分别通过kubeAPIQPS/kubeAPIBurst、connectionQPS/connectionBurst参数控制与API Server通信的QPS。

- kube-controller-manager：对于1000个节点以上的集群，建议将kubeAPIQPS/kubeAPIBurst调整为300/500以上。
- kube-scheduler：一般无需调整。Pod速率超过300/s时建议将connectionQPS/connectionBurst调整为800/1000。

## **kubelet**

kubelet组件的`kube-api-burst/qps`默认值为5/10，一般无需调整。当您的集群出现Pod状态更新缓慢、调度延迟、存储卷挂载缓慢等显著性能问题时，建议您调大参数。

**重要**

- 调大kubelet该参数会增大kubelet与API Server的通信QPS。如果kubelet发送的请求数量过多，可能会增大API Server的负载。建议您逐步增加取值，并关注API Server的性能和资源水位，确保控制面稳定性。
- 对节点kubelet执行变更操作时，需合理控制更新频率。为保证变更过程中控制面的稳定性，ACK限制单节点池每批次的最大并行数不超过10。

# **规划集群资源弹性速率**

在大规模集群中，当集群处于稳定运行状态时，通常不会给控制面带来太大的压力。但当集群进行大规模变更操作时，例如快速创建或删除大量资源，或大规模扩缩集群节点数时，可能会造成控制面压力过大，导致集群性能下降、响应延迟，甚至服务中断。

例如，在一个5,000个节点的集群中，如果存在大量固定数量的Pod且保持稳定运行状态，那么它对控制面的压力通常不会太大。但在一个1,000个节点的集群中，如果需要在一分钟内创建10,000个短暂运行的Job，或并发扩容2,000个节点，那么控制面的压力会激增。

因此，在大规模集群中进行资源变更操作时，需根据集群运行状态谨慎规划弹性操作的变更速率，以确保集群和控制面的稳定性。

推荐操作如下。

**重要**

由于集群控制面影响因素较多，以下数字仅供参考。实际操作时，请遵循变更速率由小到大的操作顺序，观察控制面响应正常后再适当提高弹性速率。

- 节点扩缩容：对于2000个节点以上的集群，建议在通过节点池手动扩缩容节点时，单个节点池单次操作的节点数不超过100，多个节点池单次操作的总节点数不超过300。
- 应用Pod扩缩容：若您的应用关联了Service，扩缩容过程中Endpoint、EndpointSlice的更新会推送到所有节点。在节点数量多的场景下，需要更新的相关数据也很多，可能导致集群风暴效应。对5000节点以上的集群，建议非Endpoint关联的Pod更新QPS不超过300/s；Endpoints关联的Pod更新QPS不超过10/s。例如，在Deployment中声明Pod滚动更新（Rolling Update）策略时，推荐您先设置较小的`maxUnavailable`和`maxSurge`取值，以降低Pod更新速率。

# 关注集群控制面指标

您可以查看控制面组件监控大盘，获取控制面核心组件的指标清单、异常指标问题解析等。在大规模集群中，请重点关注以下指标。关于指标的使用说明、详细说明，请参见[控制面组件监控](https://help.aliyun.com/zh/ack/ack-managed-and-ack-dedicated/user-guide/monitor-control-plane-components/)。

## **管控资源水位**

目前管控组件所有资源水位均可以查看，相关指标和介绍如下：

| **指标**                 | **类型** | **说明**                                         |
| ------------------------ | -------- | ------------------------------------------------ |
| cpu_utilization_ratio    | Gauge    | CPU使用率=CPU使用量/CPU资源上限，百分比形式。    |
| memory_utilization_ratio | Gauge    | 内存使用率=内存使用量/内存资源上限，百分比形式。 |

## kube-apiserver

关于如何查看指标及指标的完整说明，请参见[kube-apiserver组件监控](https://help.aliyun.com/zh/ack/ack-managed-and-ack-dedicated/user-guide/monitor-kube-apiserver)。

- 资源数，保证资源不出现泄漏

  | **名称**     | **PromQL**                                                   | **说明**                                                     |
  | ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
  | 资源对象数量 | max by(resource)(apiserver_storage_objects)max by(resource)(etcd_object_counts) | Kubernetes管理资源数量。不同ACK版本指标名称不同：当ACK为1.22及以上版本时， 指标名字为apiserver_storage_objects当ACK为1.22及以下版本时，指标名字为etcd_object_counts。**说明**由于兼容性问题，1.22版本中apiserver_storage_objects名称和etcd_object_counts名称均存在。 |

- 请求时延

  | **名称**       | **PromQL**                                                   | **说明**                                                     |
  | -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
  | GET读请求时延  | histogram_quantile($quantile, sum(irate(apiserver_request_duration_seconds_bucket{verb="GET",resource!="",subresource!~"log\|proxy"}[$interval])) by (pod, verb, resource, subresource, scope, le)) | 展示GET请求的响应时间，维度包括API Server Pod、Verb(GET)、Resources（例如Configmaps、Pods、Leases等）、Scope（范围例如Namespace级别、Cluster级别）。 |
  | LIST读请求时延 | histogram_quantile($quantile, sum(irate(apiserver_request_duration_seconds_bucket{verb="LIST"}[$interval])) by (pod_name, verb, resource, scope, le)) | 展示LIST请求的响应时间，维度包括API Server Pod、Verb(GET)、Resources（例如Configmaps、Pods、Leases等）、Scope（范围例如Namespace级别、Cluster级别）。 |
  | 写请求时延     | histogram_quantile($quantile, sum(irate(apiserver_request_duration_seconds_bucket{verb!~"GET\|WATCH\|LIST\|CONNECT"}[$interval])) by (cluster, pod_name, verb, resource, scope, le)) | 展示Mutating请求的响应时间，维度包括API Server Pod、Verb(GET)、Resources（例如Configmaps、Pods、Leases等）、Scope（范围例如Namespace级别、Cluster级别）。 |

- 请求限流

  | **名称**     | **PromQL**                                                   | **说明**                                               |
  | ------------ | ------------------------------------------------------------ | ------------------------------------------------------ |
  | 请求限流速率 | sum(irate(apiserver_dropped_requests_total{request_kind="readOnly"}[$interval])) by (name)sum(irate(apiserver_dropped_requests_total{request_kind="mutating"}[$interval])) by (name) | API Server限流速率，**No data**或者**0**表示没有限流。 |

## **kube-scheduler**

关于如何查看指标及指标的完整说明，请参见[kube-scheduler组件监控](https://help.aliyun.com/zh/ack/ack-managed-and-ack-dedicated/user-guide/monitor-kube-scheduler)。

- Pending Pod数

  | **名称**               | **PromQL**                                  | **说明**                                                     |
  | ---------------------- | ------------------------------------------- | ------------------------------------------------------------ |
  | Scheduler Pending Pods | scheduler_pending_pods{job="ack-scheduler"} | Pending Pod的数量。队列种类如下：**unschedulable**：表示不可调度的Pod数量。**backoff**：表示backoffQ的Pod数量。**active**：表示activeQ的Pod数量。 |

- 请求时延

  | **名称**         | **PromQL**                                                   | **说明**                                                     |
  | ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
  | Kube API请求时延 | histogram_quantile($quantile, sum(rate(rest_client_request_duration_seconds_bucket{job="ack-scheduler"}[$interval])) by (verb,url,le)) | 调度器对kube-apiserver发起的HTTP请求时延，从方法（Verb）和请求URL维度进行分析。 |

## **kube-controller-manager**

关于如何查看指标及指标的完整说明，请参见[kube-controller-manager组件监控](https://help.aliyun.com/zh/ack/ack-managed-and-ack-dedicated/user-guide/monitor-kube-controller-manager)。

- workqueue depth

  | **名称**      | **PromQL**                                                   | **说明**                      |
  | ------------- | ------------------------------------------------------------ | ----------------------------- |
  | Workqueue深度 | sum(rate(workqueue_depth{job="ack-kube-controller-manager"}[$interval])) by (name) | Workqueue各资源当前队列深度。 |

- workqueue latency

  | **名称**          | **PromQL**                                                   | **说明**                                              |
  | ----------------- | ------------------------------------------------------------ | ----------------------------------------------------- |
  | Workqueue处理时延 | histogram_quantile($quantile, sum(rate(workqueue_queue_duration_seconds_bucket{job="ack-kube-controller-manager"}[5m])) by (name, le)) | 记录Workqueue中任务从加入队列到开始处理所经历的时间。 |

## **etcd**

关于如何查看指标及指标的完整说明，请参见[etcd组件监控](https://help.aliyun.com/zh/ack/ack-managed-and-ack-dedicated/user-guide/monitor-etcd)。

- kv总数

  | **名称** | **PromQL**                     | **说明**           |
  | -------- | ------------------------------ | ------------------ |
  | kv总数   | etcd_debugging_mvcc_keys_total | etcd集群kv对总数。 |

- 数据库大小（DB Size）

  | **名称** | **PromQL**                                                   | **说明**                                             |
  | -------- | ------------------------------------------------------------ | ---------------------------------------------------- |
  | 磁盘大小 | etcd_mvcc_db_total_size_in_bytesetcd_mvcc_db_total_size_in_use_in_bytes | etcd backend db总大小。etcd backend db实际使用大小。 |

# 优化客户端访问集群模式

在Kubernetes集群中，客户端（客户业务应用程序或kubectl等）通过API Server来获取集群资源信息。随着集群中的资源数量的增加，如果客户端以相同的频率进行请求，这些请求可能会给集群控制面带来更大的负载，导致控制面响应延迟缓慢，甚至导致管控面雪崩。在访问集群资源时，需了解并规划访问资源的大小和频率。大规模集群使用建议如下。

## **优先使用informer访问本地缓存数据**

优先使用client-go的[informer](https://pkg.go.dev/k8s.io/client-go/informers)获取资源，通过本地缓存（Cache）查询数据，避免List请求直接访问API Server，以减少API Server的负载压力。

## **优化通过API Server获取资源的方式**

对于未访问过的本地缓存，仍需直接通过API Server获取资源。但可以遵循以下建议。

- 在List请求中设置`resourceVersion=0`。

  `resourceVersion`表示资源状态的版本。设置为`0`时，请求会获取 API Server的缓存数据，而非直接访问etcd，减少API Server与etcd之间的内部交互次数，更快地响应客户端List请求。示例如下。

   

  ```shell
  k8sClient.CoreV1().Pods("").List(metav1.ListOptions{ResourceVersion: "0"})
  ```

- 避免全量List资源，防止数据检索量过大。

  为了减少请求返回的数据量，应使用过滤条件（Filter）来限定List请求的范围，例如lable-selector（基于资源标签筛选）或field-selector（基于资源字段筛选）。

  **说明**

  etcd是一个键值（KV）存储系统，本身不具备按label或field过滤数据的功能，请求带有的过滤条件实际由API Server处理。因此，当使用Filter功能时，建议同时将List请求的`resourceVersion`设置为`0`。请求数据将从API Server的缓存中获取，而不会直接访问etcd，减少对etcd的压力。

- 使用Protobuf（而非JSON）访问非CRD资源。

  API Server可以以不同的数据格式向客户端返回资源对象，包括JSON和Protobuf。默认情况下，当客户端请求Kubernetes API时，Kubernetes返回序列化为JSON的对象，其内容类型（Content-Type）为`application/json`。 客户端可以指定请求使用Protobuf格式的数据，Protobuf在内存使用和网络传输流量方面相较JSON更有优势。

  但并非所有API资源类型都支持Protobuf。发送请求时，可以在`Accept`请求头中指定多种内容类型（例如 `application/json`、`application/vnd.kubernetes.protobuf`），从而支持在无法使用Protobuf时回退到默认的JSON格式。更多信息，请参见[Alternate representations of resources](https://kubernetes.io/docs/reference/using-api/api-concepts/#alternate-representations-of-resources) 。示例如下。

   

  ```shell
  Accept: application/vnd.kubernetes.protobuf, application/json
  ```

## **使用中心化控制器**

避免在每个节点上都创建独立的控制器用于Watch集群的全量数据。在这种情况下，控制器启动时，将几乎同时向API Server发送大量的List请求以同步当前的集群状态，对控制面造成巨大压力，继而导致服务不稳定或崩溃。

为了避免此问题，建议采用中心化的控制器设计，为整个集群创建一个或一组集中管理的控制器实例，运行在单个节点或少数几个节点上。中心化的控制器会负责监听和处理所需的集群数据，仅启动一次（或少数几次）List请求，并仅维护必要数量的Watch连接，大大减少了对API Server的压力。

# **合理规划大规模工作负载**

## **停用自动挂载默认的Service Account**

为了确保Pod中的Secret的同步更新，kubelet会为Pod配置的每个Secret建立一个Watch长连接。Watch机制可以让kubelet实时接收Secret的更新通知。但当所有节点创建的Watch数量总和过多时，大量Watch连接可能会影响集群控制面的性能。

- Kubernetes 1.22版本以前：创建Pod时，如果未指定ServiceAccount，Kubernetes会自动挂载默认的ServiceAccount作为Pod的Secret，使得Pod内部的应用能够与API Server安全通信。

  对于批处理系统和无需访问API Server的业务Pod，建议您显式声明禁止自动挂载ServiceAccount Token，以避免创建相关的Secret和Watch（更多信息，请参见[automountServiceAccountToken](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#opt-out-of-api-credential-automounting)）。在大规模集群中，该操作可以避免创建不必要的Secret和与API Server的Watch连接，减轻集群控制面的负担。

- Kubernetes 1.22及之后：使用TokenRequest API来获取一个短期的、自动轮换的令牌，并以[投射卷](https://kubernetes.io/docs/concepts/storage/projected-volumes/#serviceaccounttoken)的形式进行挂载此令牌。在提高Secret安全性的同时，该操作还能减少kubelet为每个ServiceAccount的Secret建立的Watch连接，降低集群的性能开销。

  关于如何启用ServiceAccount Token投射卷功能，请参见[部署服务账户令牌卷投影](https://help.aliyun.com/zh/ack/ack-managed-and-ack-dedicated/security-and-compliance/enable-service-account-token-volume-projection)。

## **控制**Kubernetes **Object数量和大小**

请及时清理无需使用的Kubernetes资源，例如ConfigMap、Secret、PVC等，减少系统资源的占用，保持集群健康、高效运转。有以下使用建议。

- 限制Deployment历史记录数：[revisionHistoryLimit](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#revision-history-limit)声明了要为Deployment保留多少个旧的ReplicaSet。如果取值太高，Kubernetes会保留很多历史版本的ReplicaSet，增加kube-controller-manager的管理负担。在大规模集群中，如果Deployment较多且更新频繁，您可以调低Deployment的revisionHistoryLimit的取值，清理旧的ReplicaSet。Deployment的revisionHistoryLimit默认取值为10。
- 清理无需使用的Job和相关的Pod：如果集群中通过CronJob或其他机制创建了大量的作业（Job）对象，请使用[ttlSecondsAfterFinished](https://kubernetes.io/docs/concepts/workloads/controllers/ttlafterfinished/)来自动清理集群中较旧的Pod，指定在某个时间周期后Job和相关的Pod将完成自动清理（删除）。

## **合理配置Informer类型组件的资源配置**

Informer类型的组件主要用于监控和同步Kubernetes集群的资源状态。Informer类组件会建立对集群API Server资源状态的Watch连接，并在本地维护一个资源对象的缓存，以便快速响应资源状态的变化。

对于Informer类型的组件，例如Controller组件、kube-scheduler等，组件内存占用与其Watch的资源大小相关。大规模集群下，请关注该类组件的内存资源消耗，避免组件OOM。组件频繁OOM会导致组件持续资源监听出现问题。组件频繁重启时，每次执行的List-Watch操作也会对集群控制面（尤其是API Server）造成额外的压力。