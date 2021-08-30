# 什么是Istio

开源服务网络，独立语言之外提供无侵入的微服务框架和服务网络

通常来说，Istio主要和kubernetes一同构建云原生的微服务体系

## 基本概念

- **Host**：能够进行网络通信的实体（如移动设备、服务器上的应用程序）。在此文档中，主机是逻辑网络应用程序。一块物理硬件上可能运行有多个主机，只要它们是可以独立寻址的。在EDS接口中，也使用“Endpoint”来表示一个应用实例，对应一个IP+Port的组合。
- **Downstream**：下游主机连接到 Envoy，发送请求并接收响应。
- **Upstream**：上游主机接收来自 Envoy 的连接和请求，并返回响应。
- **Listener**：监听器是命名网地址（例如，端口、unix domain socket等)，可以被下游客户端连接。Envoy 暴露一个或者多个监听器给下游主机连接。在Envoy中,Listener可以绑定到端口上直接对外服务，也可以不绑定到端口上，而是接收其他listener转发的请求。
- **Cluster**：集群是指 Envoy 连接的一组上游主机，集群中的主机是对等的，对外提供相同的服务，组成了一个可以提供负载均衡和高可用的服务集群。Envoy 通过服务发现来发现集群的成员。可以选择通过主动健康检查来确定集群成员的健康状态。Envoy 通过负载均衡策略决定将请求路由到哪个集群成员。

# 架构

Istio 服务网格从逻辑上分为**数据平面**和**控制平面**。

- **数据平面** 由一组智能代理（[Envoy](https://www.envoyproxy.io/)）组成，被部署为 Sidecar。这些代理负责协调和控制微服务之间的所有网络通信。它们还收集和报告所有网格流量的遥测数据。
- **控制平面（Istiod）** 管理并配置代理来进行流量路由。

## Envoy

Envoy 是用 C++ 开发的高性能代理，用于协调服务网格中所有服务的入站和出站流量。Envoy 代理是唯一与数据平面流量交互的 Istio 组件。

和k8s配合时一般作为sidecar部署到每个应用里面，接管所有的应用进出流量

### 管理接口

Envoy提供了管理接口，缺省为localhost的15000端口，可以获取listener，cluster以及完整的配置数据导出功能。

```bash
# kubectl -n istio-test exec productpage-v1-5d9b4c9849-99kff -c istio-proxy curl http://127.0.0.1:15000/help
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1490    0  1490    0     0  1455k      0 --:--:-- --:--:-- --:--:-- 1455k
admin commands are:
  /: Admin home page
  /certs: print certs on machine
  /clusters: upstream cluster status
  /config_dump: dump current Envoy configs (experimental)
  /contention: dump current Envoy mutex contention stats (if enabled)
  /cpuprofiler: enable/disable the CPU profiler
  /drain_listeners: drain listeners
  /healthcheck/fail: cause the server to fail health checks
  /healthcheck/ok: cause the server to pass health checks
  /heapprofiler: enable/disable the heap profiler
  /help: print out list of admin commands
  /hot_restart_version: print the hot restart compatibility version
  /init_dump: dump current Envoy init manager information (experimental)
  /listeners: print listener info
  /logging: query/change logging levels
  /memory: print current allocation/heap usage
  /quitquitquit: exit the server
  /ready: print server state, return 200 if LIVE, otherwise return 503
  /reopen_logs: reopen access logs
  /reset_counters: reset all counters to zero
  /runtime: print runtime values
  /runtime_modify: modify runtime values
  /server_info: print server version/status information
  /stats: print server stats
  /stats/prometheus: print server stats in prometheus format
  /stats/recentlookups: Show recent stat-name lookups
  /stats/recentlookups/clear: clear list of stat-name lookups and counter
  /stats/recentlookups/disable: disable recording of reset stat-name lookup names
  /stats/recentlookups/enable: enable recording of reset stat-name lookup names
```

### 端口

进入productpage pod 中的istio-proxy(Envoy) container，可以看到有下面的监听端口

- 9080: productpage进程对外提供的服务端口
- 15001: Envoy的Virtual Outbound监听器，iptable会将productpage服务发出的出向流量导入该端口中由Envoy进行处理
- 15006: Envoy的Virtual Inbound监听器，iptable会将发到productpage的入向流量导入该端口中由Envoy进行处理
- 15000: Envoy管理端口，该端口绑定在本地环回地址上，只能在Pod内访问。
- 15090：指向127.0.0.1：15000/stats/prometheus, 用于对外提供Envoy的性能统计指标

```bash
# kubectl -n istio-test exec productpage-v1-5d9b4c9849-99kff -c istio-proxy -- netstat -ln
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 0.0.0.0:15021           0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:15090           0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:15000         0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:9080            0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:15001           0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:15004         0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:15006           0.0.0.0:*               LISTEN
tcp6       0      0 :::15020                :::*                    LISTEN
Active UNIX domain sockets (only servers)
Proto RefCnt Flags       Type       State         I-Node   Path
unix  2      [ ACC ]     STREAM     LISTENING     2887483574 etc/istio/proxy/SDS
unix  2      [ ACC ]     STREAM     LISTENING     2887483576 etc/istio/proxy/XDS
```

## Istiod

老版本的控制平台是分成了3个组件（Pilot、Mixer、Citadel），从1.5版本以后合成了一个Istiod。

Istiod 提供服务发现、配置和证书管理。里面包含了以前的pilot服务

Istiod 将控制流量行为的高级路由规则转换为 Envoy 特定的配置，并在运行时将其传播给 Sidecar。Pilot 提取了特定平台的服务发现机制，并将其综合为一种标准格式，任何符合 [Envoy API](https://www.envoyproxy.io/docs/envoy/latest/api/api) 的 Sidecar 都可以使用。

# 数据面

## 注入

数据面主要是通过注入来实现的，可以自动也可以手动

- 自动注入: 利用 [Kubernetes Dynamic Admission Webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/) 对 新建的pod 进行注入: init container + sidecar， 需要用 `istio-injection=enabled` 标记想部署应用程序的命名空间
- 手动注入: 使用`istioctl kube-inject`

不论是手动注入还是自动注入，Istio 都从 `istio-sidecar-injector` 和的 `istio` 两个 Configmap 对象中获取配置。

注入Pod内容:

- istio-init: 通过配置iptables来劫持Pod中的流量
- istio-proxy: 两个进程pilot-agent和envoy, pilot-agent 进行初始化并启动envoy

### istio-init

数据面的每个Pod会被注入一个名为`istio-init` 的initContainer，启动命令是

```bash
istio-iptables -p 15001 -z 15006 -u 1337 -m REDIRECT -i '*' -x "" -b * -d "15090,15021,15020"
```

istio-iptables 源码地址为 https://github.com/istio/istio/tree/master/tools/istio-iptables

### istio-proxy

istio-proxy容器中有两个进程pilot-agent和envoy:

```bash
~ % kubectl exec productpage-v1-f8c8fb8-wgmzk -c istio-proxy -- ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
istio-p+     1     0  0 Jan03 ?        00:00:27 /usr/local/bin/pilot-agent proxy sidecar --configPath /etc/istio/proxy --binaryPath /usr/local/bin/envoy --serviceCluster productpage --drainDuration 45s --parentShutdownDuration 1m0s --discoveryAddress istio-pilot.istio-system:15007 --discoveryRefreshDelay 1s --zipkinAddress zipkin.istio-system:9411 --connectTimeout 10s --proxyAdminPort 15000 --controlPlaneAuthPolicy NONE
istio-p+    21     1  0 Jan03 ?        01:26:24 /usr/local/bin/envoy -c /etc/istio/proxy/envoy-rev0.json --restart-epoch 0 --drain-time-s 45 --parent-shutdown-time-s 60 --service-cluster productpage --service-node sidecar~172.18.3.12~productpage-v1-f8c8fb8-wgmzk.default~default.svc.cluster.local --max-obj-name-len 189 --allow-unknown-fields -l warn --v2-config-only
```

可以看到:

- `/usr/local/bin/pilot-agent` 是 `/usr/local/bin/envoy` 的父进程, Pilot-agent进程根据启动参数和K8S API Server中的配置信息生成Envoy的初始配置文件(`/etc/istio/proxy/envoy-rev0.json`)，并负责启动Envoy进程
- pilot-agent 的启动参数里包括: discoveryAddress(pilot服务地址), Envoy 二进制文件的位置, 服务集群名, 监控指标上报地址, Envoy 的管理端口, 热重启时间等

Envoy配置初始化流程:

1. Pilot-agent根据启动参数和K8S API Server中的配置信息生成Envoy的初始配置文件envoy-rev0.json，该文件告诉Envoy从xDS server中获取动态配置信息，并配置了xDS server的地址信息，即控制面的Pilot
2. Pilot-agent使用envoy-rev0.json启动Envoy进程
3. Envoy根据初始配置获得Pilot地址，采用xDS接口从Pilot获取到Listener，Cluster，Route等d动态配置信息
4. Envoy根据获取到的动态配置启动Listener，并根据Listener的配置，结合Route和Cluster对拦截到的流量进行处理

## 流量劫持

sidecar 既要作为服务消费者端的正向代理，又要作为服务提供者端的反向代理, 具体拦截过程如下:

- Pod 所在的network namespace内, 除了envoy发出的流量外, iptables规则会对进入和发出的流量都进行拦截，通过nat redirect重定向到Envoy监听的15001端口.
- envoy 会根据从Pilot拿到的 XDS 规则, 对流量进行转发.
- envoy 的 listener 0.0.0.0:15001 接收进出 Pod 的所有流量，然后将请求移交给对应的virtual listener
- 对于本pod的服务, 有一个http listener `podIP+端口` 接受inbound 流量
- 每个service+非http端口, 监听器配对的 Outbound 非 HTTP 流量
- 每个service+http端口, 有一个http listener: `0.0.0.0+端口` 接受outbound流量

整个拦截转发过程对业务容器是透明的, 业务容器仍然使用 Service 域名和端口进行通信, service 域名仍然会转换为service IP, 但service IP 在sidecar 中会被直接转换为 pod IP, 从容器中出去的流量已经使用了pod IP会直接转发到对应的Pod, 对比传统kubernetes 服务机制, service IP 转换为Pod IP 在node上进行, 由 kube-proxy维护的iptables实现.

## xDS

xDS是一类发现服务的总称，包含LDS，RDS，CDS，EDS以及 SDS。Envoy通过xDS API可以动态获取Listener(监听器)， Route(路由)，Cluster(集群)，Endpoint(集群成员)以 及Secret(证书)配置

Sidecar Envoy 提供了管理接口，缺省为localhost的15000端口，可以获取listener，cluster以及完整的配置数据

```bash
kubectl exec productpage-v1-f8c8fb8-zjbhh -c istio-proxy curl http://127.0.0.1:15000/help
```

可以使用istioctl 查看代理配置:

```yaml
istioctl pc {xDS类型}  {POD_NAME} {过滤条件} {-o json/yaml}
```

比如查看 product 的所有listener:

```
istioctl pc listener productpage-v1-5d9b4c9849-99kff -n istio-test
```

# 控制面

Istiod 提供服务发现、配置和证书管理。

Istiod 将控制流量行为的高级路由规则转换为 Envoy 特定的配置，并在运行时将其传播给 Sidecar。Pilot 提取了特定平台的服务发现机制，并将其综合为一种标准格式，任何符合 [Envoy API](https://www.envoyproxy.io/docs/envoy/latest/api/api) 的 Sidecar 都可以使用。

## 流量管理

相关的CRD有：

- [**Virtualservice**](https://istio.io/docs/reference/config/networking/virtual-service/)：用于定义路由规则，如根据来源或 Header 制定规则，或在不同服务版本之间分拆流量。

- [**DestinationRule**](https://istio.io/docs/reference/config/networking/destination-rule/)：定义目的服务的配置策略以及可路由子集。策略包括断路器、负载均衡以及 TLS 等。

  > DestinationRule的host必须是一个有sidecar的服务，或者通过ServiceEntry加入进来的非网格服务

- [**ServiceEntry**](https://istio.io/docs/reference/config/networking/service-entry/)：可以使用ServiceEntry向Istio中加入附加的服务条目，以使网格内可以向istio 服务网格之外的服务发出请求。

- [**Gateway**](https://istio.io/docs/reference/config/networking/gateway/)：为网格配置网关，以允许一个服务可以被网格外部访问。

- [**EnvoyFilter**](https://istio.io/docs/reference/config/networking/envoy-filter/)：可以为Envoy配置过滤器。由于Envoy已经支持Lua过滤器，因此可以通过EnvoyFilter启用Lua过滤器，动态改变Envoy的过滤链行为。我之前一直在考虑如何才能动态扩展Envoy的能力，EnvoyFilter提供了很灵活的扩展性。

- [**Sidecar**](https://istio.io/docs/reference/config/networking/sidecar/)：缺省情况下，Pilot将会把和Envoy Sidecar所在namespace的所有services的相关配置，包括inbound和outbound listenter, cluster, route等，都下发给Enovy。使用Sidecar可以对Pilot向Envoy Sidcar下发的配置进行更细粒度的调整，例如只向其下发该Sidecar 所在服务需要访问的那些外部服务的相关outbound配置。