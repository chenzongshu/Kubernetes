# Kubeedge简介

kubeedge是华为开源的边缘云容器插件, 基于Kubernetes, 本质上是提供了Kubernetes的Controller来监控资源的变化并和边缘节点进行信息的交互

官方文档见链接:  https://docs.kubeedge.io/en/latest/



# 实战部署

基于最新的1.3版本试用kubeedge, 目前有2种方式安装, 一种是使用Keadm, 一种是Locally本地安装, Locally方式官方不推荐使用在生产环境, 因为本次略过该种方式.

安装包下载地址为: https://github.com/kubeedge/kubeedge/releases , 本次下载了1.3.1版本, 需要下载对应的keadm和kubeedge安装包, keadm就一个执行文件, 

## kubeedge Cloud Core安装

### Kubernetes集群

首先我们需要一个Kubernetes集群, 这里集群安装过程省略, 可以去找我以前的文档, 为了节约资源, 这里准备了一个master, 一个worker, 一个边缘节点( 边缘节点在未加入集群时候不可见 ), 如下: 

```
[root@centos-kata ~]# kubectl get nodes
NAME          STATUS   ROLES    AGE    VERSION
centos-kata   Ready    master   125d   v1.17.1
vm1           Ready    <none>   125d   v1.17.1
```

### Cloud Core安装

注意:

-  目前keadm只支持Ubuntu和CentOS
- keadm需要root权限
- 默认端口是10000和10002(1.3版本开始使用)

`keadm init`将安装cloud core，生成证书并安装CRD。它还提供了一个标志，通过它可以设置特定的版本。



