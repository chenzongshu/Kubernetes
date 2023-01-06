# 限流

使用`client-go`写一个程序去读取每个pod的实际CPU和内存的时候，发现日志中有大量如下错误：

```go
I1110 10:06:18.858448 1 request.go:597] Waited for 191.236403ms due to client-side throttling, not priority and fairness, request: GET:https://xxx.xxx.xxx.xxx:443/apis/metrics.k8s.io/v1beta1/namespaces/kube-system/pods/coredns-5d579d874f-mg9cc
```

看了APIServer，没有限流，然后研究了下，发现是Client-go的限流算法在起作用

# client-go限流算法

## client-go源码

`vendor/k8s.io/client-go/rest/config.go`
在创建`rest.Config`的时候，里面有对应的变量和结构体

```go
// QPS indicates the maximum QPS to the master from this client.
// If it's zero, the created RESTClient will use DefaultQPS: 5
QPS float32
// Maximum burst for throttle.
// If it's zero, the created RESTClient will use DefaultBurst: 10.
Burst int
// Rate limiter for limiting connections to the master from this client. If present overwrites QPS/Burst
RateLimiter flowcontrol.RateLimiter
```

主要就是2个参数，QPS和Burst

- QPS：每秒补充多少token，默认值5
- Burst： 总token的上限，即令牌桶的容量，默认值10

## 限流算法

> 目前主流两种限流算法，漏桶算法和令牌桶算法

client-go 使用了 `golang.org/x/time/rat` 包来实现，基于的是令牌桶限速算法
所谓令牌桶算法简单来说，就是有一个令牌桶，初始容量是Burst。一个Request会消耗一个令牌（token），系统已恒定的速率（QPS）往令牌桶里面放令牌，当很多请求进来，令牌桶空了的时候，新的Request就得排队了

## 用法

自己设置一个限流器，来调整令牌桶，注意如果是`metrics.Clientset` ，同样也需要设置限流器

```go
cfg, err = clientcmd.BuildConfigFromFlags(master, kubeconfig)
······
cfg.RateLimiter = flowcontrol.NewTokenBucketRateLimiter(100, 100)
······
return clientset.NewForConfig(cfg)
```

# 附：控制面组件访问最佳实践

大规模集群（节点规模大于100，Kubernentes资源量较大）的场景下，可以提升集群整体稳定性。

- 尽量使用Informer、Lister方式从APIServer读取数据，对APIServer和etcd综合压力小。

- 如果必须要使用全量LIST，建议请求增加`resourceVersion=0`，从APIServer Cache中读取数据，避免一次请求访问全量击穿到etcd，如果确实需要从etcd读取数据，需要基于Limit使用分页访问。

- API序列化协议使用`Protobuf`，相比于JSON更节省内存和传输流量。更多信息，请参见[Alternate representations of resources](https://kubernetes.io/docs/reference/using-api/api-concepts/#alternate-representations-of-resources)。代码样例如下：
  
  ```go
  kubeConfig, err := clientcmd.BuildConfigFromFlags(s.Master, s.Kubeconfig)
  if err != nil {
      return nil, err
  }
  kubeConfig.AcceptContentTypes = strings.Join([]string{runtime.ContentTypeProtobuf, runtime.ContentTypeJSON}, ",")
  kubeConfig.ContentType = runtime.ContentTypeProtobuf
  client, err := clientset.NewForConfig(restclient.AddUserAgent(kubeConfig, "content-type-example"))
  ...
  ```

- 及时清理不使用的Kubernetes资源，例如ConfigMap、Secret和PVC等。避免出现超过1000的Pending Pod，因为大量Pending Pod会对kube-apsierver、kube-controller-manager和kube-scheduler持续产生压力。

- 注意关注控制平面组件使用情况，尤其是CPU和内存利用率指标，避免持续高水位导致组件OOM等异常。如果出现持续高水位，建议通过清理无效资源、优化客户端行为、拆分集群业务等措施，保证集群处于合理水位。

- 部分开源组件对控制平面压力较大，官方输出了相应的优化治理方案，建议关注并应用到实践。以Argo workflow为例，官方推出的Kube API过量访问的优化方案建议使用Argo时，需开启相关配置