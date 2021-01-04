# Kong

kong是一个高效可扩展的API网关，与kubernetes集成也很方便，并且集成了动态路由，熔断，健康检查，日志，鉴权，SSL，认证，限流等功能。

一句话： Kong = OpenResty + Nginx + Lua

Kong有原生的Ingress Controller，可以方便的集成到kubernetes集群中。

Kong的一个重要功能是它只能在一个环境中运行（而不是支持跨命名空间）。这是一个颇具争议的话题：有些人认为它是一个缺点（必须为每个环境生成实例），而另一些人则认为它是一项特殊功能（较高的隔离度，因此一个控制器的故障影响仅限于其环境） 。

默认监听下面端口

- 8000，监听来自客户端的HTTP流量，转发到你的upstream服务上。

- 8443，监听HTTPS的流量，功能跟8000一样。可以通过配置文件禁止。

- 8001，Kong的HTTP监听的api管理接口。

- 8444，Kong的HTTPS监听的API管理接口。

这些端口可以在`/etc/kong/kong.conf`中修改，:8000 和 :8443 默认绑定0.0.0.0；:8001 和 :8444 默认绑定 127.0.0.1，

这些端口其实是两类，Proxy端口(`8000` or `8443`)用于代理后端服务，Admin端口(`8001` or `8444`)用于管理Kong配置，对Kong配置进行CRUD操作([Konga](https://github.com/pantsel/konga)就是利用Admin API实现的GUI)

## 使用方式

kong部署到kubernetes集群里面，有2种使用方式，下面简单介绍下

### Kong原生

使用Admin API配置Kong来访问Kubernetes service，举例如下：

```json
$ curl -X POST http://194.28.255.207:8001/services/nginx/plugins \
--data "name=rate-limiting" \
--data "config.second=50" \
| python -m json.tool
{
    "config": {
        "day": null,
        "fault_tolerant": true,
        "hide_client_headers": false,
        "hour": null,
        "limit_by": "consumer",
        "minute": null,
        "month": null,
        "policy": "cluster",
        "redis_database": 0,
        "redis_host": null,
        "redis_password": null,
        "redis_port": 6379,
        "redis_timeout": 2000,
        "second": 50,
        "year": null
    },
    "consumer": null,
    "created_at": 1580567002,
    "enabled": true,
    "id": "ce629c6f-046a-45fa-bb0a-2e6aaea70a83",
    "name": "rate-limiting",
    "protocols": [
        "grpc",
        "grpcs",
        "http",
        "https"
    ],
    "route": null,
    "run_on": "first",
    "service": {
        "id": "14100336-f5d2-48ef-a720-d341afceb466"
    },
    "tags": null
}
```

### Ingress Controller

部署Ingress Controller配合4种CRD，原则上可以完成Kong所涵盖的所有特性

Kong-ingress-controller给出了四种CRDs对功能进行扩展，如下：

- [KongIngress](https://github.com/Kong/kubernetes-ingress-controller/blob/master/docs/concepts/custom-resources.md#kongingress)
- [KongPlugin](https://github.com/Kong/kubernetes-ingress-controller/blob/master/docs/concepts/custom-resources.md#kongplugin)
- [KongConsumer](https://github.com/Kong/kubernetes-ingress-controller/blob/master/docs/concepts/custom-resources.md#kongconsumer)
- [KongCredential (Deprecated)](https://github.com/Kong/kubernetes-ingress-controller/blob/master/docs/concepts/custom-resources.md#kongcredential-deprecated)

## 配置模式

kong有两种配置模式，DB mode和DB-less mode， 应该很好理解，DB mode会把配置保存到数据库中，一般是PostgreSQL or Cassandra，DB-less 所有配置存放于一个配置文件中(YAML or JSON格式）

但是需要注意的是DB-less mode更新规则必须全量更新，重置整个配置文件，无法做到局部更新(调用Kong Admin API/config)，而且兼容性较差，无法支持所有插件，比如Konga就不支持

# Ingress控制器

为了让 Ingress 资源工作，集群必须有一个正在运行的 Ingress 控制器。但是Ingress Controller并不会直接作用于Ingress规则，Ingress Controller是用来修改Ingress规则的。举个例子，Ingress-Nginx，Ingress Controller做的事情就是利用lua把Ingress规则写成Nginx的配置，并加载。

以在集群中部署任意数量的 ingress 控制器。 创建 ingress 时，应该使用适当的 `ingress.class` 注解每个 Ingress 以表明在集群中如果有多个 Ingress 控制器时，应该使用哪个 Ingress 控制器。

# Kong部署

目前最新的版本是2.2.x， 官网地址是： https://docs.konghq.com/2.2.x/kong-for-kubernetes/install/

可以通过yaml文件，也可以通过helm包来部署，从官网文档来看，生产最好使用chart包来部署。

从官方默认的仓库下载chart，然后解压，修改values.yaml文件

```
helm repo add kong https://charts.konghq.com
helm pull kong/kong
tar -zxvf kong-1.12.0.tgz
```

修改values.yaml文件之前，看看里面参数的含义，官方chart 的github地址是：https://github.com/Kong/charts/blob/main/charts/kong/README.md

**默认chart是DB-less模式，安装到了default空间**

生产上使用DB模式，安装到自己的空间，PG挂载一个PV，使用LB来暴露Service。

## values.yaml

### 解析

- `Kong Enterprise`是kong的企业版，默认是false，安装需要单独打开
- `Kong Manager`是基于视觉浏览器的工具，用于监视和管理Kong Enterprise，企业版才有

### 修改

```yaml
ingressController  
  installCRDs: false   # helm3 要把这个改为false
```

## 默认安装

在阿里云ACK上，使用LoadBalancer方式暴露负载均衡，其他参数默认

```
[root@node000006 ~]# kubectl create ns kong
namespace/kong created
[root@node000006 ~]#
[root@node000006 ~]# helm install -n kong kong kong
NAME: kong
LAST DEPLOYED: Mon Jan  4 11:16:05 2021
NAMESPACE: kong
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
To connect to Kong, please execute the following commands:

HOST=$(kubectl get svc --namespace kong kong-kong-proxy -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
PORT=$(kubectl get svc --namespace kong kong-kong-proxy -o jsonpath='{.spec.ports[0].port}')
export PROXY_IP=${HOST}:${PORT}
curl $PROXY_IP

Once installed, please follow along the getting started guide to start using
Kong: https://bit.ly/k4k8s-get-started
```

使用上面提示的方式设置临时变量，可以看到LB的访问方式为

```
[hll_root@ALY-01 ~]$ echo $PROXY_IP
119.23.250.137:80
```

在安装完成但是没有任何Ingress规格和服务情况下，使用该方式访问，会得到默认的404

```
[hll_root@ALY-01 ~]$ curl -i $PROXY_IP
HTTP/1.1 404 Not Found
Date: Mon, 04 Jan 2021 06:17:38 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Content-Length: 48
X-Kong-Response-Latency: 0
Server: kong/2.2.1
{"message":"no Route matched with those values"}
```

## 验证

创建一个test  namespace

```
[hll_root@ALY-01 ~]$ kubectl create ns test
namespace/test created
```

然后创建一个nginx对应的deployment，service，Ingress

```
[root@ALY-HN1-ACK-Agent-STG-01 test]# cat ng-test.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: test
  labels:
    app: nginx
spec:
  replicas: 1
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
        image: registry-vpc.cn-shenzhen.aliyuncs.com/hll_app/nginx:1.19
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: test
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: nginx
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-czs
  namespace: test
  annotations:
    kubernetes.io/ingress.class: kong
spec:
  rules:
  - host: "ng.czs.com"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          serviceName: nginx
          servicePort: 80
```

`kubectl apply -f ng-test.yaml`启动实例

然后在集群之外的地方配置域名解析，即把上面定义的域名ng.czs.com指向kong Ingress SLB的地址

```
vim /etc/hosts

xxx.xxx.xxx.xxx ng.czs.com
```

通过curl访问域名ng.czs.com

```
[root@ALY-01 test]# curl ng.czs.com
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

