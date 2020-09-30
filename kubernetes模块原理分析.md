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



<img src="https://mmbiz.qpic.cn/mmbiz_png/HO2NI1o25ZYcrYxA727gp7I1H3uianCiaicKvFO7vicVqrrHJiawPvnAbANGvl7IKLPNiadibeWx9fYyibYLnpelk0EgTw/640?wx_fmt=png&amp;tp=webp&amp;wxfrom=5&amp;wx_lazy=1&amp;wx_co=1" alt="APIServer层级" style="zoom:67%;" />






