前面说的各种资源类型都是Kubernetes中服务和应用模型的抽象, 但是我们要怎么访问这些服务呢?

考虑两个因素: 

1. Pod 的 IP 不是固定的，
2. 一 组 Pod 实例之间总会有负载均衡的需求

Kubernetes提出了通过service对Pod对外服务来进行一层抽象

那么service的本质是什么? 内部外部怎么访问? 原理是什么? 流量怎么走? 

# Service

**所谓 Service，其实就是 Kubernetes 为 Pod 分配的、固定的、基于 iptables（或者 IPVS）的访问入口。而这些访问入口代理的 Pod 信息，则来自于 Etcd，由 kube-proxy 通过控制循环来维护。**

> endpoint: 就是service里被selector 选中的 Pod, 注意的是只有处于 Running 状态，且 readinessProbe 检查通过的 Pod，才会出现在 Service 的 Endpoints 列表里。并且，当某一个 Pod 出现问题时，Kubernetes 会自动把它从 Service 里摘除掉。

service机制只能在集群内提供服务

## 原理

### iptables模式

一旦创建service, kube-proxy 就可以通过 Service 的 Informer 感知到这样一个 Service 对象的添加。而作为对这个事件的响应，它就会在宿主机上创建这样一条 iptables 规则（你可以通过 iptables-save 看到它），如下所示：

```
-A KUBE-SERVICES -d 10.0.1.175/32 -p tcp -m comment --comment "default/hostnames: cluster IP" -m tcp --dport 80 -j KUBE-SVC-NWV5X2332I4OT4T3
```

可以看到，这条 iptables 规则的含义是：凡是目的地址是 10.0.1.175、目的端口是 80 的 IP 包，都应该跳转到另外一条名叫 KUBE-SVC-NWV5X2332I4OT4T3的 iptables 链进行处理。

这个10.0.1.175, 其实就是Service的VIP, 因为VIP只是iptables中一条规则的配置, 并没有任何网络设备, 所以ping不通

我们跳转到的 KUBE-SVC-NWV5X2332I4OT4T3 规则，又有什么作用呢？

实际上，它是一组规则的集合，如下所示：

```
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-WNBA2IHDGP2BOBGZ
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-X3P2623AGDH6CDF3
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -j KUBE-SEP-57KPRZ3JQVENLNBR
```

可以看到，这一组规则，实际上是一组随机模式（–mode random）的 iptables 链。

而随机转发的目的地，分别是 KUBE-SEP-WNBA2IHDGP2BOBGZ、KUBE-SEP-X3P2623AGDH6CDF3 和 KUBE-SEP-57KPRZ3JQVENLNBR。

而这三条链指向的最终目的地，其实就是这个 Service 代理的三个 Pod。所以这一组规则，就是 Service 实现负载均衡的位置。

需要注意的是，iptables 规则的匹配是从上到下逐条进行的，所以为了保证上述三条规则每条被选中的概率都相同，我们应该将它们的 probability 字段的值分别设置为 1/3（0.333…）、1/2 和 1。

这么设置的原理很简单：第一条规则被选中的概率就是 1/3；而如果第一条规则没有被选中，那么这时候就只剩下两条规则了，所以第二条规则的 probability 就必须设置为 1/2；类似地，最后一条就必须设置为 1。

通过查看上述三条链的明细，我们就很容易理解 Service 进行转发的具体原理了，如下所示：

```
-A KUBE-SEP-57KPRZ3JQVENLNBR -s 10.244.3.6/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-57KPRZ3JQVENLNBR -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.3.6:9376

-A KUBE-SEP-WNBA2IHDGP2BOBGZ -s 10.244.1.7/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-WNBA2IHDGP2BOBGZ -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.1.7:9376

-A KUBE-SEP-X3P2623AGDH6CDF3 -s 10.244.2.3/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-X3P2623AGDH6CDF3 -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.2.3:9376
```

可以看到，这三条链，其实是三条 DNAT 规则。DNAT 规则的作用，就是在 PREROUTING 检查点之前，也就是在路由之前，将流入 IP 包的目的地址和端口，改成–to-destination 所指定的新的目的地址和端口。可以看到，这个目的地址和端口，正是被代理 Pod 的 IP 地址和端口。

这样，访问 Service VIP 的 IP 包经过上述 iptables 处理之后，就已经变成了访问具体某一个后端 Pod 的 IP 包了。不难理解，这些 Endpoints 对应的 iptables 规则，正是 kube-proxy 通过监听 Pod 的变化事件，在宿主机上生成并维护的。



### ipvs模式

IPVS 模式的工作原理，其实跟 iptables 模式类似。当我们创建了前面的 Service 之后，kube-proxy 首先会在宿主机上创建一个虚拟网卡（叫作：kube-ipvs0），并为它分配 Service VIP 作为 IP 地址，如下所示：

```
# ip addr
...
73：kube-ipvs0：<BROADCAST,NOARP>  mtu 1500 qdisc noop state DOWN qlen 1000
link/ether  1a:ce:f5:5f:c1:4d brd ff:ff:ff:ff:ff:ff
inet 10.0.1.175/32  scope global kube-ipvs0
valid_lft forever  preferred_lft forever
```

而接下来，kube-proxy 就会通过 Linux 的 IPVS 模块，为这个 IP 地址设置三个 IPVS 虚拟主机，并设置这三个虚拟主机之间使用轮询模式 (rr) 来作为负载均衡策略。我们可以通过 ipvsadm 查看到这个设置，如下所示：

```
# ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
->  RemoteAddress:Port           Forward  Weight ActiveConn InActConn     
TCP  10.102.128.4:80 rr
->  10.244.3.6:9376    Masq    1       0          0         
->  10.244.1.7:9376    Masq    1       0          0
->  10.244.2.3:9376    Masq    1       0          0
```

可以看到，这三个 IPVS 虚拟主机的 IP 地址和端口，对应的正是三个被代理的 Pod。

这时候，任何发往 10.102.128.4:80 的请求，就都会被 IPVS 模块转发到某一个后端 Pod 上了。

而相比于 iptables，IPVS 在内核中的实现其实也是基于 Netfilter 的 NAT 模式，所以在转发这一层上，理论上 IPVS 并没有显著的性能提升。但是，IPVS 并不需要在宿主机上为每个 Pod 设置 iptables 规则，而是把对这些“规则”的处理放到了内核态，从而极大地降低了维护这些规则的代价

不过需要注意的是，IPVS 模块只负责上述的负载均衡和代理功能。而一个完整的 Service 流程正常工作所需要的包过滤、SNAT 等操作，还是要靠 iptables 来实现。只不过，这些辅助性的 iptables 规则数量有限，也不会随着 Pod 数量的增加而增加。

### nodePort与SNAT

nodePort原理几乎和ClusterIP一样, 就是在每台宿主机上生成这样一条 iptables 规则：

```
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/my-nginx: nodePort" -m tcp --dport 8080 -j KUBE-SVC-67RL4FN6JRUPOJYM
```

需要注意的是，在 NodePort 方式下，Kubernetes 会在 IP 包离开宿主机发往目的 Pod 时，对这个 IP 包做一次 SNAT 操作，如下所示：

```
-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE
```

可以看到，这条规则设置在 POSTROUTING 检查点，也就是说，它给即将离开这台主机的 IP 包，进行了一次 SNAT 操作，将这个 IP 包的源地址替换成了这台宿主机上的 CNI 网桥地址，或者宿主机本身的 IP 地址（如果 CNI 网桥不存在的话）。

当然，这个 SNAT 操作只需要对 Service 转发出来的 IP 包进行（否则普通的 IP 包就被影响了）。而 iptables 做这个判断的依据，就是查看该 IP 包是否有一个“0x4000”的“标志”。

可是，**为什么一定要对流出的包做 SNAT操作呢**

道理很简单, 当Client通过nodePort访问Pod时候, 首先放到到Node1, 可能实际得Pod并不在上面, 还需要把数据包转发到Node2上, Node2处理完数据会根据数据包中的源地址回包, 即Node2直接给Client回包, 对Client来说, 明明给Node1发的包, 回包确是Node2, 很可能报错.

所以需要SNAT, 在离开Node1的时候把源IP改为Node1的CNI网桥地址或者Node1的地址, 这样Pod处理完后将回包到Node1, 再由Node1回包到Client

也可以将 Service 的 spec.externalTrafficPolicy 字段设置为 local, 关闭SNAT, 适用于那些Pod需要看到真正请求源地址的场景

# Ingress

**所谓 Ingress，就是 Service 的“Service”, 即这种全局的、为了代理不同后端 Service 而设置的负载均衡服务**

看看常见的Ingress资源示例:

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-wildcard-host
spec:
  rules:
  - host: "foo.bar.com"
    http:
      paths:
      - pathType: Prefix
        path: "/bar"
        backend:
          service:
            name: service1
            port:
              number: 80
  - host: "*.foo.com"
    http:
      paths:
      - pathType: Prefix
        path: "/foo"
        backend:
          service:
            name: service2
            port:
              number: 80
```

可以看到Ingress资源就是定义了把host来的请求转发到对应的service上面去, 一个 Ingress 对象的主要内容，实际上就是一个“反向代理”服务（比如：Nginx）的配置文件 的描述。

这个转发操作是怎么做到的呢? 以常见的Nginx Ingress为例看看, 其实挺简单

Nginx Ingress Controller就是一个Kubernetes Controller + Nginx + Lua的组合, Controller负责监控资源变化, Nginx负责转发请求, Controller监控到资源变化之后, 由Lua动态来更改Nginx的配置.

所有的Ingress Controller原理都差不多

**需要注意的是外部访问的时候需要访问的是Ingress Controller的Service的地址**

# DNS

在集群中定义的每个 Service（包括 DNS 服务器自身）都会被指派一个 DNS 名称。目前Kubernetes默认的DNS服务器是CoreDNS

那怎么数据流是怎么样的呢?

Pod内的resolv.conf文件指定了DNS相关信息，根据域名去找服务的IP

```
nameserver 169.254.25.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

- 其中169.254.25.10就是CoreDNS Service的ClusterIP.

- 注意配置的`search`域和`ndots`值，ndots是什么意思呢？ 简单说，如果你的域名请求参数中，`点的个数`比配置的`ndots`小，则会按照配置的`search`内容，依次添加相应的后缀直到获取到域名解析后的地址。如果通过添加了search之后还是找不到域名，则会按照一开始请求的域名进行解析。

  >  举个例子，你在Pod访问一个服务serviceA，使用的服务名serviceA，ndots数量为0，小于5，所以实际上自动给你加上search域的第一个域名变成 serviceA.default.svc.cluster.local，如果没找到继续往下，直到search域里面的找完，然后再使用本来的域名serviceA来解析
  
  所以实际中常常调整ndots的值，减少为2，来减少DNS查询次数，优化效率，当然Kubernetes设置ndots默认值为5也是有它的道理的，后面我们来摆一摆

如果是外部域名，又是怎么来解析出来的呢？

首先要明确，外部域名也需要按照resolve.conf的搜索域来走一遍DNS请求，然后最后CoreDNS里面还是找不到该域名对应的解析，这个时候，CoreDNS就会根据自身的配置，把请求转向它自己配置的forward的DNS服务器中，如果CoreDNS是容器化部署的，这个配置在configmap中：

```yaml
[root@node000006 ~]# kubectl -n kube-system get cm coredns -o yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf {
          prefer_udp
        }
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
metadata:
```

forward插件的的默认配置是转发coredns节点的resolv.conf，即使用了宿主机的DNS配置

Kubernetes 默认配置下 ndots 的值是 5，其原因官方在 [issue 33554](https://github.com/kubernetes/kubernetes/issues/33554#issuecomment-266251056) 中做了解释：

- 有些人需要配置集群 `$zone` 的后缀以支持多集群，所以一个完整的 Service 的域名为 `$service.$namespace.svc.$zone`，为了不在代码中配置完整的域名，就需要设置 ndots 和 search。
- 同 namespace 下的 Service 的请求是最常用的，因此需要解析 `$service`，此时需 ndots >= 1，且 search 列表中第一个应为 `$namespace.D.$zone`。
- 跨 namespace 的 Service 也经常会被请求，因此需要解析 `$service.$namespace`，此时需 ndots >= 2，且 search 列表中第二个应为 `svc.$zone`。
- 为了解析 `$service.$namespace.svc`，此时需 ndots >= 3，且 search 列表包含 `$zone`。
- 在 Kubernetes 1.4 之前，StatefulSet 为PetSet，当每个 Pet 被创建时，它会获得一个匹配的 DNS 子域，域格式为 `$petname.$service.$namespace.svc.$zone`，此时需 ndots >= 4。
- Kubernetes 还支持 SRV 记录，因此 `_$port.$proto.$service.$namespace.svc.$zone` 需要可以解析，此时 ndots = 5。

这就是为什么 ndots 为 5。总结来说是为了支持更复杂的 Pod 内的域名解析，但通常情况下只会用到同 namespace 下的 Service（形如 `service-b`）和跨 namespace 下 Service（形如 `service-c.sre`）的访问，因此 ndots 的默认值设为 2 便可满足业务需求并避免大量无效的解析请求。

