怎么编写自己的存储插件, 目前有两种方式: FlexVolume与CSI

# FlexVolume

FlexVolume 是 Kubernetes v1.8+ 支持的一种存储插件扩展方式。类似于 CNI 插件，它需要外部插件将二进制文件放到预先配置的路径中（如 /usr/libexec/kubernetes/kubelet-plugins/volume/exec/），并需要在系统中安装好所有需要的依赖。

实现一个 FlexVolume 包括两个步骤
- 实现 FlexVolume 插件接口，包括 `init/attach/detach/waitforattach/isattached/mountdevice/unmountdevice/mount/umount` 等命令（可参考 lvm 示例 和 NFS 示例）,返回JSON格式数据
- 将插件放到 `/usr/libexec/kubernetes/kubelet-plugins/volume/exec/<vendor~driver>/<driver>` 目录中

FlexVolume实现方式，虽然简单，但局限性却很大

- 跟 Kubernetes 内置的 NFS 插件类似，这个 NFS FlexVolume 插件，也不能支持 Dynamic Provisioning（即：为每个 PVC 自动创建 PV 和对应的 Volume）。除非你再为它编写一个专门的 External Provisioner
- 我的插件在执行 mount 操作的时候，可能会生成一些挂载信息。这些信息，在后面执行 unmount 操作的时候会被用到。可是，在上述 FlexVolume 的实现里，你没办法把这些信息保存在一个变量里，等到 unmount 的时候直接使用

原因：FlexVolume 每一次对插件可执行文件的调用，都是一次完全独立的操作。所以，我们只能把这些信息写在一个宿主机上的临时文件里，等到 unmount 的时候再去读取。

# CSI

无论是 FlexVolume，还是 Kubernetes 内置的其他存储插件，它们实际上担任的角色，仅仅是 Volume 管理中的“Attach 阶段”和“Mount 阶段”的具体执行者。而像 Dynamic Provisioning 这样的功能，就不是存储插件的责任，而是 Kubernetes 本身存储管理功能的一部分。

相比之下，**CSI 插件体系的设计思想，就是把这个 Provision 阶段，以及 Kubernetes 里的一部分存储管理功能，从主干代码里剥离出来，做成了几个单独的组件**。这些组件会通过 Watch API 监听 Kubernetes 里与存储相关的事件变化，比如 PVC 的创建，来执行具体的存储管理动作。

而这些管理动作，比如“Attach 阶段”和“Mount 阶段”的具体操作，实际上就是通过调用 CSI 插件来完成的。

示意图如下：
![](https://static001.geekbang.org/resource/image/d4/ad/d4bdc7035f1286e7a423da851eee89ad.png)

可以看到，这套存储插件体系多了三个独立的外部组件（External Components），即：Driver Registrar、External Provisioner 和 External Attacher，对应的正是从 Kubernetes 项目里面剥离出来的那部分存储管理功能。

需要注意的是，External Components 虽然是外部组件，但依然由 Kubernetes 社区来开发和维护。

而图中最右侧的部分，就是需要我们编写代码来实现的 CSI 插件。一个 CSI 插件只有一个二进制文件，但它会以 gRPC 的方式对外提供三个服务（gRPC Service），分别叫作：CSI Identity、CSI Controller 和 CSI Node。

## Driver Registrar 

其中，Driver Registrar 组件，负责将插件注册到 kubelet 里面（这可以类比为，将可执行文件放在插件目录下）。而在具体实现上，Driver Registrar 需要请求 CSI 插件的 Identity 服务来获取插件信息。

## External Provisioner 

而External Provisioner 组件，负责的正是 Provision 阶段。在具体实现上，External Provisioner 监听（Watch）了 APIServer 里的 PVC 对象。当一个 PVC 被创建时，它就会调用 CSI Controller 的 CreateVolume 方法，为你创建对应 PV。

此外，如果你使用的存储是公有云提供的磁盘（或者块设备）的话，这一步就需要调用公有云（或者块设备服务）的 API 来创建这个 PV 所描述的磁盘（或者块设备）了。

不过，由于 CSI 插件是独立于 Kubernetes 之外的，所以在 CSI 的 API 里不会直接使用 Kubernetes 定义的 PV 类型，而是会自己定义一个单独的 Volume 类型。

## External Attacher

最后一个External Attacher 组件，负责的正是“Attach 阶段”。在具体实现上，它监听了 APIServer 里 VolumeAttachment 对象的变化。VolumeAttachment 对象是 Kubernetes 确认一个 Volume 可以进入“Attach 阶段”的重要标志，我会在下一篇文章里为你详细讲解。

一旦出现了 VolumeAttachment 对象，External Attacher 就会调用 CSI Controller 服务的 ControllerPublish 方法，完成它所对应的 Volume 的 Attach 阶段。

而 Volume 的“Mount 阶段”，并不属于 External Components 的职责。当 kubelet 的 VolumeManagerReconciler 控制循环检查到它需要执行 Mount 操作的时候，会通过 pkg/volume/csi 包，直接调用 CSI Node 服务完成 Volume 的“Mount 阶段”。

在实际使用 CSI 插件的时候，我们会将这三个 External Components 作为 sidecar 容器和 CSI 插件放置在同一个 Pod 中。由于 External Components 对 CSI 插件的调用非常频繁，所以这种 sidecar 的部署方式非常高效。

下面来讲讲CSI Identity、CSI Controller 和 CSI Node。

## CSI Identity

其中，CSI 插件的 CSI Identity 服务，负责对外暴露这个插件本身的信息，如下所示

```go
service Identity {
// return the version and name of the plugin
rpc GetPluginInfo(GetPluginInfoRequest)
returns (GetPluginInfoResponse) {}
// reports whether the plugin has the ability of serving the Controller interface
rpc GetPluginCapabilities(GetPluginCapabilitiesRequest)
returns (GetPluginCapabilitiesResponse) {}
// called by the CO just to check whether the plugin is running or not
rpc Probe (ProbeRequest)
returns (ProbeResponse) {}
}
```

## CSI Controller

CSI Controller 服务，定义的则是对 CSI Volume（对应 Kubernetes 里的 PV）的管理接口，比如：创建和删除 CSI Volume、对 CSI Volume 进行 Attach/Dettach（在 CSI 里，这个操作被叫作 Publish/Unpublish），以及对 CSI Volume 进行 Snapshot 等，它们的接口定义如下所示：

```go
service Controller {
// provisions a volume
rpc CreateVolume (CreateVolumeRequest)
returns (CreateVolumeResponse) {}

// deletes a previously provisioned volume
rpc DeleteVolume (DeleteVolumeRequest)
returns (DeleteVolumeResponse) {}

// make a volume available on some required node
rpc ControllerPublishVolume (ControllerPublishVolumeRequest)
returns (ControllerPublishVolumeResponse) {}

// make a volume un-available on some required node
rpc ControllerUnpublishVolume (ControllerUnpublishVolumeRequest)
returns (ControllerUnpublishVolumeResponse) {}

...

// make a snapshot
rpc CreateSnapshot (CreateSnapshotRequest)
returns (CreateSnapshotResponse) {}

// Delete a given snapshot
rpc DeleteSnapshot (DeleteSnapshotRequest)
returns (DeleteSnapshotResponse) {}

...
}
```

不难发现，CSI Controller 服务里定义的这些操作有个共同特点，那就是它们都无需在宿主机上进行，而是属于 Kubernetes 里 Volume Controller 的逻辑，也就是属于 Master 节点的一部分。

需要注意的是，正如我在前面提到的那样，CSI Controller 服务的实际调用者，并不是 Kubernetes（即：通过 pkg/volume/csi 发起 CSI 请求），而是 External Provisioner 和 External Attacher。这两个 External Components，分别通过监听 PVC 和 VolumeAttachement 对象，来跟 Kubernetes 进行协作。

## CSI Volume

而 CSI Volume 需要在宿主机上执行的操作，都定义在了 CSI Node 服务里面，如下所示：

```go

service Node {
// temporarily mount the volume to a staging path
rpc NodeStageVolume (NodeStageVolumeRequest)
returns (NodeStageVolumeResponse) {}

// unmount the volume from staging path
rpc NodeUnstageVolume (NodeUnstageVolumeRequest)
returns (NodeUnstageVolumeResponse) {}

// mount the volume from staging to target path
rpc NodePublishVolume (NodePublishVolumeRequest)
returns (NodePublishVolumeResponse) {}

// unmount the volume from staging path
rpc NodeUnpublishVolume (NodeUnpublishVolumeRequest)
returns (NodeUnpublishVolumeResponse) {}

// stats for the volume
rpc NodeGetVolumeStats (NodeGetVolumeStatsRequest)
returns (NodeGetVolumeStatsResponse) {}

...

// Similar to NodeGetId
rpc NodeGetInfo (NodeGetInfoRequest)
returns (NodeGetInfoResponse) {}
}
```

需要注意的是，“Mount 阶段”在 CSI Node 里的接口，是由 NodeStageVolume 和 NodePublishVolume 两个接口共同实现的

# CSI插件编写实战

## 使用

而有了 CSI 插件之后，持久化存储的用法就非常简单了，你只需要创建一个如下所示的 StorageClass 对象即可：

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
name: do-block-storage
namespace: kube-system
annotations:
storageclass.kubernetes.io/is-default-class: "true"
provisioner: com.digitalocean.csi.dobs
```

## CSI插件编写

其实，一个 CSI 插件的代码结构非常简单，如下所示：

```bash
tree $GOPATH/src/github.com/digitalocean/csi-digitalocean/driver  
$GOPATH/src/github.com/digitalocean/csi-digitalocean/driver 
├── controller.go
├── driver.go
├── identity.go
├── mounter.go
└── node.go
```

其中，CSI Identity 服务的实现，就定义在了 driver 目录下的 identity.go 文件里。

当然，为了能够让 Kubernetes 访问到 CSI Identity 服务，我们需要先在 driver.go 文件里，定义一个标准的 gRPC Server，如下所示：

```go
// Run starts the CSI plugin by communication over the given endpoint
func (d *Driver) Run() error {
...

listener, err := net.Listen(u.Scheme, addr)
...

d.srv = grpc.NewServer(grpc.UnaryInterceptor(errHandler))
csi.RegisterIdentityServer(d.srv, d)
csi.RegisterControllerServer(d.srv, d)
csi.RegisterNodeServer(d.srv, d)

d.ready = true // we're now ready to go!
...
return d.srv.Serve(listener)
}
```

可以看到，只要把编写好的 gRPC Server 注册给 CSI，它就可以响应来自 External Components 的 CSI 请求了。

### CSI Identity

**CSI Identity 服务中，最重要的接口是 GetPluginInfo**，它返回的就是这个插件的名字和版本号，如下所示：

```go
func (d *Driver) GetPluginInfo(ctx context.Context, req *csi.GetPluginInfoRequest) (*csi.GetPluginInfoResponse, error) {
resp := &csi.GetPluginInfoResponse{
Name:          driverName,
VendorVersion: version,
}
...
}
```

其中，driverName 的值，就是`StorageClass`的yaml里面的`provisioner`

所以说，Kubernetes 正是通过 GetPluginInfo 的返回值，来找到你在 StorageClass 里声明要使用的 CSI 插件的。

另外一个**GetPluginCapabilities 接口也很重要**。这个接口返回的是这个 CSI 插件的“能力”。

比如，当你编写的 CSI 插件不准备实现“Provision 阶段”和“Attach 阶段”（比如，一个最简单的 NFS 存储插件就不需要这两个阶段）时，你就可以通过这个接口返回：本插件不提供 CSI Controller 服务，即：没有 csi.PluginCapability_Service_CONTROLLER_SERVICE 这个“能力”。这样，Kubernetes 就知道这个信息了。

最后，**CSI Identity 服务还提供了一个 Probe 接口**。Kubernetes 会调用它来检查这个 CSI 插件是否正常工作。

一般情况下，我建议你在编写插件时给它设置一个 Ready 标志，当插件的 gRPC Server 停止的时候，把这个 Ready 标志设置为 false。或者，你可以在这里访问一下插件的端口，类似于健康检查的做法。

### CSI Controller

这个服务主要实现的就是 Volume 管理流程中的“Provision 阶段”和“Attach 阶段”。

**“Provision 阶段”对应的接口，是 CreateVolume 和 DeleteVolume**，它们的调用者是 External Provisoner。以 CreateVolume 为例，它的主要逻辑如下所示：

```go
func (d *Driver) CreateVolume(ctx context.Context, req *csi.CreateVolumeRequest) (*csi.CreateVolumeResponse, error) {
...

volumeReq := &godo.VolumeCreateRequest{
Region:        d.region,
Name:          volumeName,
Description:   createdByDO,
SizeGigaBytes: size / GB,
}

...

vol, _, err := d.doClient.Storage.CreateVolume(ctx, volumeReq)

...

resp := &csi.CreateVolumeResponse{
Volume: &csi.Volume{
Id:            vol.ID,
CapacityBytes: size,
AccessibleTopology: []*csi.Topology{
{
Segments: map[string]string{
"region": d.region,
},
},
},
},
}

return resp, nil
}
```

可以看到, CreateVolume 需要做的操作, 就是调用云或者其他块存储(比如Ceph, Cinder)创建存储卷的 API

而“**Attach 阶段”对应的接口是 ControllerPublishVolume 和 ControllerUnpublishVolume**，它们的调用者是 External Attacher。以 ControllerPublishVolume 为例，它的逻辑如下所示：

```
func (d *Driver) ControllerPublishVolume(ctx context.Context, req *csi.ControllerPublishVolumeRequest) (*csi.ControllerPublishVolumeResponse, error) {
...

dropletID, err := strconv.Atoi(req.NodeId)

// check if volume exist before trying to attach it
_, resp, err := d.doClient.Storage.GetVolume(ctx, req.VolumeId)

...

// check if droplet exist before trying to attach the volume to the droplet
_, resp, err = d.doClient.Droplets.Get(ctx, dropletID)

...

action, resp, err := d.doClient.StorageActions.Attach(ctx, req.VolumeId, dropletID)

...

if action != nil {
ll.Info("waiting until volume is attached")
if err := d.waitAction(ctx, req.VolumeId, action.ID); err != nil {
return nil, err
}
}

ll.Info("volume is attached")
return &csi.ControllerPublishVolumeResponse{}, nil
}
```

可以看到, ControllerPublishVolume 在“Attach 阶段”需要做的工作，是调用 API，将我们前面创建的存储卷，挂载到指定的虚拟机上

其中，存储卷由请求中的 VolumeId 来指定。而虚拟机，也就是将要运行 Pod 的宿主机，则由请求中的 NodeId 来指定。这些参数，都是 External Attacher 在发起请求时需要设置的。

