
# 环境信息

三个VM,一个为master,另外两个为worker,并关闭firewalld和selinux.

```
vm1: 172.16.104.131
vm2: 172.16.104.132
vm3: 172.16.104.133
```

配置网易的yum源

在每个节点安装docker,etcd flannel kubernetes(node不需要使用etcd)
```
yum install -y docker etcd flannel kubernetes
```

关闭selinux 和firewalld(这步省略)

```
vim /etc/selinux/config

SELINUX=disable
```

# Master节点

master节点需要安装以下组件：

- etcd
- flannel
- docker
- kubernets

## etcd

- 修改etcd配置文件`/etc/etcd/etcd.conf`

```
ETCD_ADVERTISE_CLIENT_URLS="http://172.16.104.131:2379"
```

- 启动etcd

```
systemctl start etcd
systemctl enable etcd
```

- 验证etcd是否成功

```
[root@vm1 default.etcd]# etcdctl -C http://172.16.104.131:2379 cluster-health
member 8e9e05c52164694d is healthy: got healthy result from http://172.16.104.131:2379
cluster is healthy
```

## flannel

- 配置flannel：`/etc/sysconfig/flanneld`

```
FLANNEL_ETCD_ENDPOINTS="http://172.16.104.131:2379"
```

- 配置etcd中关于flannel的key

```
etcdctl mk /atomic.io/network/config '{ "Network": "10.0.0.0/16" }'
```

- 启动flannel并设置开机自启

```
[root@vm1 default.etcd]# systemctl start flanneld
[root@vm1 default.etcd]# systemctl enable flanneld
```

## docker

安装省略,不需要特别配置,直接启动即可.

## kubernetes

master上需要运行以下组件：

- kube-apiserver
- kube-scheduler
- kube-controller-manager

- 配置apiserver, `/etc/kubernetes/apiserver`

注意,如果没有配置身份认证的话,去掉配置文件`KUBE_ADMISSION_CONTROL`项中的`SecurityContextDeny,ServiceAccount`

```
KUBE_API_ADDRESS="--insecure-bind-address=0.0.0.0"

# The port on the local server to listen on.
KUBE_API_PORT="--port=8080"

# Port minions listen on
KUBELET_PORT="--kubelet-port=10250"

KUBE_ETCD_SERVERS="--etcd-servers=http://172.16.104.131:2379"
```

- 配置`/etc/kubernetes/config`

```
KUBE_MASTER="--master=http://172.16.104.131:8080"
```

- 启动k8s各服务并设置为开机自启

```
[root@vm1 default.etcd]# systemctl start kube-apiserver
[root@vm1 default.etcd]# systemctl start kube-controller-manager
[root@vm1 default.etcd]# systemctl start kube-scheduler

[root@vm1 default.etcd]# systemctl enable kube-apiserver
[root@vm1 default.etcd]# systemctl enable kube-controller-manager
[root@vm1 default.etcd]# systemctl enable kube-scheduler
```

# node节点

节点需要安装以下组件：

- flannel
- docker
- kubernetes

## flannel

不需要配置etcd

- 配置flannel：`/etc/sysconfig/flanneld`

```
FLANNEL_ETCD_ENDPOINTS="http://172.16.104.131:2379"
```

- 启动flannel并设置开机自启

```
[root@vm1 default.etcd]# systemctl start flanneld
[root@vm1 default.etcd]# systemctl enable flanneld
```

## docker

启动并设置为开机自启动

## kubernetes

不同于master节点，slave节点上需要运行kubernetes的如下组件：

- kubelet
- kubernets-proxy

- 配置`/etc/kubernetes/config`

```
KUBE_MASTER="--master=http://172.16.104.131:8080"
```

- 配置`/etc/kubernetes/kubelet`

```
# The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--address=0.0.0.0"

# You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname-override=172.16.104.132"

# location of the api-server
KUBELET_API_SERVER="--api-servers=http://172.16.104.131:8080"
```

- 启动kube服务

```
systemctl start kubelet.service
systemctl start kube-proxy.service

systemctl enable kubelet.service
systemctl enable kube-proxy.service
```

## 配置kubectl

```
kubectl config set-cluster kubernetes --server=http://172.16.104.131:8080

kubectl config set-credentials kubectl

kubectl config set-context kubernetes --cluster=kubernetes --user=kubectl --namespace=${namesapce}

kubectl config use-context kubernetes
```

# 验证集群是否成功

```
root@vm1 kubernetes]# kubectl get endpoints
NAME         ENDPOINTS             AGE
kubernetes   172.16.104.131:6443   2h

[root@vm1 kubernetes]# kubectl cluster-info
Kubernetes master is running at http://localhost:8080

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

[root@vm1 kubernetes]# kubectl get nodes
NAME             STATUS    AGE
172.16.104.132   Ready     1h
172.16.104.133   Ready     6m
```

# 故障解决

## No API token

查看pods状态, 提示"No Resource"

```
[root@vm1 ~]# kubectl get pods --all-namespaces
No resources found.
```

查看event有如下打印

```
27s        4m          30        mycentos-2383264970             ReplicaSet               Warning   FailedCreate            {replicaset-controller }   Error creating: No API token found for service account "default", retry after the token is automatically created and added to the service account
```

是因为没有配置身份认证的话,去掉配置文件`/etc/kubernetes/apiserver`里面`KUBE_ADMISSION_CONTROL`项中的`SecurityContextDeny,ServiceAccount`

## 找不到镜像"pod-infrastructure"

查看pod状态,发现一直不是ready状态

```
[root@vm1 kubernetes]# kubectl get pods
NAME                                  READY     STATUS              RESTARTS   AGE
kubernetes-bootcamp-390780338-p7hss   0/1       ContainerCreating   0          8h
```

查看日志

```
m        28m       117       mycentos2-1008844078-rqxck   Pod                 Warning   FailedSync   {kubelet 172.16.104.132}   Error syncing pod, skipping: failed to "StartContainer" for "POD" with ImagePullBackOff: "Back-off pulling image \"registry.access.redhat.com/rhel7/pod-infrastructure:latest\""
```

解决方法是, 在node节点, 可以登录国内yum源, os目录下, 下载两个rpm包: `python-rhsm-1.19.10-1.el7_4.x86_64.rpm`, `python-rhsm-certificates-1.19.10-1.el7_4.x86_64.rpm`

删除rpm包`subscription-manager-rhsm-certificates.x86_64 0:1.20.11-1.el7.centos`

然后安装上面下载的两个rpm包, 再pull缺的镜像

```
docker pull registry.access.redhat.com/rhel7/pod-infrastructure:latest
```

后面就可以正常创建deployment了.
