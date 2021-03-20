著名的 K8S CNI 网络方案calico在2020.02宣布将于v3.13版本支持eBPF：[calico-eBPF support](https://www.projectcalico.org/introducing-the-calico-ebpf-dataplane/) ，2020.08，谷歌宣布Cilium（第一个使用eBPF的网络插件）作为GKE的下一代数据平面

我们也跟着大佬的脚步，来看看eBPF和cilium

# eBPF

## BPF

BPF（Berkeley Packet Filter ），中文翻译为伯克利包过滤器，是类 Unix 系统上数据链路层的一种原始接口，提供原始链路层封包的收发。

## eBPF

2014 年初，Alexei Starovoitov 实现了 eBPF（extended Berkeley Packet Filter）。经过重新设计，eBPF 演进为一个通用执行引擎，可基于此开发性能分析工具、软件定义网络等诸多场景。eBPF 最早出现在 3.18 内核中（该版本功能较少，要使用完整的 eBPF，需要 Linux 4.4 或以上），此后原来的 BPF 就被称为经典 BPF，缩写 cBPF（classic BPF），cBPF 现在已经基本废弃。现在，Linux 内核只运行 eBPF，内核会将加载的 cBPF 字节码透明地转换成 eBPF 再执行。

eBPF将原先的 BPF 发展成一个指令集更复杂、应用范围更广的“内核虚拟机”。

eBPF 支持在用户态将 C 语言编写的一小段“内核代码”注入到内核中运行，注入时要先用 llvm 编译得到使用 BPF 指令集的 ELF 文件，然后从 ELF 文件中解析出可以注入内核的部分，最后用 bpf_load_program()方法完成注入。 用户态程序和注入到内核中的程序通过共用一个位于内核中的 eBPF MAP 实现通信。为了防止注入的代码导致内核崩溃，eBPF 会对注入的代码进行严格检查，拒绝不合格的代码的注入。

相对于系统的性能分析和观测，eBPF 技术在网络技术中的表现，更是让人眼前一亮，BPF 技术与 XDP（eXpress Data Path） 和 TC（Traffic Control） 组合可以实现功能更加强大的网络功能，更可为 SDN 软件定义网络提供基础支撑。XDP 只作用与网络包的 Ingress 层面，BPF 钩子位于**网络驱动中尽可能早的位置**，**无需进行原始包的复制**就可以实现最佳的数据包处理性能，挂载的 BPF 程序是运行过滤的理想选择，可用于丢弃恶意或非预期的流量、进行 DDOS 攻击保护等场景；而 TC Ingress 比 XDP 技术处于更高层次的位置，BPF 程序在 L3 层之前运行，可以访问到与数据包相关的大部分元数据，是本地节点处理的理想的地方，可以用于流量监控或者 L3/L4 的端点策略控制，同时配合 TC egress 则可实现对于容器环境下更高维度和级别的网络结构。

### eBPF机制

#### 事件和钩子

eBPF 程序是在内核中被事件触发的。在一些特定的指令被执行时时，这些事件会在钩子处被捕获。钩子被触发就会执行 eBPF 程序，对数据进行捕获和操作。钩子定位的多样性正是 eBPF 的闪光点之一。例如下面几种：

- 系统调用：当用户空间程序通过系统调用执行内核功能时。
- 功能的进入和退出：在函数退出之前拦截调用。
- 网络事件：当接收到数据包时。
- kprobe 和 uprobe：挂接到内核或用户函数中。

#### 辅助函数

eBPF 程序被触发时，会调用辅助函数。这些特别的函数让 eBPF 能够有访问内存的丰富功能。例如 Helper 能够执行一系列的任务：

- 在数据表中对键值对进行搜索、更新以及删除。
- 生成伪随机数。
- 搜集和标记隧道元数据。
- 把 eBPF 程序连接起来，这个功能被称为 `tail call`。
- 执行 Socket 相关任务，例如绑定、获取 Cookie、数据包重定向等。

这些helper函数必须是内核定义的，换句话说，eBPF 程序的调用能力是受到一个白名单限制的。这个[名单](https://man7.org/linux/man-pages/man7/bpf-helpers.7.html)很长，并且还在持续增长之中。

```
┌──────────────────────────────┐                  ┌──────────────────────────────┐
│User                          │                  │kernel        ┌────────┐      │
│┌─────────┐       ┌─────────┐ │                  │  ┌────────┐  │        │      │
││  eBPF   │ LLVM  │  eBPF   │ │         ┌────────┼─▶│Verifier│──┘        ▼      │
││programe │──────▶│bytecode │─┼─────────┘        │  └────────┘      ┌─────────┐ │
│└─────────┘       └─────────┘ │                  │                  │   JIT   │ │
│                              │                  │                  │Compiler │ │
│                              │                  │                  └─────────┘ │
│          ┌────────────────┐  │                  │  ┌─────────┐          │      │
│          │ per-event data │◀─┼──────────────────┼──│   BPF   │◀─────────┘      │
│          └────────────────┘  │                  │  └─────────┘                 │
│                              │                  │                              │
│          ┌────────────────┐  │                  │  ┌─────────┐                 │
│          │   statistics   │◀─┼──────────────────┼──│  maps   │                 │
│          └────────────────┘  │                  │  └─────────┘                 │
│                              │                  │                              │
└──────────────────────────────┘                  └──────────────────────────────┘
```

要在 eBPF 程序和内核以及用户空间之间存储和共享数据，eBPF 需要使用 Map。正如其名，Map 是一种键值对。Map 能够支持多种数据结构，eBPF 程序能够通过辅助函数在 Map 中发送和接收数据。

#### 加载和校验

所有 eBPF 程序都是以字节码的形式执行的，因此需要有办法把高级语言编译成这种字节码。eBPF 使用 [LLVM](https://llvm.org/) 作为后端，前端可以介入任何语言。因为 eBPF 使用 C 编写的，所以前端使用的是 [Clang](https://clang.llvm.org/)。但在字节码被 Hook 之前，必须通过一系列的检查。在一个类似虚拟机的环境下用[内核 Verifier](https://elixir.bootlin.com/linux/latest/source/kernel/bpf/verifier.c)阻止带有循环、权限不正确或者导致崩溃的程序运行。如果程序通过了所有的检查，字节码会使用 `bpf()` 系统调用被载入到 Hook 上。

#### JIT 编译器

校验结束后，eBPF 字节码会被 JIT 编译器转译成本地机器码。eBPF 是 64 位编码，共有 11 个寄存器，因此 eBPF 和 x86、ARM 以及 arm64 等硬件都能紧密对接。虽然 eBPF 受到 VM 限制，JIT 过程保障了它的运行性能

#### 总结

eBPF 的工作流程：

- 把 eBPF 程序编译成字节码。
- 在载入到 Hook 之前，在虚拟机中对程序进行校验。
- 把程序附加到内核之中，被特定事件触发。
- JIT 编译。
- 在程序被触发时，调用辅助函数处理数据。
- 在用户空间和内核空间之间使用键值对共享数据。

# Cilium



## 安装

快速安装

```
kubectl create -f https://raw.githubusercontent.com/cilium/cilium/v1.9/install/kubernetes/quick-install.yaml
```

你也可以通过helm来安装，安装完成可以看到Pod已经启动，一个Operator，一个daemonset部署的agent

```
[root@k8s-m1 ~]# kubectl get po --all-namespaces
NAMESPACE     NAME                               READY   STATUS              RESTARTS   AGE
kube-system   cilium-9fx6z                       1/1     Running             0          2m36s
kube-system   cilium-operator-67895d78b7-xr929   1/1     Running             0          30m
kube-system   cilium-rqhc9                       1/1     Running             0          2m36s
```

## 验证

Cilium自己提供了一套验证服务的部署，但是至少需要两个节点，不然有些部署不成功

可以单独创建一个ns来部署

```bash
kubectl create ns cilium-testv

kubectl apply -n cilium-test -f https://raw.githubusercontent.com/cilium/cilium/v1.9/examples/kubernetes/connectivity-check/connectivity-check.yaml
```

如果成功，会有下面Pod

```
kubectl get pods -n cilium-test
NAME                                                     READY   STATUS    RESTARTS   AGE
echo-a-76c5d9bd76-q8d99                                  1/1     Running   0          66s
echo-b-795c4b4f76-9wrrx                                  1/1     Running   0          66s
echo-b-host-6b7fc94b7c-xtsff                             1/1     Running   0          66s
host-to-b-multi-node-clusterip-85476cd779-bpg4b          1/1     Running   0          66s
host-to-b-multi-node-headless-dc6c44cb5-8jdz8            1/1     Running   0          65s
pod-to-a-79546bc469-rl2qq                                1/1     Running   0          66s
pod-to-a-allowed-cnp-58b7f7fb8f-lkq7p                    1/1     Running   0          66s
pod-to-a-denied-cnp-6967cb6f7f-7h9fn                     1/1     Running   0          66s
pod-to-b-intra-node-nodeport-9b487cf89-6ptrt             1/1     Running   0          65s
pod-to-b-multi-node-clusterip-7db5dfdcf7-jkjpw           1/1     Running   0          66s
pod-to-b-multi-node-headless-7d44b85d69-mtscc            1/1     Running   0          66s
pod-to-b-multi-node-nodeport-7ffc76db7c-rrw82            1/1     Running   0          65s
pod-to-external-1111-d56f47579-d79dz                     1/1     Running   0          66s
pod-to-external-fqdn-allow-google-cnp-78986f4bcf-btjn7   1/1     Running   0          66s
```

# Hubble

 `Cilium` 强大之处就是提供了简单高效的网络可视化功能，它是通过 **Hubble**组件完成的。它是专门为网络可视化设计，能够利用 `Cilium` 提供的 `eBPF` 数据路径，获得对 Kubernetes 应用和服务的网络流量的深度可见性。这些网络流量信息可以对接 `Hubble CLI`、UI 工具，可以通过交互式的方式快速诊断如与 DNS 相关的问题。除了 Hubble 自身的监控工具，还可以对接主流的云原生监控体系 —— `Prometheus` 和 `Grafana`，实现可扩展的监控策略

## 安装

如果是1.9.2以上版本

```bash
kubectl apply -f https://raw.githubusercontent.com/cilium/cilium/1.9.5/install/kubernetes/quick-hubble-install.yaml
```

使用NodePort暴露`Hubble`服务

```
$ kubectl get pod -n kube-system |grep hubble
hubble-ui-5f9fc85849-hkzkr         1/1     Running   0          15h
hubble-vpxcb                       1/1     Running   0          21h
```

# Cilium原理

Cilium 数据保存在Etcd中，还支持consul

Cilium Agent作为daemonset的特权形式运行在每个节点上，功能：

- 暴露 API 给运维和安全团队，可以配置容器间的通信策略。还可以通过这些 API 获取网络监控数据。
- 收集容器的元数据，例如 Pod 的 Label，可用于 Cilium 安全策略里的 Endpoint 识别，这个跟 Kubernetes 中的 service 里的 Endpoint 类似。
- 与容器管理平台的网络插件交互，实现 IPAM 的功能，用于给容器分配 IP 地址，该功能与 [flannel](https://jimmysong.io/kubernetes-handbook/concepts/flannel.html)、[calico](https://jimmysong.io/kubernetes-handbook/concepts/calico.html) 网络插件类似。
- 将其有关容器标识和地址的知识与已配置的安全性和可见性策略相结合，生成高效的 BPF 程序，用于控制容器的网络转发和安全行为。
- 使用 clang/LLVM 将 BPF 程序编译为字节码，在容器的虚拟以太网设备中的所有数据包上执行，并将它们传递给 Linux 内核。

## 组网模式

默认是基于vxlan的overlay组网模式，除此之外，还包括

- 通过BGP路由的方式，实现集群间Pod的组网和互联；

- 在AWS的ENI（Elastic Network Interfaces）模式下部署使用Cilium；

- Flannel和Cilium的集成部署；

- 采用基于ipvlan的组网，而不是默认的基于veth；

- Cluster Mesh组网，实现跨多个Kubernetes集群的网络连通和安全性

## Overlay组网原理

安装完成之后，可以看到宿主机多了四个虚拟网络接口

```
6: cilium_net@cilium_host: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 56:fb:10:ce:74:bf brd ff:ff:ff:ff:ff:ff
    inet6 fe80::54fb:10ff:fece:74bf/64 scope link
       valid_lft forever preferred_lft forever
7: cilium_host@cilium_net: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 2a:b3:8e:61:05:ec brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.148/32 scope link cilium_host
       valid_lft forever preferred_lft forever
    inet6 fe80::28b3:8eff:fe61:5ec/64 scope link
       valid_lft forever preferred_lft forever
8: cilium_vxlan: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ether f2:ea:e0:84:fe:a9 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::f0ea:e0ff:fe84:fea9/64 scope link
       valid_lft forever preferred_lft forever
10: lxc_health@if9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 72:16:f5:52:35:cc brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::7016:f5ff:fe52:35cc/64 scope link
       valid_lft forever preferred_lft forever
```

- cilium_vxlan主要是处理对数据包的vxlan隧道操作，采用metadata模式，并不会为这个接口分配ip地址

- cilium_host作为主机上该子网的一个网关，并且在node上为其自动分配了ip地址10.0.0.148/32，

- cilium_net和cilium_host作为一对veth而创建

- 还有一个lxc_health

进入其中一个Agent Pod，使用命令 `cilium bpf tunnel list` 查看隧道

```
root@k8s-m1:/home/cilium# cilium bpf tunnel list
TUNNEL       VALUE
10.0.1.0:0   192.168.51.201:0
```

可以看到给另外一个节点（192.168.51.201）上的虚拟网络（10.0.1.0:0）创建了一个隧道；同理另外一个节点上面也创建了一个隧道

```
root@k8s-w1:/home/cilium# cilium bpf tunnel list
TUNNEL       VALUE
10.0.0.0:0   192.168.51.200:0
```

在两个节点上分别启动2个Pod

```
[root@k8s-m1 ~]# kubectl get po -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
centos-m-58fc9f6c8d-kcq87   1/1     Running   0          68s   10.0.0.157   k8s-m1   <none>           <none>
centos-m-58fc9f6c8d-mgzg5   1/1     Running   0          68s   10.0.0.243   k8s-m1   <none>           <none>
centos-w-6976fb747d-8jddt   1/1     Running   0          3s    10.0.1.206   k8s-w1   <none>           <none>
centos-w-6976fb747d-p75hv   1/1     Running   0          3s    10.0.1.58    k8s-w1   <none>           <none>
```

### 同节点

可以看到没启动一个Pod，就会在宿主机上启动一个类似

```bash
ip a
......
28: lxcaa7d281d4068@if27: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 56:3a:84:36:02:63 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::543a:84ff:fe36:263/64 scope link
```

容器内部可以看到对应的veth pair

```
[root@k8s-w1 ~]# kubectl exec -it centos-w-6976fb747d-8jddt ip a
······
27: eth0@if28: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
······
```

由此知道，同节点的Pod通信可以经过Veth Pair，由