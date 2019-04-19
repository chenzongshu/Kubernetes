# kata container简介

标签（空格分隔）： kubernetes

---

# kata简介

kata containers是由OpenStack基金会管理的容器项目, 号称提供更高的安全性, 其实是以前两个项目的合并, `runV`和`Clear Containers`

其github地址如下
https://github.com/kata-containers/

## 架构

kata container实质上是在虚拟机内部使用container（基于runc的实现）。 kata-container使用虚拟化软件(qemu-lite优化过的qemu)， 通过已经将kata-agent 安装的kernel & intrd image，启动过一个轻量级的虚拟机， 使用nvdimm将initrd image映射到guest vm中。然后由kata-agent为container创建对应的namespace和资源。 Guest VM作为实质上的sandbox可以完全与host kernel进行隔离。

目前由多个组件组成

- `runtime`：实现OCI接口，可以通过CRI-O 与kubelet对接作为k8s runtime server， containerd对接docker engine，创建运行container/pod的VM

- `agent`：运行在guest vm中的进程， 主要依赖于libcontainer项目，重用了大部分的runc代码，为container创建namespace(NS, UTS, IPC and PID)

- `shim`：以对接docker为例，这里的shim相当于是containerd-shim的一个适配，用来处理容器 进程的stdio和signals。shim可以将containerd-shim发来的数据流（如stdin）传给proxy，然 后转交给agent，也可以将agent经由proxy发过来的数据流（stdout/stderr）传给containerdshim，同时也可以传输signal。

- `proxy`：每一个container都会由一个kata-proxy进程，kata-proxy负责与kata-agent通讯，当guest vm启动后，kata-agent会随之启动并使用qemu virtio serial console 进行通讯。 

- `kernel`：kernel其实比较好理解，就是提供一个轻量化虚机的linux内核，根据不同的需要，提供 几个内核选择，最小的内核仅有4M多。

## k8s和kata

kata container是hypverisor container阵营的container runtime项目，支持OCI标准。

kata 不直接与 k8s 通信，因为对于 k8s 来说，它只跟实现了 CRI 接口的容器管理进程打交道，比如 docker-engine，rkt, containerd(使用 cri plugin) 或 CRI-O

k8s孵化项目CRI-O就是可以提供CRI并能够与满足OCI container runtime通讯的项目 k8s与kata container的work flow 如下

```
                                +---------------+                      +--->|container  |
+---------------+               |  cri-o        |                      |    +-----------+
|  kubelet      |               |               |     +-------------+  |
| +-------------+ cri protobuf  +-------------+ |<--->|  container  +<-+    +-----------+
| | grpc client |<------------->| grpc server | |     |  runtime    +<----->|container  |
| +-------------|               +-------------+ |     +-------------+       +-----------+
+---------------+               |               |
                                +---------------+

```

# kata安装包

**注意, 如果你和我一样是使用的虚拟机, 一定要打开虚拟化功能**

在centos环境之下, 安装kata包

```
$ source /etc/os-release
$ sudo yum -y install yum-utils
$ ARCH=$(arch)
$ sudo -E yum-config-manager --add-repo "http://download.opensuse.org/repositories/home:/katacontainers:/releases:/${ARCH}:/master/CentOS_${VERSION_ID}/home:katacontainers:releases:${ARCH}:master.repo"
$ sudo -E yum -y install kata-runtime kata-proxy kata-shim
```

会安装kata及对应的依赖包

```
已安装:
  kata-proxy.x86_64 0:1.3.0+git.6ddb006-9.1 kata-runtime.x86_64 0:1.3.0+git.a786643-14.1 kata-shim.x86_64 0:1.3.0+git.5fbf1f0-8.1

作为依赖被安装:
  kata-containers-image.x86_64 0:1.3.0-9.1                         kata-ksm-throttler.x86_64 0:1.3.0.git+6e903fb-11.1
  kata-linux-container.x86_64 0:4.14.67.12-10.1                    kata-proxy-bin.x86_64 0:1.3.0+git.6ddb006-9.1
  kata-shim-bin.x86_64 0:1.3.0+git.5fbf1f0-8.1                     qemu-lite.x86_64 0:2.11.0+git.f886228056-12.1
  qemu-lite-bin.x86_64 0:2.11.0+git.f886228056-12.1                qemu-lite-data.x86_64 0:2.11.0+git.f886228056-12.1
  qemu-vanilla.x86_64 0:2.11.2+git.0982a56a55-12.1                 qemu-vanilla-bin.x86_64 0:2.11.2+git.0982a56a55-12.1
  qemu-vanilla-data.x86_64 0:2.11.2+git.0982a56a55-12.1
```

安装完检查运行环境是否满足要求

```
kata-runtime kata-check
```

不能有error, 如果这项有报错请检测是否打开了虚拟化功能

```INFO[0000] CPU property found arch=amd64 description="Virtualization support" name=vmx pid=3595 source=runtime type=flag```

# Docker配置

```
$ sudo mkdir -p /etc/systemd/system/docker.service.d/
$ cat <<EOF | sudo tee /etc/systemd/system/docker.service.d/kata-containers.conf
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -D --add-runtime kata-runtime=/usr/bin/kata-runtime --default-runtime=kata-runtime
EOF
```

然后重启docker

```
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```

# 验证

```
[root@centos2 ~]# docker run busybox uname -a
Linux f79ec594806e 4.14.67-10.1.container #1 SMP Fri Sep 28 02:55:25 UTC 2018 x86_64 GNU/Linux
[root@centos2 ~]#
[root@centos2 ~]# uname -a
Linux centos2 3.10.0-862.el7.x86_64 #1 SMP Fri Apr 20 16:44:24 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
[root@centos2 ~]#
```

可以看到, 和宿主机的内核有所不同

