Kubernetes-Dashboard是一个SIG开源的免费Dashboard，功能比较单一

# 源码结构

Dashboard是一个前后端分离的结构，所有代码都在`src`目录，里面的`backend`目录就是后端代码，看目录结构

```
.
├── api
├── args
├── auth
├── cert
├── client
├── dashboard.go
├── errors
├── handler
├── integration
├── plugin
├── resource
├── scaling
├── settings
├── sync
├── systembanner
└── validation
```

其中每一个目录都是一个package

分成了manager和handler

- handler用来处理来自客户的http请求,不同的path路由到不同的的handler处理,使用的是go-restful库
- manager负责数据处理

主要的manager如下：

- client：根据前端用户请求从api-server获取相应数据, 每个http request中都会携带用户的authinfo, 用于创建apiserver client, 获取用户所需数据, 系统启动时会初始化一个insecureClient用来做一些用户无关的请求,例如获取k8s集群版本
- auth：包括所有的认证方面的处理,认证其实是交给K8S apiServer负责的,dashboard只是根据用户登录信息生成authInfo对象,加密后作为token携带在浏览器中, 即jwe协议, jwe子包是JWE协议的实现, 其中KeyHolder(rsaKeyHolder concrete class)用来管理jwe用到的密钥对, 将秘钥存放在kubernetes-dashboard-key-holder secrets对象中, 实时在不同dashboard实例间同步
- integration：用来集成显示其他信息,例如监控信息, 每个被集成的对象被称为一个integration, 并有一个integration Id 与之对应, integrationManager其实是交给metricManager来管理integration的, metricManager会为每个integration创建一个对应的MetricClient获取数据,
- settings：是一些基本的设置,包括ClusterName, ItemsPerPage, AutoRefreshTimeInterval 等信息,保存在kubernetes-dashboard-settings 这个config map中,用户可以通过页面来设置,更新这个configMap
- sync：用来监视k8s资源,会定期poll指定的资源,如果资源发生变化会调用用户注册的回调函数,并且负责对资源的CURD操作
- 