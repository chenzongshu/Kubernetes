# 前言

Kubernetes服务之间的调用, 同一个namespace之间可以通过服务名来调用, 不同namespace可以通过服务域名来调用, 默认格式是  `<serviceName>.<namespace>.svc.cluster.local`

Kubernetes 认为，内部域名，最长为5, 所以默认设置了`ndots:5`

> ndots:5，表示：如果查询的域名包含的点“.”，不到5个，那么进行DNS查找，将使用非完全限定名称（或者叫绝对域名），如果你查询的域名包含点数大于等于5，那么DNS查询，默认会使用绝对域名进行查询

Kubernetes可以单独对Pod调整`ndots`值, 具体段如下

```
spec:
  containers:
    - name: test
      image: nginx
  dnsConfig:
    options:
      - name: ndots
        value: "1"
```

# CoreDNS

整个 CoreDNS 服务都建立在一个使用 Go 编写的 HTTP/2 Web 服务器 [Caddy](https://github.com/mholt/caddy), 大多数功能都由插件提供

## corefile基本配置

另一个 CoreDNS 的特点就是它能够通过简单易懂的 DSL 定义 DNS 服务，在 Corefile 中就可以组合多个插件对外提供服务, corefile由configmap提供, 在kube-system命名空间下面

```go
[root@node000006 ~]# kubectl -n kube-system get cm coredns -o yaml
apiVersion: v1
data:
  Corefile: |
    .:53 {
        errors  #错误记录到标准输出
        health {
            lameduck 5s
        }
      
        #在端口 8181 上提供的一个 HTTP 末端，当所有能够 表达自身就绪的插件都已就绪时，在此末端返回 200 OK
        ready  
      
        #CoreDNS 将基于 Kubernetes 的服务和 Pod 的 IP 答复 DNS 查询
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        rewrite name czs.com nginx-deployment.default.svc.cluster.local
        prometheus :9153
      
        #不在 Kubernetes 集群域内的任何查询都将转发到 预定义的解析器
        forward . /etc/resolv.conf {
          prefer_udp
        }
        cache 30  #前端缓存
        loop #检测到简单的转发环，如果发现死循环，则中止 CoreDNS 进程
        reload #允许自动重新加载已更改的 Corefile
        loadbalance #这是一个轮转式 DNS 负载均衡器， 它在应答中随机分配 A、AAAA 和 MX 记录的顺序
    }
```

## 存根域和上游服务器

CoreDNS 能够使用 [forward 插件](https://coredns.io/plugins/forward/)配置存根域和上游域名服务器.

比如现在需要将`.consul.local`后缀的指向一个单独域名服务器, 可以在corefile写入

```
consul.local:53 {
        errors
        cache 30
        forward . 10.150.0.1
    }
```

## 自定义域名

有时候, 我们需要自定义域名, 比如我给集群内部的Nginx的service定义一个 czs.com 的域名, 在集群内部给其他服务使用, 为什么要这么做而不使用service的默认域名来使用呢, 因为这样我们就可以在集群的内外部使用同样的域名来访问这个Nginx服务.

**注意不要在istio注入的namespace使用该方法**

### hosts

配置corefile里面的hosts，解析域名到service的clusterIP

```go
apiVersion: v1
data:
  Corefile: |2-
    .:53 {
        errors
        health
        kubernetes cluster.local. in-addr.arpa ip6.arpa {
            pods insecure
            upstream
            fallthrough in-addr.arpa ip6.arpa
        }
        hosts {  
            192.168.1.6     harbor.oa.com  // 配置外部域名的解析
            192.168.1.8     es.oa.com
            fallthrough
        }
        prometheus :9153
        forward . /etc/resolv.conf
        ......
```

### Rewrite

配置corefile的rewrite可以把外部域名解析到服务默认的内部域名上

```go
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
        //把外部域名chen.czs.com指向service的内部域名nginx-deployment.default.svc.cluster.local
        rewrite name chen.czs.com nginx-deployment.default.svc.cluster.local
```

## CoreDNS延时

由于linux内核缺陷，导致kubernetes集群在解析DNS的时候conntrack冲突，间歇性出现5秒延迟（内核版本在5.0和4.19修复了2/3）

主要是由于 UDP 是无连接的，内核 netfilter 模块在处理同一个 socket 上的并发 UDP 包时就可能会有竞争问题。解决方法也有多种。

1. 禁止并发 DNS 查询，比如在 Pod 配置中开启 `single-request-reopen` 选项强制 A 查询和 AAAA 查询使用相同的 socket：

```text
dnsConfig:
  options:
    - name: single-request-reopen
```

2. 禁用 IPv6 从而避免 AAAA 查询，比如可以给 Grub 配置 `ipv6.disable=1` 来禁止 ipv6（需要重启节点才可以生效）。

3. 使用 TCP 协议，比如在 Pod 配置中开启 `use-vc` 选项强制 DNS 查询使用 TCP 协议：

```text
dnsConfig:
  options:
    - name: single-request-reopen
    - name: ndots
      value: "5"
    - name: use-vc
```

4. 使用nodelocaldns，本地缓存

# NodeLocalDNS

原理： 本地 DNS 缓存以 DaemonSet 方式在每个节点部署一个使用 hostNetwork 的 Pod，创建一个网卡绑上本地 DNS 的 IP，本机的 Pod 的 DNS 请求路由到本地 DNS，然后取缓存或者继续使用 TCP 请求上游集群 DNS 解析。

## NodeLocalDNS作用

- 使用当前的 DNS 体系结构，如果没有本地 kube-dns/CoreDNS 实例，则具有最高 DNS QPS 的 Pod 可能必须延伸到另一个节点。 在这种脚本下，拥有本地缓存将有助于改善延迟。

- 跳过 iptables DNAT 和连接跟踪将有助于减少 [conntrack 竞争](https://github.com/kubernetes/kubernetes/issues/56903)并避免 UDP DNS 条目填满 conntrack 表。

- 从本地缓存代理到 kube-dns 服务的连接可以升级到 TCP 。 TCP conntrack 条目将在连接关闭时被删除，相反 UDP 条目必须超时([默认](https://www.kernel.org/doc/Documentation/networking/nf_conntrack-sysctl.txt) `nf_conntrack_udp_timeout` 是 30 秒)

- 将 DNS 查询从 UDP 升级到 TCP 将减少归因于丢弃的 UDP 数据包和 DNS 超时的尾部等待时间，通常长达 30 秒（3 次重试+ 10 秒超时）。

- 在节点级别对 dns 请求的度量和可见性。

- 可以重新启用负缓存，从而减少对 coredns 服务的查询数量。

## 安装

下载yaml部署文件

```
wget https://github.com/kubernetes/kubernetes/raw/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml
```

该资源清单文件中包含几个变量，其中：

- `__PILLAR__DNS__SERVER__` ：表示 `kube-dns` 这个 Service 的 ClusterIP，可以通过命令 `kubectl get svc -n kube-system | grep kube-dns | awk '{ print $3 }'` 获取
- `__PILLAR__LOCAL__DNS__`：表示 DNSCache 本地的 IP，默认为 169.254.20.10
- `__PILLAR__DNS__DOMAIN__`：表示集群域，默认就是 `cluster.local`

然后使用命令安装, 注意镜像需要提前下载

```
$ sed 's/__PILLAR__DNS__SERVER__/10.96.0.10/g
s/__PILLAR__LOCAL__DNS__/169.254.20.10/g
s/__PILLAR__DNS__DOMAIN__/cluster.local/g
s/__PILLAR__CLUSTER__DNS__/10.96.0.10/g
s/__PILLAR__UPSTREAM__SERVERS__/\/etc\/resolv.conf/g' nodelocaldns.yaml |
kubectl apply -f -
```

> 需要注意的是这里使用 DaemonSet 部署 node-local-dns 使用了 `hostNetwork=true`，会占用宿主机的 8080 端口，所以需要保证该端口未被占用。

查看对应Pod是否已经运行正常

```
[root@k8s-m1 ~]# kubectl -n kube-system get po
NAME                             READY   STATUS    RESTARTS   AGE
node-local-dns-jq9xb             1/1     Running   0          62s
node-local-dns-zctch             1/1     Running   0          62s
```

最后修改kubelet参数中dns的指向， 让其指向 DNSCache 本地的 IP169.254.20.10，集群如果是使用kubeadm安装的，修改`/var/lib/kubelet/config.yaml`,  然后重启kubelet

```
[root@k8s-w1 ~]# vim /var/lib/kubelet/config.yaml

clusterDNS:
- 169.254.20.10

[root@k8s-w1 ~]# systemctl restart kubelet
```

## 验证

启动一个Pod，可以看到dns server配置已经变了

```
[root@centos-localdns-6bfcc46487-qkcgg /]# cat /etc/resolv.conf
nameserver 169.254.20.10
search test.svc.cluster.local svc.cluster.local cluster.local localdomain
options ndots:5
```

注意新建的Pod可以，但是老的Pod需要重建

## NodeLocalDNS导致CoreDNS Rewrite失效

在使用NodeLocalDNS的时候，配置CoreDNS的Rewrite会失效，原因是nodelocaldns配置时候，在配置里面，实际是指向本地的 `/etc/resolv.conf`，所以集群内部域名会转发给 `coredns`, 而非集群内部域名会转发给 `/etc/resolv.conf`, 根本就不会转发给 `coredns`, 所以 `coredns` 里面配置的 `hosts` 自然不会生效

```go
.:53 {
    errors
    cache 30
    reload
    loop
    bind 169.254.20.10 10.96.0.10
    forward . /etc/resolv.conf
    prometheus :9253
    }
```

解决方法：

方法1：修改NodeLocalDNS 位于 kube-system 的configmap如下：

```go
    .:53 {
        errors
        cache 30
        reload
        loop
        bind 169.254.25.10
        forward . 10.233.0.3  //指向CoreDNS的Service ClusterIP
        prometheus :9253
    }
```

方法2：初始化的时候就指定`__PILLAR__UPSTREAM__SERVERS__`为CoreDNS的Service ClusterIP

```
$ sed 's/__PILLAR__DNS__SERVER__/10.96.0.10/g
s/__PILLAR__LOCAL__DNS__/169.254.20.10/g
s/__PILLAR__DNS__DOMAIN__/cluster.local/g
s/__PILLAR__CLUSTER__DNS__/10.96.0.10/g
s/__PILLAR__UPSTREAM__SERVERS__/10.96.0.10/g' nodelocaldns.yaml |
kubectl apply -f -
```

