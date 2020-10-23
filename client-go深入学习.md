# 源码结构

`client-go`已经集成到 `kubernetes`源码当中， 目录是`staging\src\k8s.io\client-go`

```
# tree . -L 1
.
......
├── discovery
├── dynamic
├── examples
├── Godeps
├── informers
├── INSTALL.md
├── kubernetes
├── kubernetes_test
├── listers
├── metadata
├── OWNERS
├── pkg
├── plugin
├── rest
├── restmapper
├── scale
├── SECURITY_CONTACTS
├── testing
├── third_party
├── tools
├── transport
└── util
```

| 源码目录   | 说明                                                         |
| ---------- | ------------------------------------------------------------ |
| discovery  | 提供discovery client客户端                                   |
| dynamic    | 提供dynamic client客户端                                     |
| informers  | 每种kubernetes资源的informer实现                             |
| kubernetes | 提供clientsetclient客户端                                    |
| listers    | 为每种kubernetes资源提供Lister功能, 该功能对Get和List请求提供只读的缓存数据 |
| plugin     | 提供openstack, GCP, Azure等云服务商的授权插件                |
| rest       | 提供RESTClient客户端, 对APIServer执行Restful 操作            |
| scale      | 提供ScaleClient客户端, 用于扩缩容Deployment, RS, RC等资源对象 |
| tools      | 提供常用工具, 例如SharedInformer, Reflector, DealtFIFO, Indexers. 提供Client查询和缓存机制, 以减少向APIServer发起的请求 |
| transport  | 提供安全的TCP连接, 支持HTTP Stream, 某些操作需要在客户端和容器之间传输二进制流, 例如exec, attach等操作. 该功能由内部的spdy包提供 |
| util       | 提供常用方法, 例如WorkQueue工作队列, Certificate证书管理等   |

