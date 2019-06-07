
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

## 添加https支持

```
apt install -y apt-transport-https curl
```

## 安装docker

添加docker repository和docker GPG key

```
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | apt-key add -

add-apt-repository \
   "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

然后安装docker

```
apt-get install docker-ce
```

## 安装kubeadm, kubelet 和 kubectl

添加apt-key

```
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
```

添加kubernetes源

```
sudo vim /etc/apt/sources.list.d/kubernetes.list

deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
```

安装

```
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

# 使用kubeadm创建一个单Master集群

- 1.选择一个网络插件，并检查它是否需要在初始化Master时指定一些参数，比如我们可能需要根据选择的插件来设置`--pod-network-cidr`参数。

- 2.kubeadm使用eth0的默认网络接口（通常是内网IP）做为Master节点的advertise address，如果我们想使用不同的网络接口，可以使用`--apiserver-advertise-address=<ip-address>`参数来设置。如果适应IPv6，则必须使用IPv6d的地址，如：`--apiserver-advertise-address=fd00::101`。

- 3.使用`kubeadm config images pull`来预先拉取初始化需要用到的镜像，用来检查是否能连接到Kubenetes的Registries。

Kubenetes默认Registries地址是`k8s.gcr.io`，很明显，在国内并不能访问gcr.io，因此在kubeadm v1.13之前的版本，安装起来非常麻烦，但是在1.13版本中终于解决了国内的痛点，其增加了一个`--image-repository`参数，默认值是`k8s.gcr.io`，我们将其指定为国内镜像地址：`registry.aliyuncs.com/google_containers`，其它的就可以完全按照官方文档来愉快的玩耍了。

其次，我们还需要指定`--kubernetes-version`参数，因为它的默认值是`stable-1`，会导致从`https://dl.k8s.io/release/stable-1.txt`下载最新的版本号，我们可以将其指定为固定版本（最新版：v1.13.1）来跳过网络请求。

然后创建

```
root@czs-virtual-machine:~# kubeadm init --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.14.0 --pod-network-cidr=192.168.0.0/16
[init] Using Kubernetes version: v1.14.0
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

为了使用kubectl访问apiserver, 需要增加环境变量

```
vim .profile

export KUBECONFIG=/etc/kubernetes/admin.conf

source .profile
```

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

## 安装calico

默认情况下，`Calico`网络插件使用的的网段是`192.168.0.0/16`，在init的时候，我们已经通过`--pod-network-cidr=192.168.0.0/16`来适配Calico，当然你也可以修改`calico.yml`文件来指定不同的网段。

国内镜像源来拉calico镜像

```
kubectl apply -f http://mirror.faasx.com/k8s/calico/v3.3.2/rbac-kdd.yaml
kubectl apply -f http://mirror.faasx.com/k8s/calico/v3.3.2/calico.yaml
```

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

# CentOS配置差异

上面使用的是ubuntu, 如果使用CentOS, 有一些差异化的配置

配置转发参数

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

在`kubeadm init`之后

服务启动后需要根据输出提示，进行配置：

```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

然后即可以安装网络插件来拉起



