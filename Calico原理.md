# Calico简述

calico是一个比较有趣的虚拟网络解决方案，完全利用路由规则实现动态组网，通过BGP协议通告路由。

calico的优势:

- endpoints组成的网络是单纯的三层网络，报文的流向完全通过路由规则控制，没有overlay等额外开销。
- calico的endpoint可以漂移，并且实现了acl。

calico的缺点:

- 路由的数目与容器数目相同，非常容易超过路由器、三层交换、甚至node的处理能力，从而限制了整个网络的扩张。
- calico的每个node上会设置大量（海量)的iptables规则、路由，运维、排障难度大。
- calico的原理决定了它不可能支持VPC，容器只能从calico设置的网段中获取ip。
- calico目前的实现没有流量控制的功能，会出现少数容器抢占node多数带宽的情况。
- calico的网络规模受到BGP网络规模的限制。

# 专有名词

- endpoint:  接入到calico网络中的网卡称为endpoint
- AS:        网络自治系统，通过BGP协议与其它AS网络交换路由信息
- ibgp:      AS内部的BGP Speaker，与同一个AS内部的ibgp、ebgp交换路由信息。
- ebgp:      AS边界的BGP Speaker，与同一个AS内部的ibgp、其它AS的ebgp交换路由信息。
- workloadEndpoint:  虚拟机、容器使用的endpoint
- hostEndpoints:     物理机(node)的地址

# 组网原理

calico组网的核心原理就是IP路由，每个容器或者虚拟机会分配一个workload-endpoint(wl)。

从nodeA上的容器A内访问nodeB上的容器B时:

```
+--------------------+              +--------------------+ 
|   +------------+   |              |   +------------+   | 
|   |            |   |              |   |            |   | 
|   |    ConA    |   |              |   |    ConB    |   | 
|   |            |   |              |   |            |   | 
|   +-----+------+   |              |   +-----+------+   | 
|         |veth      |              |         |veth      | 
|       wl-A         |              |       wl-B         | 
|         |          |              |         |          |
+-------node-A-------+              +-------node-B-------+ 
        |    |                               |    |
        |    | type1.  in the same lan       |    |
        |    +-------------------------------+    |
        |                                         |
        |      type2. in different network        |
        |             +-------------+             |
        |             |             |             |
        +-------------+   Routers   |-------------+
                      |             |
                      +-------------+

从ConA中发送给ConB的报文被nodeA的wl-A接收，根据nodeA上的路由规则，经过各种iptables规则后，转发到nodeB。

如果nodeA和nodeB在同一个二层网段，下一条地址直接就是node-B，经过二层交换机即可到达。
如果nodeA和nodeB在不同的网段，报文被路由到下一跳，经过三层交换或路由器，一步步跳转到node-B。
```

核心问题是，nodeA怎样得知下一跳的地址？答案是node之间通过BGP协议交换路由信息。

每个node上运行一个软路由软件bird，并且被设置成BGP Speaker，与其它node通过BGP协议交换路由信息。

可以简单理解为，每一个node都会向其它node通知这样的信息:

```
我是X.X.X.X，某个IP或者网段在我这里，它们的下一跳地址是我。
```

通过这种方式每个node知晓了每个workload-endpoint的下一跳地址。

# BGP与AS

BGP是路由器之间的通信协议，主要用于AS（Autonomous System,自治系统）之间的互联。

AS是一个自治的网络，拥有独立的交换机、路由器等，可以独立运转。

每个AS拥有一个全球统一分配的16位的ID号，64512到65535共1023个AS号码可以用于私有网络。

calico默认使用的AS号是64512，可以修改：

```
calicoctl config get asNumber         //查看
calicoctl config set asNumber 64512   //设置
```

AS内部有多个BGP speaker，分为ibgp、ebgp，ebgp与其它AS中的ebgp建立BGP连接。

AS内部的BGP speaker通过BGP协议交换路由信息，最终每一个BGP speaker拥有整个AS的路由信息。

BGP speaker一般是网络中的物理路由器，可以形象的理解为: **`calico将node改造成了一个软路由器（通过软路由软件bird)node上的运行的虚拟机或者容器通过node与外部沟通`**

AS内部的BGP Speaker之间有两种互联方式:

- Mesh: BGP Speaker之间全互联，网络成网状
- RR:   Router reflection模式，BGP Speaker连接到一个或多个中心BGP Speaker，网络成星状

# BGP Speaker 全互联模式(node-to-node mesh)

全互联模式，就是一个BGP Speaker需要与其它所有的BGP Speaker建立bgp连接(形成一个bgp mesh)。

网络中bgp总连接数是按照O(n^2)增长的，有太多的BGP Speaker时，会消耗大量的连接。

calico默认使用全互联的方式，扩展性比较差，只能支持小规模集群

> say 50 nodes - although this limit is not set in stone and Calico has been deployed with over 100 nodes in a full mesh topology

可以打开/关闭全互联模式：

```
calicoctl config set nodeTonodeMesh off
calicoctl config set nodeTonodeMesh on
```

# BGP Speaker RR模式

RR模式，就是在网络中指定一个或多个BGP Speaker作为Router Reflection，RR与所有的BGP Speaker建立BGP连接。

每个BGP Speaker只需要与RR交换路由信息，就可以得到全网路由信息。

RR则必须与所有的BGP Speaker建立BGP连接，以保证能够得到全网路由信息。

在calico中可以通过Global Peer实现RR模式。

Global Peer是一个BGP Speaker，需要手动在calico中创建，所有的node都会与Global peer建立BGP连接。

```
A global BGP peer is a BGP agent that peers with every calico node in the network. 
A typical use case for a global peer might be a mid-scale deployment where all of
the calico nodes are on the same L2 network and are each peering with the same Route
Reflector (or set of Route Reflectors).
```

关闭了全互联模式后，再将RR作为Global Peers添加到calico中，calico网络就切换到了RR模式，可以支撑容纳更多的node。

calico中也可以通过node Peer手动构建BGP Speaker（也就是node）之间的BGP连接。

node Peer就是手动创建的BGP Speaker，只有指定的node会与其建立连接。

```
A BGP peer can also be added at the node scope, meaning only a single specified node will peer with it. BGP peer resources of this nature must specify a node to inform calico which node this peer is targeting.
```

因此，可以为每一个node指定不同的BGP Peer，实现更精细的规划。

例如当集群规模进一步扩大的时候，可以使用AS Per Pack model:

`每个机架是一个AS, node只与所在机架TOR交换机建立BGP连接, TOR交换机之间作为各自的ebgp全互联`

# calico网络的部署

calico网络对底层的网络的要求很少，只要求node之间能够通过IP联通。

在calico中，全网路由的数目和endpoints的数目一致，通过为node分配网段，可以减少路由数目，但不会改变数量级。

如果有1万个endpoints，那么就至少要有一台能够处理1万条路由的设备。

无论用哪种方式部署始终会有一台设备上存放着calico全网的路由。

当要部署calico网络的时候，第一步就是要确认，网络中处理能力最强的设备最多能设置多少条路由。

# node的报文处理过程



# calicoctl

calicoctl是calico访问程序, 需要能够连接 `Etcd`或`APIServer`

有二进制和容器两种形式, 这里我们以二进制举例, 先下载

```
curl -O -L  https://github.com/projectcalico/calicoctl/releases/download/v3.16.3/calicoctl
```

然后写配置文件 `/etc/calico/calicoctl.cfg`, 可以直连etcd也可以通过kubeconfig连接apiserver

```
[root@k8s-master etc]# cat /etc/calico/calicoctl.cfg 
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  datastoreType: "kubernetes"
  kubeconfig: "/root/.kube/config"
```

