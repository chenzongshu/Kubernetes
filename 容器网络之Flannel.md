# tun/tap设备

什么是 tun/tap设备, 从网络虚拟化的角度来说, 它是虚拟网卡, 一端连着网络协议栈, 一端连着用户态程序. TUN 设备的功能非常简单，即：**在操作系统内核和用户应用程序之间传递 IP 包**

但是他们还是有些区别的, tun表示虚拟的是点对点设备, tap表示虚拟的以太网设备, 这两种设备针对的网络包实施不同的封装

tun/tap设备可以将TCP/IP协议栈处理好的网络包发给任何一个使用tun/tap驱动的进程, 由进程重新处理后发到物理链路中. tun/tap设备就像埋在用户程序空间的一个钩子, 我们可以很方便的将对网络包的处理程序挂在这个钩子上

从网络协议栈的角度看, tun/tap设备这类虚拟网卡和物理网卡没有区别, 只是对tun/tap设备而言, 它与物理网卡的不同之处在于**它的数据源不是物理链路, 而是来自于用户态!**, 所哦以tun/tap设备就是利用Linux的设备文件实现内核态和用户态数据交换

普通物理网卡是通过网线收发数据包, 而tun设备通过一个设备文件(/dev/tunX)收发数据包, 所有对这个文件的写操作都会通过tun设备转换成一个数据包发给内核网络协议栈, 当内核发一个包给tun设备的时候, 用户态进程可以读取这个文件拿到包内容.

tun和tap设备区别:

- tun设备的 /dev/tunX 文件收发的是IP包, 因此只能工作在L3, 无法和物理网卡做桥接, 但可以通过三层交换,(例如 ip_forward)与物理网卡连通;
- tap设备的/dev/tapX 文件收发的是链路层数据包, 可以与物理网卡桥接



# VxLan

下面简要的以Flannel说明下Vxlan模式下怎么把数据包转发出去的

1. Flannel在启动的时候会给每个机器启动一个VTEP设备, 名字是flannel.1, 它既有 IP 地址，也有 MAC 地址

2. 从PodA的IP x.x.15.A 访问PodB的IP x.x.16.B的时候, 先到docker0网桥, 然后被路由到flannel.1, 也就是到了"隧道"的入口

怎么找到PodB锁在节点的VTEP设备呢? 原来NodeB的flannel在启动的时候, 就会给每个节点增加一条路由:

```
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
...
x.x.16.0       x.x.16.0       255.255.255.0   UG    0      0        0 flannel.1
```

这条规则的意思是：凡是发往 x.x.16.0/24 网段的 IP 包，都需要经过 flannel.1 设备发出，并且，它最后被发往的网关地址是：x.x.16.0。而x.x.16.0正是 NodeB 上的 VTEP 设备（也就是 flannel.1 设备）的 IP 地址

3. VXLAN数据包会加上 `VXLAN头(VNI)`+`目的VTEP设备的MAC地址`+`目的容器的IP地址`的头, 上一步我们只知道目的VTEP的IP地址, 这个时候就需要从ARP表里面获取对应的MAC地址, 对应的记录也是NodeB启动的时候Flannel进程添加到每个节点的ARP表里面的, 可以用`ip neigh show dev flannel.1`命令来查看

4. 通过UDP协议发包

   实际上现在源端只知道对端的VTEP设备的MAC和IP地址, 不知道节点的IP地址, 这个时候就需要flannel.1设备扮演网桥的角色, 在二层网络进行 UDP 包的转发, Linux内核通过FDB（Forwarding Database）的转发数据库, 来知道对应的Node IP

   ```
   # 在 Node A 上，使用“目的 VTEP 设备”的 MAC 地址进行查询
   $ bridge fdb show flannel.1 | grep 5e:f8:4f:00:e3:37
   5e:f8:4f:00:e3:37 dev flannel.1 dst 10.168.0.3 self permanent
   ```

   可以看到，在上面这条 FDB 记录里，指定了这样一条规则，即：

   发往我们前面提到的“目的 VTEP 设备”（MAC 地址是 5e:f8:4f:00:e3:37）的二层数据帧，应该通过 flannel.1 设备，发往 IP 地址为 10.168.0.3 的主机。显然，这台主机正是 Node B，UDP 包要发往的目的地就找到了。

5. 然后最外层还需要封装上目的节点的IP和MAC, 成为普通的UDP包, 发到对端. 对端经过网络协议栈的拆包, 发现VXLAN头, 并且VNI为1, 转给flannel.1(因为VNI为1, 所有所有的VTEP设备都叫flannel.1), 然后flannel.1进一步拆包, 发到容器的network namesapce中

#  