# Kubeedge简介

kubeedge是华为开源的边缘云容器插件, 基于Kubernetes, 本质上是提供了Kubernetes的Controller来监控资源的变化并和APIServer, 边缘节点进行信息的交互

官方文档见链接:  https://docs.kubeedge.io/en/latest/

## 干货分享

- Kubernetes只需要部署master节点即可, 不需要部署work node
- Pod对外暴露服务目前只支持hostnetwork模式, 限制颇多
- 目前cloud core支持二进制和deployment部署, 而edge core只支持二进制部署
- 目前原生无高可用设计, 得自己考虑
- Mapper现在kubeedge只提供模型, 和几个参考实现, 推荐使用daemonset部署到每个节点
- EdgeMesh和一般的SerivceMesh区别是一般ServiceMesh使用SideCar方式, 而EdgeMesh为了减少资源消耗使用每个节点部署一个Proxy, 而且内置域名解析能力, 不依赖于中心DNS

# 实战部署

基于最新的1.3版本试用kubeedge, 目前有2种方式安装, 一种是使用Keadm, 一种是Locally本地安装, Locally方式官方不推荐使用在生产环境, 因为本次略过该种方式.

安装包下载地址为: https://github.com/kubeedge/kubeedge/releases , 本次下载了1.3.1版本, 需要下载对应的keadm和kubeedge安装包, keadm就一个执行文件( 实际只需要下载keadm )

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

执行命令

```
./keadm init --advertise-address="172.16.104.171" --kubeedge-version=1.3.1  --kube-config=/root/.kube/config
```

> 注意: github的素材库可能会被DNS污染, 我在深圳3个不同环境都解析不了该域名, 解决办法是登录 https://site.ip138.com/raw.Githubusercontent.com/, 输入raw.githubusercontent.com查询该域名的IP地址, 然后写入/etc/hosts中

最后输出如下, 表面Cloud Core运行成功

```
Converted 0 files in 0 seconds.
kubeedge-v1.3.1-linux-amd64.tar.gz checksum:
checksum_kubeedge-v1.3.1-linux-amd64.tar.gz.txt content:
./kubeedge-v1.3.1-linux-amd64/
./kubeedge-v1.3.1-linux-amd64/edge/
./kubeedge-v1.3.1-linux-amd64/edge/edgecore
./kubeedge-v1.3.1-linux-amd64/cloud/
./kubeedge-v1.3.1-linux-amd64/cloud/csidriver/
./kubeedge-v1.3.1-linux-amd64/cloud/csidriver/csidriver
./kubeedge-v1.3.1-linux-amd64/cloud/admission/
./kubeedge-v1.3.1-linux-amd64/cloud/admission/admission
./kubeedge-v1.3.1-linux-amd64/cloud/cloudcore/
./kubeedge-v1.3.1-linux-amd64/cloud/cloudcore/cloudcore
./kubeedge-v1.3.1-linux-amd64/version

KubeEdge cloudcore is running, For logs visit:  /var/log/kubeedge/cloudcore.log
CloudCore started
```

查看进程, 可以看到已有cloudcore进程, 日志可以到上面提示的位置查看.

```
[root@centos-kata keadm]# ps aux|grep cloudcore
root      17575  0.1  0.7 341196 43580 pts/0    Sl   09:09   0:01 cloudcore
```

## Edge Core安装

先在Edge Node安装好docker, 过程省略

在刚刚**安装Cloud Side的节点**执行命令获取token

```
[root@centos-kata keadm]# ./keadm gettoken
402399f727a4a6ce4b298633cb81d3081d5d16f96ce81acec36189fb2633f2df.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1OTM4MjQ4MDZ9.nDqXs4hepfNZrrkEMO8xMHfqcQAl91Z6TQ8-SAG2Fg8
```

然后在edge node执行join命令, 注意最好也配置下域名被污染的问题

```
./keadm join --cloudcore-ipport=172.16.104.171:10000 --token=402399f727a4a6ce4b298633cb81d3081d5d16f96ce81acec36189fb2633f2df.eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1OTM4MjQ4MDZ9.nDqXs4hepfNZrrkEMO8xMHfqcQAl91Z6TQ8-SAG2Fg8 --edgenode-name=czsedge --kubeedge-version=1.3.1
```

注意: **节点名称一定要是小写的, 不然会加节点失败**

最后输出如下:

```
onverted 0 files in 0 seconds.
kubeedge-v1.3.1-linux-amd64.tar.gz checksum:
checksum_kubeedge-v1.3.1-linux-amd64.tar.gz.txt content:
./kubeedge-v1.3.1-linux-amd64/
./kubeedge-v1.3.1-linux-amd64/edge/
./kubeedge-v1.3.1-linux-amd64/edge/edgecore
./kubeedge-v1.3.1-linux-amd64/cloud/
./kubeedge-v1.3.1-linux-amd64/cloud/csidriver/
./kubeedge-v1.3.1-linux-amd64/cloud/csidriver/csidriver
./kubeedge-v1.3.1-linux-amd64/cloud/admission/
./kubeedge-v1.3.1-linux-amd64/cloud/admission/admission
./kubeedge-v1.3.1-linux-amd64/cloud/cloudcore/
./kubeedge-v1.3.1-linux-amd64/cloud/cloudcore/cloudcore
./kubeedge-v1.3.1-linux-amd64/version

KubeEdge edgecore is running, For logs visit:  /var/log/kubeedge/edgecore.log
```

查看安装日志, 主要就是下载了mosquitto作为MQTT Server和下载kubeedge安装包

验证下MQTT服务是否正常

```
[root@localhost keadm]# mosquitto_pub -h 172.16.104.133 -p 1883 -t "hello" -m "this is hello world"
```

如果出现`Error: Connection refused`字段表明服务或者端口没启动

然后在master节点上查看, 可以看到edge node已经加入, 如果没加入, 则看cloud core和edge core的日志

```
[root@centos-kata keadm]# kubectl get nodes
NAME          STATUS   ROLES        AGE    VERSION
centos-kata   Ready    master       125d   v1.17.1
czsedge       Ready    agent,edge   100m   v1.17.1-kubeedge-v1.3.1
```

## 安装说明

kubeedge使用的主要有2个文件夹

- `/etc/kubeedge/` : 存放可执行文件, 配置信息, 证书(edge node才会有证书), 目录结构如下:

  ```
  [root@centos-kata kubeedge]# tree
  .
  |-- ca
  |-- certs
  |-- config
  |   `-- cloudcore.yaml
  |-- crds
  |   |-- devices
  |   |   |-- devices_v1alpha1_devicemodel.yaml
  |   |   `-- devices_v1alpha1_device.yaml
  |   `-- reliablesyncs
  |       |-- cluster_objectsync_v1alpha1.yaml
  |       `-- objectsync_v1alpha1.yaml
  |-- kubeedge-v1.3.1-linux-amd64
  |   |-- cloud
  |   |   |-- admission
  |   |   |   `-- admission
  |   |   |-- cloudcore
  |   |   |   `-- cloudcore
  |   |   `-- csidriver
  |   |       `-- csidriver
  |   |-- edge
  |   |   `-- edgecore
  |   `-- version
  `-- kubeedge-v1.3.1-linux-amd64.tar.gz
  ```

- `/var/lib/kubeedge` : 云端存放sock文件, edge存放db文件

安装后配置文件的位置是 `/etc/kubeedge/config`,  cloudcore和edgecore配置文件不同



## 测试实践

运行一个deployment, 然后查看

```
[root@centos-kata ~]# kubectl get po -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP       NODE      NOMINATED NODE   READINESS GATES
my-nginx-6f5b78c795-kx6zn   1/1     Running   0          97s   <none>   czsedge   <none>           <none>
```

容器已经运行起来, 在edge node查看容器, 可以看到容器已经运行成功

```
[root@localhost keadm]# docker ps
CONTAINER ID        IMAGE                COMMAND                  CREATED             STATUS              PORTS               NAMES
6ca36d2e8cc4        nginx                "/docker-entrypoint.…"   5 minutes ago       Up 5 minutes                            k8s_my-nginx_my-nginx-6f5b78c795-kx6zn_default_66cd3953-e70e-45be-ac35-f0aa05cd8871_0
1b6d4eeb4da9        kubeedge/pause:3.1   "/pause"                 6 minutes ago       Up 6 minutes                            k8s_POD_my-nginx-6f5b78c795-kx6zn_default_66cd3953-e70e-45be-ac35-f0aa05cd8871_0
```

### 存在问题

1. kubectl logs报错, 报错如下:  待继续分析

   ```
   [root@centos-kata ~]# kubectl logs my-nginx-6f5b78c795-kx6zn
   Error from server: Get https://172.16.104.133:10350/containerLogs/default/my-nginx-6f5b78c795-kx6zn/my-nginx: dial tcp 172.16.104.133:10350: connect: connection refused
   ```

   需要添加iptables规则:

   

2. 目前不支持kubectl exec命令, 预计2020.08月支持

   

3. 默认Pod起来没有IP, 需要配置edge的IP,  待分析

   ```
   [root@centos-kata ~]# kubectl get po -o wide
   NAME                        READY   STATUS    RESTARTS   AGE   IP       NODE      NOMINATED NODE   READINESS GATES
   my-nginx-6f5b78c795-kx6zn   1/1     Running   0          49m   <none>   czsedge   <none>           <none>
   ```

4. 目前无高可用, 需要把edgecore和docker设置为systemctl模式启动, 并设置开机启动, 不然节点重启之后不能拉起进程.

   edgecore设置为service, 创建service文件如下: 

```
[root@localhost kubeedge]# cat /etc/systemd/system/edgecore.service
[Unit]
Description=edgecore.service

[Service]
Type=simple
ExecStart=/etc/kubeedge/kubeedge-v1.3.1-linux-amd64/edge/edgecore

[Install]
WantedBy=multi-user.target
```

5. 发现如果cloudcore重启了, edgecore没重启, 就会出问题, 节点连不上



