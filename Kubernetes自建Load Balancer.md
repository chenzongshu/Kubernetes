# 前言

通常生产环境service都是使用ingress的域名来暴露服务, 但在实际应用中，还是有很多需要以TCP/UDP方式存取或连接服务，这样该如何进行呢?相信有研究过Kubernetes朋友都知道NGINX Ingress也支援了TCP/UDP的反向代理，这表示Ingress也能支持TCP/UDP。但另一个问题来了，如果有两个服务需要用到同一个TCP/UDP Port时，这该怎么办呢?

还有在实际场景中，Ingress 也需要解决下面的一些问题：

1. Ingress 多用于 L7，对于 L4 的支持不多。
2. 所有的流量都会经过 Ingress Controller，需要一个 LB 将 Ingress Controller 暴露出去。

第一个问题，Ingress 也可以用于 L4，但是对于 L4 的应用，Ingress 配置过于复杂，最好的实现就是直接用 LB 暴露出去。第二个问题，测试环境可以用 NodePort 将 Ingress Controller 暴露出去或者直接 hostnetwork，但也不可避免有单点故障和性能瓶颈，也无法很好的使用 Ingress-controller 的 HA 特性。

这个时候Loader Balancer就派上用场

问题是这货只有公有云才提供, 如果我们只是一个自建的PaaS平台怎么玩

现在业界有两种纯软件方案, metallb和Porter, 后者是青云kubesphere开源的一个子模块, 2019.2.21才发布第一个版本,  只有5个版本, 目前是0.1.1, Porter更加适配青云, 网上也没太多资料, 我们主要验证下metallb

metallb的官方网站：https://metallb.universe.tf/

# 安装

执行下面3条命名, 去下载yaml文件, 然后创建一个secret

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml

kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

可以修改`metallb.yaml`的镜像下载策略, 从`Always`修改为`IfNotPresent`, 注意有2处

```
imagePullPolicy: Always   -> imagePullPolicy: IfNotPresent
```

然后可以查看所有Pod是否正常, 注意是在metallb自己的namespace

```
$ kubectl -n metallb-system get po
NAME                          READY   STATUS    RESTARTS   AGE
controller-6bcfdfd677-q9fzp   1/1     Running   0          5m
speaker-8648w                 1/1     Running   0          5m
speaker-8h4gs                 1/1     Running   0          5m
speaker-f9zh4                 1/1     Running   0          5m
speaker-dc134                 1/1     Running   0          5m
speaker-xnkt5                 1/1     Running   0          5m
speaker-zczp5                 1/1     Running   0          5m
speaker-zzn5v                 1/1     Running   0          5m
```

通过configmap配置地址池, 下面配置一个L2的地址池

```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      auto-assign: true
      addresses:
      - 172.22.132.150-172.22.132.200
    # - name: production
    #   auto-assign: false
    #   avoid-buggy-ips: true
    #   addresses:
    #   - 172.22.131.0/24
```

# 测试

创建一个nginx的deployment和service

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - name: http
          containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  type: LoadBalancer
```

可以看到service已经创建

```
$ kubectl get svc
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)          AGE
nginx        LoadBalancer   10.111.63.50   172.22.132.153   80:31827/TCP   59s
```

访问地址可以通

```
$ curl 172.22.132.153:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
...
```

# 原理

## 架构

MetalLB是基于标准[路由协定](https://en.wikipedia.org/wiki/Routing_protocol)实作的Kubernetes集群负载平衡方案。主要以两个元件实现裸机负载平衡功能，分别为:

- **Controller** :是MetalLB控制器，主要负责分配IP给Kubernetes Service资源。该组件会监听Kubernetes Service资源的事件，一但集群有LoadBalancer类型的Service被新增时，就依据内容从一个IP位址池分配负载平衡IP给Service使用。以deployment部署
- **Speaker** :利用网路协定(L2: ARP/NDP, L3: BGP)告知负载平衡IP的目的位址在何处，并且如何路由。Speaker是一个被安装在所有节点上的Controller，以daemonset部署, 而这些集群上的Speaker只会有一个负责处理事情。

### L2模式

在任何以太网环境均可使用该模式。当在第二层工作时，将由一台机器获得IP地址（即服务的所有权）。MetalLB使用标准的第地址发现协议（对于IPv4是ARP，对于IPv6是NDP）宣告IP地址，是其在本地网路中可达。从LAN的角度来看，仅仅是某台机器多配置了一个IP地址。

L2模式下，服务的入口流量全部经由单个节点，然后该节点的kube-proxy会把流量再转发给服务的Pods。也就是说，该模式下MetalLB并没有真正提供负载均衡器。尽管如此，MetalLB提供了故障转移功能，如果持有IP的节点出现故障，则默认10秒后即发生故障转移，IP被分配给其它健康的节点。

这种模式好处在于简单，且不需要外部硬件支持或配置。但受限于L2 网络协定。

### L3 BGP模式

当在第三层工作时，集群中所有机器都和你控制的最接近的路由器建立BGP会话，此会话让路由器能学习到如何转发针对K8S服务IP的数据报。

通过使用BGP，可以实现真正的跨多节点负载均衡（需要路由器支持multipath），还可以基于BGP的策略机制实现细粒度的流量控制。

具体的负载均衡行为和路由器有关，可保证的共同行为是：每个连接（TCP或UDP会话）的数据报总是路由到同一个节点上，这很重要，因为：

1. 将单个连接的数据报路由给多个不同节点，会导致数据报的reordering，并大大影像性能

2. K8S节点会在转发流量给Pod时可能导致连接失败，因为多个节点可能将同一连接的数据报发给不同Pod

BGP模式的缺点：

1. 不能优雅处理故障转移，当持有服务的节点宕掉后，所有活动连接的客户端将收到Connection reset by peer

2. BGP路由器对数据报的源IP、目的IP、协议类型进行简单的哈希，并依据哈希值决定发给哪个K8S节点。问题是K8S节点集是不稳定的，一旦（参与BGP）的节点宕掉，很大部分的活动连接都会因为rehash而坏掉

当数据包到达节点之后, 由kube-proxy负责最后一跳, 把数据包导到Pod中

这种模式适合生产环境，但需要更多的外部硬件与配置来达成。但要确保和BGP 的CNI 不会冲突

需要提前知道得信息

- MetalLB应该连接的路由器IP地址，
- 路由器的AS号，
- MetalLB应该使用的AS号，
- 以CIDR前缀表示的IP地址范围。

示例配置

```
apiVersion: v1 
kind: ConfigMap 
metadata: 
  namespace: metallb-system 
  name: config 
data: 
  config: | 
  peers: 
  - peer-address: 10.0.0.1 
  peer-asn: 64501 
  my-asn: 64500 
  address-pools: 
  - name: default 
  protocol: bgp 
  addresses: - 192.168.10.0/24
```



## 已知缺陷

与fanneld目前没有已知缺陷, 但是和Calico不能同时使用BGP方式

这个问题是由BGP协议本身导致的 —— BGP协议只每对节点之间有一个会话，这意味着当Calico和BGP路由器建立会话后，MetalLB就无法创建会话了。

由于目前Calico没有暴露扩展点，MetalLB没有办法与之集成。

当然也有一些变通方式,  可以让Calico、MetalLB和不同的BGP路由进行配对, 这里不展开描述, 见metallb官网:  https://metallb.universe.tf/configuration/calico/









