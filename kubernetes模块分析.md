# kubernetes模块分析

标签（空格分隔）： kubernetes

---

# API Server

k8s API Server提供了k8s各类资源对象（pod,RC,Service等）的增删改查及watch等HTTP Rest接口，是整个系统的数据总线和数据中心(只有API Server才直接操作etcd)。

部署在master节点, 通过进程`kube-apiserver`提供服务, 使用8080端口;

> - 在master节点上默认IP是`localhost`, 可以通过启动参数“--insecure-bind-address”的值来修改该IP地址
- 默认值端口为8080，可以通过API Server的启动参数“--insecure-port”的值来修改默认值

在node节点, Pod中的进程如何知道API Server的访问地址呢?

因为node节点把API Server做成了一个Service, 名字是"kubernetes", 并且它的ClusterIP地址就是Cluster IP地址池的第一个地址, 另外端口是HTTPS端口443

```
[root@vm1 k8s]# kubectl get svc -o wide
NAME         CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE       SELECTOR
kubernetes   10.254.0.1       <none>        443/TCP    61d       <none>
```

## API

在master节点, 运行curl命令, 得到JSON方式返回的API

### API版本

```
[root@vm1 k8s]# curl localhost:8080/api
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "172.16.104.131:6443"
    }
  ]
```

## 查看API Server支持的资源对象种类

```
root@vm1 k8s]# curl localhost:8080/api/v1
{
  "kind": "APIResourceList",
  "groupVersion": "v1",
  "resources": [
    {
      "name": "bindings",
      "namespaced": true,
      "kind": "Binding"
    },
    
    ......
```

### 查看某种资源

```
curl localhost:8080/api/v1/pods
curl localhost:8080/api/v1/services
curl localhost:8080/api/v1/replicationcontrollers
```

### 安全机制

可以使用`kubectl proxy`只暴露部分服务

```
kubectl proxy --reject-paths="^/api/v1/replicationcontrollers/" --port=8001 --v=2  #拒绝访问replicationcontrollers
```

可以设置白名单, 比如增加如下参数

```
--accept-hosts="localhost$,^127\\.0\\.0\\.1$,^\\[::1\\]$"
```

## Proxy API

k8s API Server最主要的REST接口是资源对象的增删改查，另外还有一类特殊的REST接口—k8s Proxy API接口，这类接口的作用是代理REST请求，即kubernetes API Server把收到的REST请求转发到某个Node上的kubelet守护进程的REST端口上，由该kubelet进程负责响应。








