# Kata初识

Kata是安全容器, 原理是起一层薄虚拟机, 加强和宿主机的隔离性

![](https://github.com/chenzongshu/Kubernetes/blob/master/pic/kata-explained1.png?raw=true)

kata 不直接与 k8s 通信，因为对于 k8s 来说，它只跟实现了 CRI 接口的容器管理进程打交道，比如 docker-engine，rkt, containerd(使用 cri plugin) 或 CRI-O，而 kata 跟 runc 是同一个级别的进程。所以如果要与 k8s 集成，则需要安装 CRI-O 或 CRI-containerd 来支持 CRI 接口，本文使用 CRI-O。CRI-O 的 O 的意思是 OCI-compatible，即 CRI-O 是实现 CRI 接口来跑 OCI 容器的。

# 安装kata

在CentOS上, 通过yum安装kata

```
$ source /etc/os-release
$ sudo yum -y install yum-utils
$ ARCH=$(arch)
$ BRANCH="${BRANCH:-master}"
$ sudo -E yum-config-manager --add-repo "http://download.opensuse.org/repositories/home:/katacontainers:/releases:/${ARCH}:/${BRANCH}/CentOS_${VERSION_ID}/home:katacontainers:releases:${ARCH}:${BRANCH}.repo"
$ sudo -E yum -y install kata-runtime kata-proxy kata-shim
```

验证安装：

```
$ kata-runtime --version
kata-runtime  : 1.3.1
   commit   : 258eae0
   OCI specs: 1.0.1
```

如果你是在虚拟机里安装 kata，最好执行下面的命令检测是否成功安装。因为 kata 实际上会创建虚拟机，所以要求安装 kata 的主机开启 nested virtualization：

```
$ kata-runtime kata-check
```

# 安装CRI-O

```
modprobe overlay
modprobe br_netfilter

# Setup required sysctl params, these persist across reboots.
cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system

# Install prerequisites
yum-config-manager --add-repo=https://cbs.centos.org/repos/paas7-crio-311-candidate/x86_64/os/

# Install CRI-O
yum install --nogpgcheck cri-o
```

然后启动CRI-O

```
systemctl start crio
```

