# Etcd集群搭建

## 环境信息

主机1 | 主机2 | 主机3  
------- | ------- | -------  
10.25.72.164 | 10.25.72.233 | 10.25.73.196  

## 安装etcd

`yum install -y etcd` 安装etcd

## 配置第一台

编辑etcd配置文件

`vim /etc/etcd/etcd.conf`

```
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"  #etcd数据保存目录
ETCD_LISTEN_CLIENT_URLS="http://10.25.72.164:2379,http://localhost:2379"  #供外部客户端使用的url
ETCD_ADVERTISE_CLIENT_URLS="http://10.25.72.164:2379,http://localhost:2379" #广播给外部客户端使用的url
ETCD_NAME="etcd1"   #etcd实例名称

ETCD_LISTEN_PEER_URLS="http://10.25.72.164:2380"  #集群内部通信使用的URL
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.25.72.164:2380"  #广播给集群内其他成员访问的URL
ETCD_INITIAL_CLUSTER="etcd1=http://10.25.72.164:2380,etcd2=http://10.25.72.233:2380,etcd3=http://10.25.73.196:2380"    #初始集群成员列表
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster" #集群的名称
ETCD_INITIAL_CLUSTER_STATE="new"  #初始集群状态，new为新建集群

```

然后执行`systemctl start etcd`启动etcd进程

## 其他两台

etcd2和etcd3为加入etcd-cluster集群的实例，需要将其ETCD_INITIAL_CLUSTER_STATE设置为"exist"

```
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"  
ETCD_LISTEN_CLIENT_URLS="http://10.25.72.233:2379,http://localhost:2379"  
ETCD_ADVERTISE_CLIENT_URLS="http://10.25.72.233:2379,http://localhost:2379" 
ETCD_NAME="etcd2"  

ETCD_LISTEN_PEER_URLS="http://10.25.72.233:2380" 
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.25.72.233:2380"
ETCD_INITIAL_CLUSTER="etcd1=http://10.25.72.164:2380,etcd2=http://10.25.72.233:2380,etcd3=http://10.25.73.196:2380"  
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="exist"  

```
搭建完毕，可以查看集群节点来确定有没有搭建成功

```
[root@SZD-L0110301 default.etcd]# etcdctl member list
4536e7d0bdb3b43c: name=etcd3 peerURLs=http://10.25.73.196:2380 clientURLs=http://10.25.73.196:2379,http://localhost:2379 isLeader=false
c441e6c11a47ff3d: name=etcd1 peerURLs=http://10.25.72.164:2380 clientURLs=http://10.25.72.164:2379,http://localhost:2379 isLeader=true
ddc007546c89f163: name=etcd2 peerURLs=http://10.25.72.223:2380 clientURLs=http://10.25.72.223:2379,http://localhost:2379 isLeader=false
```

# Etcd使用

## 集群数据操作命令

### 设键值

```
[root@SZD-L0110301 default.etcd]# etcdctl set /testdir/testkey "hello world"
hello world
```

- key存在的方式和zookeeper类似，为  `/路径/key`
- 设置完之后，其他集群也可以查询到该值
- 如果`dir`和`key`不存在，该命令会创建对应的项

### 查看键值

切换到另外一个节点

```
[root@SZD-L0110303 etcd]# etcdctl get /testdir/testkey
hello world
```
不存在的时候会报错

### 更新
当键不存在时，会报错

```
[root@SZD-L0110303 etcd]# etcdctl update /testdir/testkey "hello bruce"
hello bruce
```

### 删除

```
[root@SZD-L0110303 etcd]# etcdctl rm /testdir/testkey
PrevNode.Value: hello bruce
```

更多的操作命令省略，可以见`help`

## 查看API的版本

```
[root@SZD-L0072834 ~]# etcdctl -version
etcdctl version: 3.1.10
API version: 2
```
## 切换API版本

```
export ETCDCTL_API=3
```

## 集群管理命令


## 查看API的版本

```
[root@SZD-L0072834 ~]# etcdctl -version
etcdctl version: 3.1.10
API version: 2
```
## 切换API版本

```
export ETCDCTL_API=3
```

### 查看集群健康状态

```
[root@SZD-L0110301 default.etcd]# etcdctl cluster-health
member 4536e7d0bdb3b43c is healthy: got healthy result from http://10.25.73.196:2379
member c441e6c11a47ff3d is healthy: got healthy result from http://10.25.72.164:2379
member ddc007546c89f163 is healthy: got healthy result from http://10.25.72.223:2379
cluster is healthy
```

### backup

备份 etcd 的数据,参数有：

> --data-dir         etcd 的数据目录
--backup-dir     备份到指定路径

### watch

监测一个键值的变化，一旦键值发生更新，就会输出最新的值并退出。

```
[root@SZD-L0110301 default.etcd]# etcdctl watch /testdir/testkey
hello bruce
```

```
[root@SZD-L0110303 etcd]# etcdctl update /testdir/testkey "hello bruce"
```

- 在第二个节点update之后，第一个watch的才有结果输出
- watch会直接退出，如果不想退出可以设置 `--forever`参数, 这样就会一直监测，直到用户按 `CTRL+C` 退出

### exec-watch

监测一个键值的变化，一旦键值发生更新，就执行给定命令。

```
[root@SZD-L0110301 default.etcd]# etcdctl exec-watch /testdir/testkey -- sh -c 'ls'
member
```
