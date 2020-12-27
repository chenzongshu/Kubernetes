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

### hosts



### Rewrite



# NodeLocalDNS



