# 最大规模

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