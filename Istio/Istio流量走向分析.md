

# Envoy启动流程分析

前面有说道 `istio-proxy`容器中有两个进程`Pilot-agent`和`envoy`

```bash
# kubectl -n istio-test exec productpage-v1-5d9b4c9849-99kff -c istio-proxy -- ps -ef
UID          PID    PPID  C STIME TTY          TIME CMD
istio-p+       1       0  0 Aug24 ?        00:02:46 /usr/local/bin/pilot-agent proxy sidecar --domain istio-test.svc.cluster.local --proxyLogLevel=warning --proxyComponentLogLevel=misc:error --log_output_level=default:info --concurrency 2
istio-p+      18       1  0 Aug24 ?        00:14:18 /usr/local/bin/envoy -c etc/istio/proxy/envoy-rev0.json --restart-epoch 0 --drain-time-s 45 --drain-strategy immediate --parent-shutdown-time-s 60 --local-address-ip-version v4 --bootstrap-version 3 --file-flush-interval-msec 1000 --disable-hot-restart --log-format %Y-%m-%dT%T.%fZ.%l.envoy %n.%v -l warning --component-log-level misc:error --concurrency 2
```

Envoy的大部分配置都是dynamic resource，包括网格中服务相关的service cluster, listener, route规则等。这些dynamic resource是通过xDS接口从Istio控制面中动态获取的。但Envoy如何知道xDS server的地址呢？这是在Envoy初始化配置文件中以static resource的方式配置的。

## Envoy初始配置文件

Pilot-agent进程根据启动参数和K8S API Server中的配置信息生成Envoy的初始配置文件，并负责启动Envoy进程。从ps命令输出可以看到Pilot-agent在启动Envoy进程时传入了控制面的地址，并为Envoy生成了一个初始化配置文件envoy-rev0.json。

可以使用下面的命令将productpage pod中该文件导出来查看其中的内容：

```bash
kubectl -n istio-test exec productpage-v1-5d9b4c9849-99kff -c istio-proxy -- cat /etc/istio/proxy/envoy-rev0.json > envoy-rev0.json
```

里面大致分了如下结构

```bash
JSON
|-- node   #包含了Envoy所在节点相关信息。
|-- stats_config  
|-- admin  #配置Envoy的日志路径以及管理端口
|-- dynamic_resources  #配置动态资源,这里配置了ADS服务器
|-- static_resources #配置静态资源，包括了prometheus_stats、xds-grpc和zipkin三个cluster和一个在15090上监听的listener
|-- tracing  #配置分布式链路跟踪
```

## Envoy配置

`Pilot-agent`生成 `envoy-rev0.json`之后，`envoy` 进程根据  `envoy-rev0.json`根据pilot 地址，采用xDS接口去获取了Listener，Cluster，Route等动态配置信息

如果要获取完整的envoy配置，得用下面命令

```bash
kubectl -n istio-test exec productpage-v1-5d9b4c9849-99kff -c istio-proxy -- curl http://127.0.0.1:15000/config_dump > config_dump
```

该文件内容长达近43000行，打开浏览下结构，发现大致分成

```bash
JSON
|-- BootstrapConfigDump
|-- ClustersConfigDump
|-- ListenersConfigDump
|-- RoutesConfigDump
|-- SecretsConfigDump
```

### Bootstrap

和`envoy-rev0.json`一样，是初始化配置

### Clusters

在Envoy中，Cluster是一个服务集群，Cluster中包含一个到多个endpoint，每个endpoint都可以提供服务，Envoy根据负载均衡算法将请求发送到这些endpoint中。

clusters配置中包含`static_clusters`和`dynamic_active_clusters`两部分，其中static_clusters是来自于envoy-rev0.json的初始化配置中的prometheus_stats、xDS server和zipkin server信息。dynamic_active_clusters是通过xDS接口从Istio控制面获取的动态服务信息

Dynamic Cluster中有以下几类Cluster：

#### Outbound Cluster

这部分的Cluster占了绝大多数，该类Cluster对应于Envoy所在节点的外部服务。以reviews为例，对于Productpage来说,reviews是一个外部服务，因此其Cluster名称中包含outbound字样。

从reviews 服务对应的cluster配置中可以看到，其类型为EDS，即表示该Cluster的endpoint来自于动态发现，动态发现中eds_config则指向了ads，最终指向static Resource中配置的xds-grpc cluster,即Pilot的地址。

```bash
{
    "version_info": "2019-12-04T03:08:06Z/13",
    "cluster": {
        "name": "outbound|9080||reviews.default.svc.cluster.local",
        "type": "EDS",
        "eds_cluster_config": {
            "eds_config": {
                "ads": {}
            },
            "service_name": "outbound|9080||reviews.default.svc.cluster.local"
        },
```

可以通过Pilot的调试接口获取该Cluster的endpoint：

```json
curl http://10.97.222.108:15014/debug/edsz > pilot_eds_dump
```

仔细查看其文件，可以看到reviews cluster配置了3个endpoint地址，是reviews的pod ip。

#### Inbound Cluster

该类Cluster对应于Envoy所在节点上的服务。如果该服务接收到请求，当然就是一个入站请求。

对于Productpage Pod上的Envoy，其对应的Inbound Cluster只有一个，即productpage。该cluster对应的host为127.0.0.1,即环回地址上productpage的监听端口。由于iptable规则中排除了127.0.0.1,入站请求通过该Inbound cluster处理后将跳过Envoy，直接发送给Productpage进程处理。

#### BlackHoleCluster

这是一个特殊的Cluster，并没有配置后端处理请求的Host。如其名字所暗示的一样，请求进入后将被直接丢弃掉。如果一个请求没有找到其对的目的服务，则被发到cluster

#### PassthroughCluster

和BlackHoleCluter相反，发向PassthroughCluster的请求会被直接发送到其请求中要求的原始目地的，Envoy不会对请求进行重新路由。

### Listeners

Envoy采用listener来接收并处理downstream发过来的请求，listener采用了插件式的架构，可以通过配置不同的filter在Listener中插入不同的处理逻辑。

Listener可以绑定到IP Socket或者Unix Domain Socket上，以接收来自客户端的请求;也可以不绑定，而是接收从其他listener转发来的数据。Istio利用了Envoy listener的这一特点，通过VirtualOutboundListener在一个端口接收所有出向请求，然后再按照请求的端口分别转发给不同的listener分别处理。

#### VirtualOutbound Listener

Envoy创建了一个在15001端口监听的入口监听器。Iptable将Envoy所在Pod的对外请求拦截后发向本地的15001端口，该监听器接收后并不进行业务处理，而是根据请求的目的端口分发给其他监听器处理。该监听器取名为"virtual”（虚拟）监听器也是这个原因。

Envoy是如何做到按请求的目的端口进行分发的呢？ 从`VirtualOutbound Listener`的配置中可以看到[use_original_dest](https://www.envoyproxy.io/docs/envoy/latest/configuration/listeners/listener_filters/original_dst_filter)被设置为true,表明监听器将接收到的请求转交给和请求原目的地址关联的listener进行处理。

如果在Enovy的配置中找不到和请求目的地端口的listener，则将会根据Istio的outboundTrafficPolicy全局配置选项进行处理。存在两种情况：

- 如果[outboundTrafficPolicy](https://istio.io/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-OutboundTrafficPolicy)设置为ALLOW_ANY：Mesh允许发向任何外部服务的请求，不管该服务是否在Pilot的服务注册表中。在该策略下，Pilot将会在下发给Enovy的VirtualOutbound Listener加入一个upstream cluster为[PassthroughCluster](https://zhaohuabing.com/post/2018-09-25-istio-traffic-management-impl-intro/#passthroughcluster)的TCP proxy filter，找不到匹配端口listener的请求会被该TCP proxy filter处理，请求将会被发送到其IP头中的原始目的地地址。
- 如果[outboundTrafficPolicy](https://istio.io/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-OutboundTrafficPolicy)设置为REGISTRY_ONLY：只允许发向Pilot服务注册表中存在的服务的对外请求。在该策略下，Pilot将会在下发给Enovy的VirtualOutbound Listener加入一个upstream cluster为[BlackHoleCluster](https://zhaohuabing.com/post/2018-09-25-istio-traffic-management-impl-intro/#blackholecluster)的TCP proxy filter，找不到匹配端口listener的请求会被该TCP proxy filter处理，由于BlackHoleCluster中没有配置upstteam host，请求实际上会被丢弃。

#### Outbound Listener

Envoy为网格中的外部服务按端口创建多个Listener，以用于处理出向请求。

Productpage Pod中的Envoy创建了多个Outbound Listener

- 0.0.0.0_9080 :处理对details,reviews和rating服务的出向请求
- 0.0.0.0_9411 :处理对zipkin的出向请求
- 0.0.0.0_3000 :处理对grafana的出向请求
- 0.0.0.0_15014 :处理对citadel、galley、pilot、(Mixer)policy、(Mixer)telemetry的出向请求
- 0.0.0.0_15004 :处理对(Mixer)policy、(Mixer)telemetry的出向请求
- ……

这里主要分析一下9080这个业务端口的Listenrer。和Outbound Listener一样，该Listener同样配置了"bind_to_port”: false属性，因此该listener也没有被绑定到tcp端口上，其接收到的所有请求都转发自15001端口的Virtual listener。

监听器name为0.0.0.0_9080,推测其含义应为匹配发向任意IP的9080的请求，从bookinfo程序结构可以看到该程序中的productpage,revirews,ratings,details四个service都是9080端口，那么Envoy如何区别处理这四个service呢？

首先需要区分入向（发送给productpage）请求和出向（发送给其他几个服务）请求：

- 发给productpage的入向请求，Iptables规则会将其重定向到15006端口的VirtualInbound listener上，因此不会进入0.0.0.0_9080 listener处理。
- 从productpage外发给reviews、details和ratings的出向请求，virtualOutbound listener无法找到和其目的IP完全匹配的listener，因此根据通配原则转交给0.0.0.0_9080这个Outbound Listener处理。

#### VirtualInbound Listener

在较早的版本中，Istio采用同一个VirtualListener在端口15001上同时处理入向和出向的请求。该方案存在一些潜在的问题，例如可能会导致出现死循环，参见[这个PR](https://github.com/istio/istio/pull/15713)。在1.4中，Istio为Envoy单独创建了一个VirtualInboundListener，在15006端口监听入向请求，原来的15001端口只用于处理出向请求。

另外一个变化是当VirtualInboundListener接收到请求后，将直接在VirtualInboundListener采用一系列filterChain对入向请求进行处理，而不是像VirtualOutboundListener一样分发给其它独立的Listener进行处理。

这样修改后，Envoy配置中入向和出向的请求处理流程被完全拆分开，请求处理流程更为清晰，可以避免由于配置导致的一些潜在错误。

### Routes

配置Envoy的路由规则。Istio下发的缺省路由规则中对每个端口设置了一个路由规则，根据host来对请求进行路由分发。

下面是Proudctpage服务中9080的路由配置，从文件中可以看到对应了5个`virtual host`，分别是details、productpage、ratings、reviews和allow_any，前三个virtual host分别对应到不同服务的`outbound cluster`。最后一个对应到`PassthroughCluster`,即当入向的i请求没有找到对应的服务时，也会让其直接通过。

# BookInfo流量分析

以官方的bookinfo为例, 来分析流量走向

Productpage服务调用Reviews服务的请求流程:



