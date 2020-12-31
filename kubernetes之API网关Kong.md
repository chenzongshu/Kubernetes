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

**默认chart是DB-less模式**





