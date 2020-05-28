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

External Attacher 的工作原理，是监听（Watch）了一种名叫 VolumeAttachment 的 API 对象。这种 API 对象的主要字段如下所示：

```
// VolumeAttachmentSpec is the specification of a VolumeAttachment request.
type VolumeAttachmentSpec struct {
// Attacher indicates the name of the volume driver that MUST handle this
// request. This is the name returned by GetPluginName().
Attacher string

// Source represents the volume that should be attached.
Source VolumeAttachmentSource

// The node that the volume should be attached to.
NodeName string
}
```

而这个对象的生命周期，正是由 AttachDetachController 负责管理的

这个控制循环的职责，是不断检查 Pod 所对应的 PV，在它所绑定的宿主机上的挂载情况，从而决定是否需要对这个 PV 进行 Attach（或者 Dettach）操作。

而这个 Attach 操作，在 CSI 体系里，就是创建出上面这样一个 VolumeAttachment 对象。可以看到，Attach 操作所需的 PV 的名字（Source）、宿主机的名字（NodeName）、存储插件的名字（Attacher），都是这个 VolumeAttachment 对象的一部分。

而当 External Attacher 监听到这样的一个对象出现之后，就可以立即使用 VolumeAttachment 里的这些字段，封装成一个 gRPC 请求调用 CSI Controller 的 ControllerPublishVolume 方法。

### CSI Node

CSI Node 服务对应的，是 Volume 管理流程里的“Mount 阶段”。它的代码实现，在 node.go 文件里。

kubelet 的 VolumeManagerReconciler 控制循环会直接调用 CSI Node 服务来完成 Volume 的“Mount 阶段”。

不过，在具体的实现中，这个“Mount 阶段”的处理其实被细分成了 NodeStageVolume 和 NodePublishVolume 这两个接口。这里的原因其实也很容易理解: 对于磁盘以及块设备来说，它们被 Attach 到宿主机上之后，就成为了宿主机上的一个待用存储设备。而到了“Mount 阶段”，我们首先需要格式化这个设备，然后才能把它挂载到 Volume 对应的宿主机目录上。

在 kubelet 的 VolumeManagerReconciler 控制循环中，这两步操作分别叫作**MountDevice 和 SetUp。**

其中，MountDevice 操作，就是直接调用了 CSI Node 服务里的 NodeStageVolume 接口。顾名思义，这个接口的作用，就是格式化 Volume 在宿主机上对应的存储设备，然后挂载到一个临时目录（Staging 目录）上。

NodeStageVolume 接口的实现示例如下所示：

```
func (d *Driver) NodeStageVolume(ctx context.Context, req *csi.NodeStageVolumeRequest) (*csi.NodeStageVolumeResponse, error) {
...

vol, resp, err := d.doClient.Storage.GetVolume(ctx, req.VolumeId)

...

source := getDiskSource(vol.Name)
target := req.StagingTargetPath

...

if !formatted {
ll.Info("formatting the volume for staging")
if err := d.mounter.Format(source, fsType); err != nil {
return nil, status.Error(codes.Internal, err.Error())
}
} else {
ll.Info("source device is already formatted")
}

...

if !mounted {
if err := d.mounter.Mount(source, target, fsType, options...); err != nil {
return nil, status.Error(codes.Internal, err.Error())
}
} else {
ll.Info("source device is already mounted to the target path")
}

...
return &csi.NodeStageVolumeResponse{}, nil
}
```

可以看到，在 NodeStageVolume 的实现里，我们首先通过 API 获取到了这个 Volume 对应的设备路径（getDiskSource）；然后，我们把这个设备格式化成指定的格式（ d.mounter.Format）；最后，我们把格式化后的设备挂载到了一个临时的 Staging 目录（StagingTargetPath）下。

而 SetUp 操作则会调用 CSI Node 服务的 NodePublishVolume 接口。有了上述对设备的预处理工作后，它的实现就非常简单了，如下所示：

```
func (d *Driver) NodePublishVolume(ctx context.Context, req *csi.NodePublishVolumeRequest) (*csi.NodePublishVolumeResponse, error) {
...
source := req.StagingTargetPath
target := req.TargetPath

mnt := req.VolumeCapability.GetMount()
options := mnt.MountFlag
...

if !mounted {
ll.Info("mounting the volume")
if err := d.mounter.Mount(source, target, fsType, options...); err != nil {
return nil, status.Error(codes.Internal, err.Error())
}
} else {
ll.Info("volume is already mounted")
}

return &csi.NodePublishVolumeResponse{}, nil
}
```

可以看到，在这一步实现中，我们只需要做一步操作，即：将 Staging 目录，绑定挂载到 Volume 对应的宿主机目录上。

由于 Staging 目录，正是 Volume 对应的设备被格式化后挂载在宿主机上的位置，所以当它和 Volume 的宿主机目录绑定挂载之后，这个 Volume 宿主机目录的“持久化”处理也就完成了。

当然，我在前面也曾经提到过，对于文件系统类型的存储服务来说，比如 NFS 和 GlusterFS 等，它们并没有一个对应的磁盘“设备”存在于宿主机上，所以 kubelet 在 VolumeManagerReconciler 控制循环中，会跳过 MountDevice 操作而直接执行 SetUp 操作。所以对于它们来说，也就不需要实现 NodeStageVolume 接口了。

## 部署CSI

CSI 插件的 YAML 文件的主要内容如下所示（其中，非重要的内容已经被略去）：

```
kind: DaemonSet
apiVersion: apps/v1beta2
metadata:
name: csi-do-node
namespace: kube-system
spec:
selector:
matchLabels:
app: csi-do-node
template:
metadata:
labels:
  app: csi-do-node
  role: csi-do
spec:
serviceAccount: csi-do-node-sa
hostNetwork: true
containers:
  - name: driver-registrar
    image: quay.io/k8scsi/driver-registrar:v0.3.0
    ...
  - name: csi-do-plugin
    image: digitalocean/do-csi-plugin:v0.2.0
    args :
      - "--endpoint=$(CSI_ENDPOINT)"
      - "--token=$(DIGITALOCEAN_ACCESS_TOKEN)"
      - "--url=$(DIGITALOCEAN_API_URL)"
    env:
      - name: CSI_ENDPOINT
        value: unix:///csi/csi.sock
      - name: DIGITALOCEAN_API_URL
        value: https://api.digitalocean.com/
      - name: DIGITALOCEAN_ACCESS_TOKEN
        valueFrom:
          secretKeyRef:
            name: digitalocean
            key: access-token
    imagePullPolicy: "Always"
    securityContext:
      privileged: true
      capabilities:
        add: ["SYS_ADMIN"]
      allowPrivilegeEscalation: true
    volumeMounts:
      - name: plugin-dir
        mountPath: /csi
      - name: pods-mount-dir
        mountPath: /var/lib/kubelet
        mountPropagation: "Bidirectional"
      - name: device-dir
        mountPath: /dev
volumes:
  - name: plugin-dir
    hostPath:
      path: /var/lib/kubelet/plugins/com.digitalocean.csi.dobs
      type: DirectoryOrCreate
  - name: pods-mount-dir
    hostPath:
      path: /var/lib/kubelet
      type: Directory
  - name: device-dir
    hostPath:
      path: /dev
---
kind: StatefulSet
apiVersion: apps/v1beta1
metadata:
name: csi-do-controller
namespace: kube-system
spec:
serviceName: "csi-do"
replicas: 1
template:
metadata:
labels:
  app: csi-do-controller
  role: csi-do
spec:
serviceAccount: csi-do-controller-sa
containers:
  - name: csi-provisioner
    image: quay.io/k8scsi/csi-provisioner:v0.3.0
    ...
  - name: csi-attacher
    image: quay.io/k8scsi/csi-attacher:v0.3.0
    ...
  - name: csi-do-plugin
    image: digitalocean/do-csi-plugin:v0.2.0
    args :
      - "--endpoint=$(CSI_ENDPOINT)"
      - "--token=$(DIGITALOCEAN_ACCESS_TOKEN)"
      - "--url=$(DIGITALOCEAN_API_URL)"
    env:
      - name: CSI_ENDPOINT
        value: unix:///var/lib/csi/sockets/pluginproxy/csi.sock
      - name: DIGITALOCEAN_API_URL
        value: https://api.digitalocean.com/
      - name: DIGITALOCEAN_ACCESS_TOKEN
        valueFrom:
          secretKeyRef:
            name: digitalocean
            key: access-token
    imagePullPolicy: "Always"
    volumeMounts:
      - name: socket-dir
        mountPath: /var/lib/csi/sockets/pluginproxy/
volumes:
  - name: socket-dir
    emptyDir: {}
```



我们**部署 CSI 插件的常用原则是：**

**第一，通过 DaemonSet 在每个节点上都启动一个 CSI 插件，来为 kubelet 提供 CSI Node 服务**。这是因为，CSI Node 服务需要被 kubelet 直接调用，所以它要和 kubelet“一对一”地部署起来。

此外，在上述 DaemonSet 的定义里面，除了 CSI 插件，我们还以 sidecar 的方式运行着 driver-registrar 这个外部组件。它的作用，是向 kubelet 注册这个 CSI 插件。这个注册过程使用的插件信息，则通过访问同一个 Pod 里的 CSI 插件容器的 Identity 服务获取到。

需要注意的是，由于 CSI 插件运行在一个容器里，那么 CSI Node 服务在“Mount 阶段”执行的挂载操作，实际上是发生在这个容器的 Mount Namespace 里的。可是，我们真正希望执行挂载操作的对象，都是宿主机 /var/lib/kubelet 目录下的文件和目录。

所以，在定义 DaemonSet Pod 的时候，我们需要把宿主机的 /var/lib/kubelet 以 Volume 的方式挂载进 CSI 插件容器的同名目录下，然后设置这个 Volume 的 mountPropagation=Bidirectional，即开启双向挂载传播，从而将容器在这个目录下进行的挂载操作“传播”给宿主机，反之亦然。

**第二，通过 StatefulSet 在任意一个节点上再启动一个 CSI 插件，为 External Components 提供 CSI Controller 服务**。所以，作为 CSI Controller 服务的调用者，External Provisioner 和 External Attacher 这两个外部组件，就需要以 sidecar 的方式和这次部署的 CSI 插件定义在同一个 Pod 里。

你可能会好奇，为什么我们会用 StatefulSet 而不是 Deployment 来运行这个 CSI 插件呢。

这是因为，由于 StatefulSet 需要确保应用拓扑状态的稳定性，所以它对 Pod 的更新，是严格保证顺序的，即：只有在前一个 Pod 停止并删除之后，它才会创建并启动下一个 Pod。

而像我们上面这样将 StatefulSet 的 replicas 设置为 1 的话，StatefulSet 就会确保 Pod 被删除重建的时候，永远有且只有一个 CSI 插件的 Pod 运行在集群中。这对 CSI 插件的正确性来说，至关重要。

而在今天这篇文章一开始，我们就已经定义了这个 CSI 插件对应的 StorageClass（即：do-block-storage），所以你接下来只需要定义一个声明使用这个 StorageClass 的 PVC 即可，如下所示：

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
name: csi-pvc
spec:
accessModes:
- ReadWriteOnce
resources:
requests:
storage: 5Gi
storageClassName: do-block-storage
```

当你把上述 PVC 提交给 Kubernetes 之后，你就可以在 Pod 里声明使用这个 csi-pvc 来作为持久化存储了。这一部分使用 PV 和 PVC 的内容，我就不再赘述了。

## 流程总结

对于一个部署了 CSI 存储插件的 Kubernetes 集群来说：

当用户创建了一个 PVC 之后，你前面部署的 StatefulSet 里的 External Provisioner 容器，就会监听到这个 PVC 的诞生，然后调用同一个 Pod 里的 CSI 插件的 CSI Controller 服务的 CreateVolume 方法，为你创建出对应的 PV。

这时候，运行在 Kubernetes Master 节点上的 Volume Controller，就会通过 PersistentVolumeController 控制循环，发现这对新创建出来的 PV 和 PVC，并且看到它们声明的是同一个 StorageClass。所以，它会把这一对 PV 和 PVC 绑定起来，使 PVC 进入 Bound 状态。

然后，用户创建了一个声明使用上述 PVC 的 Pod，并且这个 Pod 被调度器调度到了宿主机 A 上。这时候，Volume Controller 的 AttachDetachController 控制循环就会发现，上述 PVC 对应的 Volume，需要被 Attach 到宿主机 A 上。所以，AttachDetachController 会创建一个 VolumeAttachment 对象，这个对象携带了宿主机 A 和待处理的 Volume 的名字。

这样，StatefulSet 里的 External Attacher 容器，就会监听到这个 VolumeAttachment 对象的诞生。于是，它就会使用这个对象里的宿主机和 Volume 名字，调用同一个 Pod 里的 CSI 插件的 CSI Controller 服务的 ControllerPublishVolume 方法，完成“Attach 阶段”。

上述过程完成后，运行在宿主机 A 上的 kubelet，就会通过 VolumeManagerReconciler 控制循环，发现当前宿主机上有一个 Volume 对应的存储设备（比如磁盘）已经被 Attach 到了某个设备目录下。于是 kubelet 就会调用同一台宿主机上的 CSI 插件的 CSI Node 服务的 NodeStageVolume 和 NodePublishVolume 方法，完成这个 Volume 的“Mount 阶段”。

至此，一个完整的持久化 Volume 的创建和挂载流程就结束了。