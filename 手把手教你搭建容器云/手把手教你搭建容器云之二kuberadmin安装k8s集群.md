# 准备


## 禁用swap

Kubernetes 1.8开始要求必须禁用Swap，如果不关闭，默认配置下kubelet将无法启动。

```
vim /etc/fstab

# / was on /dev/sda1 during installation
UUID=8cc33106-20fc-43b7-ad52-298eed8ccae6 /               ext4    errors=remount-ro 0       1
#/swapfile                                 none            swap    sw              0       0
```

把`swapfile`行注释掉后执行

```
swapoff -a
```

## 系统配置

```
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

设置国内kubernetes阿里云yum源

```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

设置docker源
```
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -P /etc/yum.repos.d/
```

加载ipvs内核，使node节点kube-proxy支持ipvs代理规则。
```
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
```

并添加到开机启动文件/etc/rc.local里面。
```
cat <<EOF >> /etc/rc.local
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
EOF
```

## 安装docker、kubeadm、 kubelet 和 kubectl



```
yum install -y docker-ce kubeadm kubelet kubectl
```

如果是Kubernetes 1.17版本, 可能还需要修改docker配置文件, 

```
vim /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```

# 使用kubeadm创建一个单Master集群

- 1.选择一个网络插件，并检查它是否需要在初始化Master时指定一些参数，比如我们可能需要根据选择的插件来设置`--pod-network-cidr`参数。

- 2.kubeadm使用eth0的默认网络接口（通常是内网IP）做为Master节点的advertise address，如果我们想使用不同的网络接口，可以使用`--apiserver-advertise-address=<ip-address>`参数来设置。如果适应IPv6，则必须使用IPv6d的地址，如：`--apiserver-advertise-address=fd00::101`。

- 3.使用`kubeadm config images pull`来预先拉取初始化需要用到的镜像，用来检查是否能连接到Kubenetes的Registries。

Kubenetes默认Registries地址是`k8s.gcr.io`，很明显，在国内并不能访问gcr.io，因此在kubeadm v1.13之前的版本，安装起来非常麻烦，但是在1.13版本中终于解决了国内的痛点，其增加了一个`--image-repository`参数，默认值是`k8s.gcr.io`，我们将其指定为国内镜像地址：`registry.aliyuncs.com/google_containers`，其它的就可以完全按照官方文档来愉快的玩耍了。

其次，我们还需要指定`--kubernetes-version`参数，因为它的默认值是`stable-1`，会导致从`https://dl.k8s.io/release/stable-1.txt`下载最新的版本号，我们可以将其指定为固定版本（最新版：v1.13.1）来跳过网络请求。

如果是本地镜像, 可以使用`kubeadm config images list --kubernetes-version=v1.17.0 `查看对应版本需要哪些镜像

然后创建

```
root@czs-virtual-machine:~# kubeadm init --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.19.3 --pod-network-cidr=10.244.0.0/16
[init] Using Kubernetes version: v1.17.0
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [czs-virtual-machine localhost] and IPs [172.16.104.170 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [czs-virtual-machine localhost] and IPs [172.16.104.170 127.0.0.1 ::1]
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [czs-virtual-machine kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 172.16.104.170]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[kubelet-check] Initial timeout of 40s passed.
[apiclient] All control plane components are healthy after 65.503845 seconds
[upload-config] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.14" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --experimental-upload-certs
[mark-control-plane] Marking the node czs-virtual-machine as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node czs-virtual-machine as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: q3j2nl.aepa1nc8fjjhfixx
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.16.104.170:6443 --token q3j2nl.aepa1nc8fjjhfixx \
    --discovery-token-ca-cert-hash sha256:72d3c4a3033ac4b0edfedc1c6320c7dd18437b2a1579a27dbbf2670d7d61dc2e
```

ubuntu系统下, `kube-apiserver`的监听端口可以看到只监听了https的6443端口

```
root@czs-virtual-machine:~# netstat -nltp | grep apiserver
tcp6       0      0 :::6443                 :::*                    LISTEN      2149/kube-apiserver
```

为了使用kubectl访问apiserver, 需要进行配置

```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

记住其他节点需要创建集群的master的节点的对应文件, 就能执行`kubectl`了

然后即可以看到pod

```
root@czs-virtual-machine:~# kubectl get pods --all-namespaces
NAMESPACE     NAME                                          READY   STATUS    RESTARTS   AGE
kube-system   coredns-8686dcc4fd-4j57c                      0/1     Pending   0          171m
kube-system   coredns-8686dcc4fd-kzqw4                      0/1     Pending   0          171m
kube-system   etcd-czs-virtual-machine                      1/1     Running   8          171m
kube-system   kube-apiserver-czs-virtual-machine            1/1     Running   19         171m
kube-system   kube-controller-manager-czs-virtual-machine   1/1     Running   3          171m
kube-system   kube-proxy-vgkfp                              1/1     Running   1          171m
kube-system   kube-scheduler-czs-virtual-machine            1/1     Running   3          171m
```

可以看到`coredns`处于pending状态, 因为还没安装网络插件

## 安装flanneld

flanneld或者Calico只需要安装一个, 这里我们flanneld采用daemonset模式安装

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

等镜像下载完成, 安装完即可以看到对应的ds的pod已经起来

```
[root@k8s-master ~]# kubectl get po --all-namespaces
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   kube-flannel-ds-55wnp                1/1     Running   0          100s
kube-system   kube-flannel-ds-5gz5j                1/1     Running   0          100s
```

## 安装calico

默认情况下，`Calico`网络插件使用的的网段是`192.168.0.0/16`，在init的时候，我们已经通过`--pod-network-cidr=192.168.0.0/16`来适配Calico，当然你也可以修改`calico.yml`文件来指定不同的网段。



先下载calico镜像, 有4个镜像

```
[root@localhost czs]# docker images
REPOSITORY                                                        TAG                 IMAGE ID            CREATED             SIZE
calico/node                                                       v3.11.2             81f501755bb9        4 days ago          255MB
calico/cni                                                        v3.11.2             c317181e3b59        4 days ago          204MB
calico/pod2daemon-flexvol                                         v3.11.2             f69bca7e2325        5 days ago          111MB
calico/kube-controllers                                           v3.11.2             9e897df2f2af        5 days ago          52.5MB
```

然后下载对应的yaml文件, 注意修改里面的镜像版本, 

```
wget https://docs.projectcalico.org/v3.11/manifests/calico.yaml
```

然后使用`apply`命令跑起来，`kubectl apply -f calico.yaml`


然后可以看到全部pods拉起来, 变成了running状态

```
root@czs-virtual-machine:~# kubectl get pods --all-namespaces
NAMESPACE     NAME                                          READY   STATUS    RESTARTS   AGE
kube-system   calico-node-sk75d                             2/2     Running   0          2m30s
kube-system   coredns-8686dcc4fd-4j57c                      0/1     Running   0          4h56m
kube-system   coredns-8686dcc4fd-kzqw4                      1/1     Running   0          4h56m
kube-system   etcd-czs-virtual-machine                      1/1     Running   17         4h56m
kube-system   kube-apiserver-czs-virtual-machine            1/1     Running   34         4h56m
kube-system   kube-controller-manager-czs-virtual-machine   1/1     Running   12         4h56m
kube-system   kube-proxy-vgkfp                              1/1     Running   5          4h56m
kube-system   kube-scheduler-czs-virtual-machine            1/1     Running   12         4h56m
```

## Master隔离

默认情况下，由于安全原因，集群并不会将pods部署在Master节点上。但是在开发环境下，我们可能就只有一个Master节点，这时可以使用下面的命令来解除这个限制：

```
root@czs-virtual-machine:~# kubectl taint nodes --all node-role.kubernetes.io/master-
```

如果需要恢复master隔离, 可以执行命令

```
kubectl taint nodes <master node name> node-role.kubernetes.io/master=:NoSchedule
```



## 加入工作节点

要把其他节点加入集群, 也可以使用kubeadm

成为root用户, 运行`kubeadm init`输出的结果来

```
kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>
```

如果我们忘记了Master节点的加入token，可以使用如下命令来查看

```
root@czs-virtual-machine:~# kubeadm token list
TOKEN                     TTL       EXPIRES                     USAGES                   DESCRIPTION                                                EXTRA GROUPS
q3j2nl.aepa1nc8fjjhfixx   1h        2019-04-09T18:29:14+08:00   authentication,signing   The default bootstrap token generated by 'kubeadm init'.   system:bootstrappers:kubeadm:default-node-token
```

默认情况下，token的有效期是24小时，如果我们的token已经过期的话，可以使用以下命令重新生成：

```
kubeadm token create

# 输出
u2mt59.tyqpo0v5wf05lx2q
```

如果我们也没有`--discovery-token-ca-cert-hash`的值，可以使用以下命令生成：

```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'

# 输出
eebfe256113bee397b218ba832f412273ae734bd4686241fb910885d26efd222
```

然后, 登录到工作节点服务器，然后运行如下命令加入集群

```
sudo kubeadm join 172.17.20.210:6443 --token 6pkrlg.8glf2fqpuf3i489m --discovery-token-ca-cert-hash sha256:eebfe256113bee397b218ba832f412273ae734bd4686241fb910885d26efd222
```

## 验证

```
kubectl create deployment nginx --image=nginx:alpine
kubectl scale deployment nginx --replicas=2

# 验证Nginx Pod是否正确运行，并且会分配192.168.开头的集群IP

root@czs-virtual-machine:~# kubectl get pods -l app=nginx -o wide
NAME                    READY   STATUS    RESTARTS   AGE     IP            NODE                  NOMINATED NODE   READINESS GATES
nginx-77595c695-44znf   1/1     Running   0          2m55s   192.168.0.4   czs-virtual-machine   <none>           <none>
nginx-77595c695-9srkm   1/1     Running   0          4s      192.168.0.5   czs-virtual-machine   <none>           <none>
```

再验证一下`kube-proxy`是否正常：

```
# 以 NodePort 方式对外提供服务
root@czs-virtual-machine:~# kubectl expose deployment nginx --port=80 --type=NodePort
service/nginx exposed

root@czs-virtual-machine:~# kubectl get services nginx
NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
nginx   NodePort   10.99.235.119   <none>        80:30357/TCP   14s

```

在另外的环境就可以通过端口访问该服务

```
chenzongshudeMacBook-Pro:vmnet8 chenzongshu$ curl http://172.16.104.170:30357

<!DOCTYPE html>
......省略......
```

# 卸载集群

想要撤销kubeadm执行的操作，首先要排除节点，并确保该节点为空, 然后再将其关闭。

在Master节点上运行：

```
kubectl drain <node name> --delete-local-data --force --ignore-daemonsets
kubectl delete node <node name>
```

然后在需要移除的节点上，重置kubeadm的安装状态：

```
sudo kubeadm reset
```

如果你想重新配置集群，使用新的参数重新运行`kubeadm init`或者`kubeadm join`即可


# 部署高可用Kubernetes集群

采用keepalived + haproxy软负载的方案, 考虑到VIP分配的问题, 尝试使用一个master节点的真实IP作为VIP(但是后续失败, 所以选了一个IP作为VIP), 其中keepalived和haproxy都使用容器化部署

部署高可用集群之前, 不要使用`kuberadm init`操作初始化集群, 因为要部署高可用集群, 需要先创建一个控制平面, 可见官网地址:  https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/high-availability/

## haproxy

在所有节点创建haproxy目录

```
mkdir -p /etc/haproxy/
```

然后在所有节点创建配置文件, 具体参数含义自行查询

```
cat <<EOF > /etc/haproxy/haproxy.cfg
global
  log 127.0.0.1 local0
  log 127.0.0.1 local1 notice
  tune.ssl.default-dh-param 2048

defaults
  log global
  mode http
  option dontlognull
  timeout connect 5000ms
  timeout client 600000ms
  timeout server 600000ms

listen stats
    bind :9090
    mode http
    balance
    stats uri /haproxy_stats
    stats auth admin:admin123
    stats admin if TRUE

frontend kube-apiserver-https
   mode tcp
   bind :8443
   default_backend kube-apiserver-backend

backend kube-apiserver-backend
    mode tcp
    balance roundrobin
    stick-table type ip size 200k expire 30m
    stick on src
    server apiserver1 172.16.104.171:6443 check
    server apiserver2 172.16.104.131:6443 check
    server apiserver3 172.16.104.132:6443 check
EOF
```

然后使用static pod把haproxy部署起来, 创建`/etc/kubernetes/manifests/haproxy.yaml`文件

```
cat <<EOF > /etc/kubernetes/manifests/haproxy.yaml
kind: Pod
apiVersion: v1
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
  labels:
    component: haproxy
    tier: control-plane
  name: kube-haproxy
  namespace: kube-system
spec:
  hostNetwork: true
  priorityClassName: system-cluster-critical
  containers:
  - name: kube-haproxy
    image: haproxy:1.9.7
    resources:
      requests:
        cpu: 100m
    volumeMounts:
    - name: haproxy-cfg
      readOnly: true
      mountPath: /usr/local/etc/haproxy/haproxy.cfg
  volumes:
  - name: haproxy-cfg
    hostPath:
      path: /etc/haproxy/haproxy.cfg
      type: FileOrCreate
EOF
```

## keepalivd

在所有节点创建yaml文件 `/etc/kubernetes/manifests/keepalived.yaml`

```
cat <<EOF > /etc/kubernetes/manifests/keepalived.yaml
kind: Pod
apiVersion: v1
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
  labels:
    component: keepalived
    tier: control-plane
  name: kube-keepalived
  namespace: kube-system
spec:
  hostNetwork: true
  priorityClassName: system-cluster-critical
  containers:
  - name: kube-keepalived
    image: osixia/keepalived:2.0.19
    env:
    - name: KEEPALIVED_VIRTUAL_IPS
      value: 172.16.104.200
    - name: KEEPALIVED_INTERFACE
      value: ens33
    - name: KEEPALIVED_UNICAST_PEERS
      value: "#PYTHON2BASH:['172.16.104.171', '172.16.104.131', '172.16.104.132']"
    - name: KEEPALIVED_PASSWORD
      value: docker
    - name: KEEPALIVED_PRIORITY
      value: "100"
    - name: KEEPALIVED_ROUTER_ID
      value: "51"
    resources:
      requests:
        cpu: 100m
    securityContext:
      privileged: true
      capabilities:
        add:
        - NET_ADMIN
EOF
```

> - KEEPALIVED_VIRTUAL_IPS：Keepalived 提供的 VIPs。
- KEEPALIVED_INTERFACE：VIPs 绑定的网卡。
- KEEPALIVED_UNICAST_PEERS：其他 Keepalived 节点单点传播的IP。
- KEEPALIVED_PASSWORD： Keepalived auth_type 的 Password。
- KEEPALIVED_PRIORITY：指定了VIP更换的优先级权重, 数字越小优先顺序越高, 主节点设置为100, 其他2个设置为150
- KEEPALIVED_ROUTER_ID：一组Keepalived instance 的数字识别号。

## 控制平面

使用kubeadm创建控制平面, 在主节点创建一个`kubeadm-config.yaml`文件

```
cat <<EOF > kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.17.0
imageRepository: registry.aliyuncs.com/google_containers
controlPlaneEndpoint: "172.16.104.200:8443"
networking:
  podSubnet: "10.244.0.0/16"
EOF
```

然后使用这个配置文件来初始化master节点 , 配置文件和直接命令执行一样的

```
kubeadm init --config=kubeadm-config.yaml --upload-certs

......
You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 172.16.104.200:8443 --token 5lzdvt.pipwh1bu0ju5gvxm \
    --discovery-token-ca-cert-hash sha256:e8d55f3d76cbddf9562170b36d15a47ba7ef33c6f35831333c164a9284767d7e \
    --control-plane --certificate-key 2ee1e36ebc5e18e6da8632993db232ff5e896df143914dc44edda36288746d20

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.16.104.200:8443 --token 5lzdvt.pipwh1bu0ju5gvxm \
    --discovery-token-ca-cert-hash sha256:e8d55f3d76cbddf9562170b36d15a47ba7ef33c6f35831333c164a9284767d7e
```

其他master节点加入, 之后可以看到

```
[root@vm1 kubernetes]# kubectl get nodes
NAME          STATUS   ROLES    AGE   VERSION
centos-kata   Ready    master   51m   v1.17.1
vm1           Ready    master   39m   v1.17.1
vm2           Ready    master   24m   v1.17.1


[root@centos-kata ~]# kubectl get pods --all-namespaces
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-5b644bc49c-j4w6c   1/1     Running   0          5h10m
kube-system   calico-node-48k8s                          1/1     Running   0          5h4m
kube-system   calico-node-9wstc                          1/1     Running   1          5h10m
kube-system   calico-node-bkdsz                          1/1     Running   0          5h10m
kube-system   coredns-9d85f5447-bvb9k                    1/1     Running   12         5h31m
kube-system   coredns-9d85f5447-tqhdq                    1/1     Running   1          5h31m
kube-system   etcd-centos-kata                           1/1     Running   8          5h31m
kube-system   etcd-vm1                                   1/1     Running   0          5h19m
kube-system   etcd-vm2                                   1/1     Running   0          5h4m
kube-system   kube-apiserver-centos-kata                 1/1     Running   9          5h31m
kube-system   kube-apiserver-vm1                         1/1     Running   0          5h19m
kube-system   kube-apiserver-vm2                         1/1     Running   0          5h4m
kube-system   kube-controller-manager-centos-kata        1/1     Running   3          5h31m
kube-system   kube-controller-manager-vm1                1/1     Running   1          5h19m
kube-system   kube-controller-manager-vm2                1/1     Running   1          5h4m
kube-system   kube-haproxy-centos-kata                   1/1     Running   1          5h31m
kube-system   kube-haproxy-vm1                           1/1     Running   0          47s
kube-system   kube-keepalived-centos-kata                1/1     Running   1          5h31m
kube-system   kube-keepalived-vm1                        1/1     Running   0          47s
kube-system   kube-proxy-c6zlc                           1/1     Running   0          5h19m
kube-system   kube-proxy-mk2tb                           1/1     Running   0          5h4m
kube-system   kube-proxy-qt6gx                           1/1     Running   1          5h31m
kube-system   kube-scheduler-centos-kata                 1/1     Running   3          5h31m
kube-system   kube-scheduler-vm1                         1/1     Running   2          5h19m
kube-system   kube-scheduler-vm2                         1/1     Running   0          5h4m

```

遗留问题:

- 另外两个master节点加入的时候, 发现提示manifest目录不为空, 需要把haproxy和keepalived的yaml文件删掉才能加入
- VIP不能选择节点本身实体的IP, 如果把VIP配置成node IP, 手动把另外两个master节点的haproxy和keepalived的yaml文件加入到manifest中, 会导致集群出问题, 提示"Error from server: etcdserver: request timed out"

## 验证

把第一个最开始创建keepalived和haproxy的节点poweroff关掉, 可以看到集群任然处于可用状态

```
[root@vm1 manifests]# kubectl get nodes
NAME          STATUS     ROLES    AGE     VERSION
centos-kata   NotReady   master   6h32m   v1.17.1
vm1           Ready      master   6h20m   v1.17.1
vm2           Ready      master   6h5m    v1.17.1
```

# 问题排查

可以使用`-v=10`来打印详细信息来定位, 比如使用

```
kubeadm join 172.16.104.200:8443 --token 5lzdvt.pipwh1bu0ju5gvxm \
    --discovery-token-ca-cert-hash sha256:e8d55f3d76cbddf9562170b36d15a47ba7ef33c6f35831333c164a9284767d7e -v=10
```

# RPM包安装Keepalived和Haproxy提供软负载

如果不想用静态Pod形式部署Keepalived和Haproxy, 可以用RPM包方式安装并提供服务, 方法和配置如下

每台master上安装对应软件

```
yum install -y keepalived haproxy
```

创建或者更改haproxy的配置文件,三个master上一样

```
cat <<EOF > /etc/haproxy/haproxy.cfg
global
  log 127.0.0.1 local0
  log 127.0.0.1 local1 notice
  tune.ssl.default-dh-param 2048
  maxconn 4000

defaults
  log global
  mode http
  option dontlognull
  timeout connect 5000ms
  timeout client 600000ms
  timeout server 600000ms

listen stats
    bind :9090
    mode http
    balance
    stats uri /haproxy_stats
    stats auth admin:admin123
    stats admin if TRUE

frontend kube-apiserver-https
   mode tcp
   bind :8443
   default_backend kube-apiserver-backend

backend kube-apiserver-backend
    mode tcp
    balance roundrobin
    stick-table type ip size 200k expire 30m
    stick on src
    server apiserver1 172.16.104.171:6443 check
    server apiserver2 172.16.104.131:6443 check
    server apiserver3 172.16.104.132:6443 check
EOF
```

然后启动haproxy  `systemctl start haproxy`

然后创建或者更改keepalived的配置文件 `/etc/keepalived/keepalived.conf`, 每台不一样,请注意, 这一步和容器化部署不一样

参数含义如下

```
global_defs {
    router_id centos-kata  ## 本节点标识，可使用主机名
}
#监测haproxy进程状态，健康检查，每2秒执行一次 
vrrp_script chk_haproxy {
    script "/etc/keepalived/chk_haproxy.sh" #监控脚本
    interval 2   #每两秒进行一次
    weight -10    #如果script中的指令执行失败，vrrp_instance的优先级会减少10个点
}
vrrp_instance VI_1 {
    state MASTER          #主服务器MASTER，从服务器为BACKUP
    interface ens33        #服务器固有IP（非VIP）的网卡
    virtual_router_id 51  #取值在0-255之间，用来区分多个instance的VRRP组播，同一网段中virtual_router_id的值不能重复，否则会出错。
    priority 100          #用来选举master的，要成为master，那么这个选项的值最好高于其他机器50个点。此时，从服务器要低于100；
    advert_int 1          #健康查检时间间隔
    mcast_src_ip 172.16.104.200    #MASTER服务器IP,从服务器写从服务器的IP
    authentication { #认证区域
        auth_type PASS  #推荐使用PASS（密码只识别前8位）
        auth_pass 12345678
    }
    track_script {
        chk_haproxy    #监测haproxy进程状态
    }
    virtual_ipaddress {
        172.16.104.200   #虚拟IP，作IP漂移使用
    }
}
```


master配置如下, 如果是其他节点, 则需要修改 router_id , priority, mcast_src_ip 这三项

```
global_defs {
    router_id centos-kata  
}

vrrp_script chk_haproxy {
    script "/etc/keepalived/chk_haproxy.sh" 
    interval 2
    weight -10
}
vrrp_instance VI_1 {
    state MASTER         
    interface ens33        
    virtual_router_id 51  
    priority 200         
    advert_int 1          
    mcast_src_ip 172.16.104.171    
    authentication { 
        auth_type PASS  
        auth_pass 12345678
    }
    track_script {
        chk_haproxy    
    }
    virtual_ipaddress {
        172.16.104.200 
    }
}
```

创建一个检测脚本, 并设置属性是可执行

```
vim /etc/keepalived/chk_haproxy.sh

if [ "${status}" = "0" ]; then
    systemctl start haproxy

    status2=$(ps aux|grep haproxy | grep -v grep | grep -v bash |wc -l)

    if [ "${status2}" = "0"  ]; then
            systemctl stop keepalived
    fi
fi
```

然后启动keepalived: `systemctl start keepalived`

验证, 用kubeadm创建集群, 可以看到多master创建成功

```
[root@centos-kata czs]# kubectl get nodes
NAME          STATUS   ROLES    AGE     VERSION
centos-kata   Ready    master   4m39s   v1.17.1
vm1           Ready    master   3m22s   v1.17.1

可以看到没有其他网络组件或者keepalived,haproxy的组件
[root@centos-kata czs]# kubectl get po -n kube-system
NAME                                  READY   STATUS              RESTARTS   AGE
coredns-9d85f5447-dmv45               0/1     ContainerCreating   0          6m9s
coredns-9d85f5447-dwlxb               0/1     ContainerCreating   0          6m9s
etcd-centos-kata                      1/1     Running             0          6m4s
etcd-vm1                              1/1     Running             0          5m9s
kube-apiserver-centos-kata            1/1     Running             0          6m4s
kube-apiserver-vm1                    1/1     Running             0          5m10s
kube-controller-manager-centos-kata   1/1     Running             1          6m4s
kube-controller-manager-vm1           1/1     Running             0          5m10s
kube-proxy-gb6d6                      1/1     Running             0          5m11s
kube-proxy-l9zb7                      1/1     Running             0          6m9s
kube-scheduler-centos-kata            1/1     Running             1          6m4s
kube-scheduler-vm1                    1/1     Running             0          5m9s
```

# 查看Etcd数据

kubeadm 创建的集群, etcd 默认使用 tls ，这时你可以在 master 节点上使用以下命令来访问 etcd

```
ETCDCTL_API=3 etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/peer.crt --key=/etc/kubernetes/pki/etcd/peer.key get /registry/namespaces/default -w=json|python -m json.tool
```

使用`--prefix`可以看到所有的子目录

可以使用下面脚本来查看Kubernetes元数据

```
#!/bin/bash
# Get kubernetes keys from etcd
export ETCDCTL_API=3
keys=`etcdctl get /registry --prefix -w json|python -m json.tool|grep key|cut -d ":" -f2|tr -d '"'|tr -d ","`
for x in $keys;do
  echo $x|base64 -d|sort
done
```





