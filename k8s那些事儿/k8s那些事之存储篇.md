# 概述

Kubernetes存储体系是围绕着PV/PVC资源打造的，支持各种主流的存储卷

通过下面命令可以看到支持的卷

```
kubectl explain pod.spec.volumes
```

另外也提供插件机制，允许其他类型的存储服务接入到 Kubernetes 系统中来

Kubernetes存储插件有两种模式：`In-Tree` 和 `Out-Of-Tree`。`In-Tree` 就是在 Kubernetes 源码内部实现的，和 Kubernetes 一起发布、管理的，但是更新迭代慢、灵活性比较差，`Out-Of-Tree` 是独立于 Kubernetes 的，目前主要有 `CSI` 和 `FlexVolume` 两种机制，

# PV、PVC、StorageClass

这一套，基本上是Kubernetes最基本的存储用法

```
┌─────────────┐                        
│     Pod     │                        
└─────────────┘                        
       │                               
       ▼                               
┌─────────────┐                        
│     PVC     │────Check───────┐       
└─────────────┘                ▼       
       │                ┌─────────────┐
       │                │StorageClass │
     Bind               ├─────────────┤
       │                │ Provisioner │
       ▼                └─────────────┘
┌─────────────┐                │       
│     PV      │◀───Provision───┘       
└─────────────┘                        
```

- PVC 描述的，是 Pod 想要使用的持久化存储的属性，比如存储的大小、读写权限等。

- PV 描述的，则是一个具体的 Volume 的属性，比如 Volume 的类型、挂载目录、远程存储 服务器地址等。

- 而 StorageClass 的作用，则是充当 PV 的模板。并且，只有同属于一个 StorageClass 的 PV 和 PVC，才可以绑定在一起。当然不用StorageClass也是可以的，但是生产上如果集群有成千上万PVC的时候，不可能手动去创建PV。

# FlexVolume

flexvolume是一个二进制或者shell脚本，放在每个节点上，来实现 FlexVolume 的相关接口，其操作参数和执行参数都是固定的。

默认存储插件的存放路径为`/usr/libexec/kubernetes/kubelet-plugins/volume/exec/<vendor~driver>/<driver>`，`VolumePlugins` 组件会不断 watch 这个目录来实现插件的添加、删除等功能。

其中 `vendor~driver` 的名字需要和 Pod 中`flexVolume.driver` 的字段名字匹配

并不一定所有接口都需要实现，比如NFS这样的存储就没必要实现 `attach/detach` 这些接口了，因为不需要，只需要实现 `init/mount/umount` 3个接口即可，官方提供了一个简单的示例：https://github.com/kubernetes/examples/blob/master/staging/volumes/flexvolume/nfs

**flexvolume的本质是相当于就是一个普通的 shell 命令**，类似于平时我们在 Linux 下面执行的 `ls` 命令一样，只是返回的信息是 JSON 格式的数据，并不是我们通常认为的一个常驻内存的进程

因为其本质，决定了flexvolume即不能Dynamic Provisioning（为每个 PVC 自动创建 PV 和对应的 Volume），又不能保存挂载信息等，所以有了CSI这种更完善的存储插件体系

# CSI

Kubernetes内置的存储插件和Flexvolume，仅仅是去操作存储卷，而Dynamic Provisioning这样的功能，需要去监控Kubernetes里面存储事件变化，如PVC创建，来做下一步的逻辑，这部分功能原来是Kubernetes存储管理的一部分。

CSI的设计思想就是把这个 Provision 阶段，以及 Kubernetes 里的一部分存储管理功能，从主干代码里剥离出来，做成了几个单独的组件。把插件的职责从“两阶段处理”，扩展成了 Provision、Attach 和 Mount 三个阶段。其中，Provision 等价于“创建磁盘”，Attach 等价 于“挂载磁盘到虚拟机”，Mount 等价于“将该磁盘格式化后，挂载在 Volume 的宿主机目录上”。

```
                                ┌─────────────────────────────────────────────────┐     
                                │                                                 │     
                                ▼                                                 │     
                     ┌────────────────────┐         ┌───────────────┐        ┌─────────┐
                     │  Driver Registrar  │────────▶│ CSI Identity  │        │kubelete │
                     └────────────────────┘         └───────────────┘        └─────────┘
                                                                                  │     
                                                                                  │     
┌──────┐             ┌────────────────────┐         ┌───────────────┐             │     
│      │      ┌─────▶│External Provisioner│────┬───▶│CSI Controller │             │     
│      │    PVC      └────────────────────┘    │    └───────────────┘             │     
│ API  │      │                                │                                  │     
│Server│──────┤                                │                                  │     
│      │ VolumeAtta                            │                                  │     
│      │   chment    ┌────────────────────┐    │    ┌───────────────┐             │     
└──────┘      └─────▶│ External Attacher  │────┘    │   CSI Node    │◀────────────┘     
                     └────────────────────┘         └───────────────┘                   
                                                            │                           
                                                            │                           
                                                            ▼                           
                                                    ┌───────────────┐                   
                                                    │AWS/NFS/Ceph...│                   
                                                    └───────────────┘                   
```

CSI插件可以是一个二进制或者容器，提供三个服务：CSI Identity，CSI Controller 和 CSI Node

- Identity：对外暴露这个插件本身的信息，确保插件的健康状态
- Controller：实现 Volume 管理流程当中的 Provision 和 Attach 阶段
- Node：控制 Kubernetes 节点上的 Volume 操作

这套存储插件体系多了三个独立的外部组件（External Components），即： Driver Registrar、External Provisioner 和 External Attacher，对应的正是从 Kubernetes 项 目里面剥离出来的那部分存储管理功能。

- Driver Registrar：负责将插件注册到 kubelet 里面，实际需要请求 CSI 插件的 Identity 服务来

  获取插件信息

- External Provisioner：负责 Provision 阶段，监听（Watch）了 APIServer 里的 PVC 对象。当一个 PVC 被创建时，它就会调用 CSI Controller 的 CreateVolume 方法，为你创建对应 PV

- External Attacher：负责Attach 阶段。它监听了 APIServer 里 VolumeAttachment 对象的变化并调用 CSI 的 ControllerPublish 和 ControllerUnpublish

官方建议了部署方式：[见图](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/storage/container-storage-interface.md)

CSI Controller + Identity部分以 StatefulSet 或者 Deployment 方式部署，Sidecar加上Provisioner和Attacher

CSI Node + Identity部分以 DaemonSet 方式部署，Sidecar加上Driver Registrar