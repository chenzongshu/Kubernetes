# 总结

CRD的全称是CustomResourceDefinition, 是Kubernetes为提高可扩展性，让开发者去自定义资源（如Deployment，StatefulSet等）的一种方法.

Operator = CRD + Controller

CRD仅仅是资源的定义，而Controller可以去监听CRD的CRUD事件来添加自定义业务逻辑。

如果说只是对CRD实例进行CRUD的话，不需要Controller也是可以实现的，只是只有数据，没有针对数据的操作。

# CRD

## CRD基本用法

yaml格式模板

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # 名称必须与下面的spec字段匹配，格式为: <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # 用于REST API的组名称: /apis/<group>/<version>
  group: stable.example.com
  # 此CustomResourceDefinition支持的版本列表
  versions:
    - name: v1
      # 每个版本都可以通过服务标志启用/禁用。
      served: true
      # 必须将一个且只有一个版本标记为存储版本。
      storage: true
  # 指定crd资源作用范围在命名空间或集群
  scope: Namespaced
  names:
    # URL中使用的复数名称: /apis/<group>/<version>/<plural>
    plural: crontabs
    # 在CLI(shell界面输入的参数)上用作别名并用于显示的单数名称
    singular: crontab
    # kind字段使用驼峰命名规则. 资源清单使用如此
    kind: CronTab
    # 短名称允许短字符串匹配CLI上的资源，意识就是能通过kubectl 在查看资源的时候使用该资源的简名称来获取。
    shortNames:
    - ct
```

然后可以创建一个CRD

```
kubectl create -f resourcedefintion.yaml
```

可以获取创建的所有CRD

```
[root@centos-kata ~]# kubectl get crd
NAME                                          CREATED AT
crontabs.stable.example.com                   2020-02-29T03:35:56Z
```

可以查看原始的yaml文件

```
[root@centos-kata ~]# kubectl get ct -o yaml
apiVersion: v1
items:
- apiVersion: stable.example.com/v1
  kind: CronTab
  metadata:
    annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
        {"apiVersion":"stable.example.com/v1","kind":"CronTab","metadata":{"annotations":{},"name":"my-new-cron-object","namespace":"default"},"spec":{"cronSpec":"* * * * */5","image":"my-awesome-cron-image"}}
    creationTimestamp: "2020-02-29T08:52:02Z"
    generation: 1
    name: my-new-cron-object
    namespace: default
    resourceVersion: "29558"
    selfLink: /apis/stable.example.com/v1/namespaces/default/crontabs/my-new-cron-object
    uid: 8c85ff73-1f93-4582-acd2-968c52285d64
  spec:
    cronSpec: '* * * * */5'
    image: my-awesome-cron-image
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```

可以创建一个my-crontab.yaml来创建一个CRD实例

```
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
```

## CRD高级特性

### Finalizer（终结器）

Finalizer（终结器）允许控制器实现异步预删除 hook。可以将终结器添加到自定义对象

```
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  finalizers:
  - finalizer.stable.example.com
```

终结器是任意字符串值，当存在时确保在资源存在时不可能进行硬删除。当所有终结器删除完之后, 才回删掉资源

### Validation（验证）

这个特性可以验证资源的字段值, 具体不展开, 可以通过`OpenAPI v3 schema`验证自定义对象是否符合标准, 此外有些字段不能限制

### 其他高级特性

此外还具有一些其他特性, 这里只做简述

- Priority（优先级）: 每列中都包含一个`priority`字段, 具有优先级的为`0`显示在标准视图中; 
优先级大于`0`的列仅在`wide`视图中显示
- Type（类型）: 列中的 type 字段可以是`OpenAPI v3`的数据类型
- Format（格式）: 多种类型
- 子资源: 自定义资源支持`/status`和`/scale`子资源, 不详细描述


# Controller

controller都是由controller-manager进行管理。
每个Controller通过API Server提供的接口实时监控整个集群的每个资源对象的当前状态，当发生各种故障导致系统状态发生变化时，会尝试通过CRUD操作将系统状态修复到“期望状态”。

如何去实现一个Controller呢？

可以使用Go来实现，并且不论是参考资料还是开源支持都非常好，推荐有Go语言基础的优先考虑用client-go来作为Kubernetes的客户端，用KubeBuilder(https://github.com/kubernetes-sigs/kubebuilder)来生成骨架代码。一个官方的Controller示例项目是sample-controller。

对于Java来说，目前Kubernetes的JavaClient有两个，一个是Jasery，另一个是Fabric8。后者要更好用一些，因为对Pod、Deployment都有DSL定义，而且构建对象是以Builder模式做的，写起来比较舒服。

Fabric8的资料目前只有https://github.com/fabric8io/kubernetes-client，注意看目录下的examples。

这些客户端本质上都是通过REST接口来与Kubernetes API Server通信的。

Controller的逻辑其实是很简单的：监听CRD实例（以及关联的资源）的CRUD事件，然后执行相应的业务逻辑

Controller主要使用到 Informer和workqueue两个核心组件。

Controller可以有一个或多个informer来跟踪某一个resource。
Informter跟API server保持通讯获取资源的最新状态并更新到本地的cache中，一旦跟踪的资源有变化，informer就会调用callback。把关心的变更的Object放到workqueue里面。
然后woker执行真正的业务逻辑，计算和比较workerqueue里items的当前状态和期望状态的差别，然后通过client-go向API server发送请求，直到驱动这个集群向用户要求的状态演化。

下面以sample-controller(https://github.com/kubernetes/sample-controller) 为例，来讲解开发自定义Controller的关键步骤和注意点。

1. 根据CRD的模板定义出自己的资源管理对象。比如crd.yaml文件

# 实例编写讲解

我现在要为 Kubernetes 添加一个名叫 Network 的 API 资源类型, yaml示例如下

```yaml
apiVersion: samplecrd.k8s.io/v1
kind: Network
metadata:
name: example-network
spec:
cidr: "192.168.0.0/16"
gateway: "192.168.0.1"
```

为了知道这个Network这个对象, 我们编写一个CRD

## 编写CRD

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
name: networks.samplecrd.k8s.io
spec:
group: samplecrd.k8s.io
version: v1
names:
kind: Network
plural: networks
scope: Namespaced
```

接下来需要稍微做些代码工作了

- **首先，我要在 GOPATH 下，创建一个结构如下的项目：**

```
$ tree $GOPATH/src/github.com/<your-name>/k8s-controller-custom-resource
.
├── controller.go
├── crd
│   └── network.yaml
├── example
│   └── example-network.yaml
├── main.go
└── pkg
└── apis
  └── samplecrd
      ├── register.go
      └── v1
          ├── doc.go
          ├── register.go
          └── types.go
```

其中，pkg/apis/samplecrd 就是 API 组的名字，v1 是版本，而 v1 下面的 types.go 文件里，则定义了 Network 对象的完整描述。

- **然后，我在 pkg/apis/samplecrd 目录下创建了一个 register.go 文件，用来放置后面要用到的全局变量。**这个文件的内容如下所示：

```go
package samplecrd

const (
GroupName = "samplecrd.k8s.io"
Version   = "v1"
)
```

- **接着，我需要在 pkg/apis/samplecrd 目录下添加一个 doc.go 文件（Golang 的文档源文件）**

  ```go
  // +k8s:deepcopy-gen=package
  
  // +groupName=samplecrd.k8s.io
  package v1
  ```

  在这个文件中，你会看到 +<tag_name>[=value] 格式的注释，这就是 Kubernetes 进行代码生成要用的 Annotation 风格的注释。

  其中，+k8s:deepcopy-gen=package 意思是，请为整个 v1 包里的所有类型定义自动生成 DeepCopy 方法；而`+groupName=samplecrd.k8s.io`，则定义了这个包对应的 API 组的名字。

  可以看到，这些定义在 doc.go 文件的注释，起到的是全局的代码生成控制的作用，所以也被称为 Global Tags。

  

- **接下来，需要添加 types.go 文件**

  顾名思义，它的作用就是定义一个 Network 类型到底有哪些字段（比如，spec 字段里的内容）。这个文件的主要内容如下所示：
  
  ```go
  package v1
  ...
  // +genclient
  // +genclient:noStatus
  // +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
  
  // Network describes a Network resource
  type Network struct {
  // TypeMeta is the metadata for the resource, like kind and apiversion
  metav1.TypeMeta `json:",inline"`
  // ObjectMeta contains the metadata for the particular object, including
  // things like...
  //  - name
  //  - namespace
  //  - self link
  //  - labels
  //  - ... etc ...
  metav1.ObjectMeta `json:"metadata,omitempty"`
  
  Spec networkspec `json:"spec"`
  }
  // networkspec is the spec for a Network resource
  type networkspec struct {
  Cidr    string `json:"cidr"`
  Gateway string `json:"gateway"`
  }
  
  // +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
  
  // NetworkList is a list of Network resources
  type NetworkList struct {
  metav1.TypeMeta `json:",inline"`
  metav1.ListMeta `json:"metadata"`
  
  Items []Network `json:"items"`
  }
  ```
  
  在上面这部分代码里，你可以看到 Network 类型定义方法跟标准的 Kubernetes 对象一样，都包括了 TypeMeta（API 元数据）和 ObjectMeta（对象元数据）字段。
  
  而其中的 Spec 字段，就是需要我们自己定义的部分。所以，在 networkspec 里，我定义了 Cidr 和 Gateway 两个字段。其中，每个字段最后面的部分比如`json:"cidr"`，指的就是这个字段被转换成 JSON 格式之后的名字，也就是 YAML 文件里的字段名字。
  
  > 如果不熟悉这个用法的话，可以查阅一下 Golang 的文档
  
  此外，除了定义 Network 类型，你还需要定义一个 NetworkList 类型，用来描述**一组 Network 对象**应该包括哪些字段。之所以需要这样一个类型，是因为在 Kubernetes 中，获取所有 X 对象的 List() 方法，返回值都是List 类型，而不是 X 类型的数组。这是不一样的。
  
  同样地，在 Network 和 NetworkList 类型上，也有代码生成注释。
  
  其中，+genclient 的意思是：请为下面这个 API 资源类型生成对应的 Client 代码（这个 Client，我马上会讲到）。而 +genclient:noStatus 的意思是：这个 API 资源类型定义里，没有 Status 字段。否则，生成的 Client 就会自动带上 UpdateStatus 方法。
  
  
  
  如果你的类型定义包括了 Status 字段的话，就不需要这句 +genclient:noStatus 注释了。比如下面这个例子：
  
  ```go
  // +genclient
  
  // Network is a specification for a Network resource
  type Network struct {
  metav1.TypeMeta   `json:",inline"`
  metav1.ObjectMeta `json:"metadata,omitempty"`
  
  Spec   NetworkSpec   `json:"spec"`
  Status NetworkStatus `json:"status"`
  }
  ```
  
  需要注意的是，+genclient 只需要写在 Network 类型上，而不用写在 NetworkList 上。因为 NetworkList 只是一个返回值类型，Network 才是“主类型”。
  
  
  
  而由于我在 Global Tags 里已经定义了为所有类型生成 DeepCopy 方法，所以这里就不需要再显式地加上 +k8s:deepcopy-gen=true 了。当然，这也就意味着你可以用 +k8s:deepcopy-gen=false 来阻止为某些类型生成 DeepCopy。
  
  
  
  你可能已经注意到，在这两个类型上面还有一句`+k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object`的注释。它的意思是，请在生成 DeepCopy 的时候，实现 Kubernetes 提供的 runtime.Object 接口。否则，在某些版本的 Kubernetes 里，你的这个类型定义会出现编译错误。这是一个固定的操作，记住即可。
  
  
  
- **最后，我需要再编写的一个 pkg/apis/samplecrd/v1/register.go 文件**。

  Network 资源类型在服务器端的注册的工作，APIServer 会自动帮我们完成。但与之对应的，我们还需要让客户端也能“知道”Network 资源类型的定义。

  这就需要我们在项目里添加一个 register.go 文件。它最主要的功能，就是定义了如下所示的 addKnownTypes() 方法：

  ```go
  package v1
  ...
  // addKnownTypes adds our types to the API scheme by registering
  // Network and NetworkList
  func addKnownTypes(scheme *runtime.Scheme) error {
  scheme.AddKnownTypes(
  SchemeGroupVersion,
  &Network{},
  &NetworkList{},
  )
  
  // register the type in the scheme
  metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
  return nil
  }
  ```

  有了这个方法，Kubernetes 就能够在后面生成客户端的时候，“知道”Network 以及 NetworkList 类型的定义了。

  像上面这种**register.go 文件里的内容其实是非常固定的**

  

  这样，Network 对象的定义工作就全部完成了。可以看到，它其实定义了两部分内容：

  1. 第一部分是，自定义资源类型的 API 描述，包括：组（Group）、版本（Version）、资源类型（Resource）等。这相当于告诉了计算机：兔子是哺乳动物。

  2. 第二部分是，自定义资源类型的对象描述，包括：Spec、Status 等。这相当于告诉了计算机：兔子有长耳朵和三瓣嘴。

  

  接下来，我就要使用 Kubernetes 提供的代码生成工具，为上面定义的 Network 资源类型自动生成 clientset、informer 和 lister。其中，clientset 就是操作 Network 对象所需要使用的客户端，而 informer 和 lister 这两个包的主要功能 

  

  这个代码生成工具名叫`k8s.io/code-generator`，使用方法如下所示：

  ```go
  # 代码生成的工作目录，也就是我们的项目路径
  $ ROOT_PACKAGE="github.com/resouer/k8s-controller-custom-resource"
  # API Group
  $ CUSTOM_RESOURCE_NAME="samplecrd"
  # API Version
  $ CUSTOM_RESOURCE_VERSION="v1"
  
  # 安装 k8s.io/code-generator
  $ go get -u k8s.io/code-generator/...
  $ cd $GOPATH/src/k8s.io/code-generator
  
  # 执行代码自动生成，其中 pkg/client 是生成目标目录，pkg/apis 是类型定义目录
  $ ./generate-groups.sh all "$ROOT_PACKAGE/pkg/client" "$ROOT_PACKAGE/pkg/apis" "$CUSTOM_RESOURCE_NAME:$CUSTOM_RESOURCE_VERSION"
  ```

  代码生成工作完成之后，我们再查看一下这个项目的目录结构：

  ```bash
  $ tree
  .
  ├── controller.go
  ├── crd
  │   └── network.yaml
  ├── example
  │   └── example-network.yaml
  ├── main.go
  └── pkg
  ├── apis
  │   └── samplecrd
  │       ├── constants.go
  │       └── v1
  │           ├── doc.go
  │           ├── register.go
  │           ├── types.go
  │           └── zz_generated.deepcopy.go
  └── client
    ├── clientset
    ├── informers
    └── listers
  ```

  其中，pkg/apis/samplecrd/v1 下面的 zz_generated.deepcopy.go 文件，就是自动生成的 DeepCopy 代码文件。

  

  而整个 client 目录，以及下面的三个包（clientset、informers、 listers），都是 Kubernetes 为 Network 类型生成的客户端库，这些库会在后面编写自定义控制器的时候用到。

  

## 编写Controller

总得来说，编写自定义控制器代码的过程包括：编写 main 函数、编写自定义控制器的定义，以及编写控制器里的业务逻辑三个部分。

### 编写main 函数

main 函数的主要工作就是，定义并初始化一个自定义控制器（Custom Controller），然后启动它。这部分代码的主要内容如下所示：

```go
func main() {
...

cfg, err := clientcmd.BuildConfigFromFlags(masterURL, kubeconfig)
...
kubeClient, err := kubernetes.NewForConfig(cfg)
...
networkClient, err := clientset.NewForConfig(cfg)
...

networkInformerFactory := informers.NewSharedInformerFactory(networkClient, ...)

controller := NewController(kubeClient, networkClient,
networkInformerFactory.Samplecrd().V1().Networks())

go networkInformerFactory.Start(stopCh)

if err = controller.Run(2, stopCh); err != nil {
glog.Fatalf("Error running controller: %s", err.Error())
}
}
```

可以看到，这个 main 函数主要通过三步完成了初始化并启动一个自定义控制器的工作。

**第一步**：main 函数根据我提供的 Master 配置（APIServer 的地址端口和 kubeconfig 的路径），创建一个 Kubernetes 的 client（kubeClient）和 Network 对象的 client（networkClient）。

但是，如果我没有提供 Master 配置呢？

这时，main 函数会直接使用一种名叫**InClusterConfig**的方式来创建这个 client。这个方式，会假设你的自定义控制器是以 Pod 的方式运行在 Kubernetes 集群里的。

Kubernetes 里所有的 Pod 都会以 Volume 的方式自动挂载 Kubernetes 的默认 ServiceAccount。所以，这个控制器就会直接使用默认 ServiceAccount 数据卷里的授权信息，来访问 APIServer。

**第二步**：main 函数为 Network 对象创建一个叫作 InformerFactory（即：networkInformerFactory）的工厂，并使用它生成一个 Network 对象的 Informer，传递给控制器。

**第三步**：main 函数启动上述的 Informer，然后执行 controller.Run，启动自定义控制器。

编写自定义控制器的过程难道就这么简单吗？下面来看看控制器原理



#### 自定义控制器的工作原理

![](https://static001.geekbang.org/resource/image/32/c3/32e545dcd4664a3f36e95af83b571ec3.png)

![controller](/Users/chenzongshu/Documents/github/Kubernetes/pic/controller.png)

**这个控制器要做的第一件事，是从 Kubernetes 的 APIServer 里获取它所关心的对象，也就是我定义的 Network 对象**。

这个操作，依靠的是一个叫作 Informer的代码库完成的。Informer 与 API 对象是一一对应的，所以我传递给自定义控制器的，正是一个 Network 对象的 Informer（Network Informer）。

事实上，Network Informer 正是使用这个 networkClient，跟 APIServer 建立了连接。不过，真正负责维护这个连接的，则是 Informer 所使用的 Reflector 包。

更具体地说，Reflector 使用的是一种叫作**ListAndWatch**的方法，来“获取”并“监听”这些 Network 对象实例的变化。

在 ListAndWatch 机制下，一旦 APIServer 端有新的 Network 实例被创建、删除或者更新，Reflector 都会收到“事件通知”。这时，该事件及它对应的 API 对象这个组合，就被称为增量（Delta），它会被放进一个 Delta FIFO Queue（即：增量先进先出队列）中。

而另一方面，Informe 会不断地从这个 Delta FIFO Queue 里读取（Pop）增量。每拿到一个增量，Informer 就会判断这个增量里的事件类型，然后创建或者更新本地对象的缓存。这个缓存，在 Kubernetes 里一般被叫作 Store。

比如，如果事件类型是 Added（添加对象），那么 Informer 就会通过一个叫作 Indexer 的库把这个增量里的 API 对象保存在本地缓存中，并为它创建索引。相反地，如果增量的事件类型是 Deleted（删除对象），那么 Informer 就会从本地缓存中删除这个对象。

这个**同步本地缓存的工作，是 Informer 的第一个职责，也是它最重要的职责。**

而**Informer 的第二个职责，则是根据这些事件的类型，触发事先注册好的 ResourceEventHandler**。这些 Handler，需要在创建控制器的时候注册给它对应的 Informer。



### 编写这个控制器的定义

```go
func (c *Controller) runWorker() {
for c.processNextWorkItem() {
}
}

func (c *Controller) processNextWorkItem() bool {
obj, shutdown := c.workqueue.Get()

...

err := func(obj interface{}) error {
...
if err := c.syncHandler(key); err != nil {
return fmt.Errorf("error syncing '%s': %s", key, err.Error())
}

c.workqueue.Forget(obj)
...
return nil
}(obj)

...

return true
}

func (c *Controller) syncHandler(key string) error {

namespace, name, err := cache.SplitMetaNamespaceKey(key)
...

network, err := c.networksLister.Networks(namespace).Get(name)
if err != nil {
if errors.IsNotFound(err) {
glog.Warningf("Network does not exist in local cache: %s/%s, will delete it from Neutron ...",
namespace, name)

glog.Warningf("Network: %s/%s does not exist in local cache, will delete it from Neutron ...",
namespace, name)

// FIX ME: call Neutron API to delete this network by name.
//
// neutron.Delete(namespace, name)

return nil
}
...

return err
}

glog.Infof("[Neutron] Try to process network: %#v ...", network)

// FIX ME: Do diff().
//
// actualNetwork, exists := neutron.Get(namespace, name)
//
// if !exists {
//   neutron.Create(namespace, name)
// } else if !reflect.DeepEqual(actualNetwork, network) {
//   neutron.Update(namespace, name)
// }

return nil
}
```

可以看到，在这个执行周期里（processNextWorkItem），我们**首先**从工作队列里出队（workqueue.Get）了一个成员，也就是一个 Key（Network 对象的：namespace/name）。

**然后**，在 syncHandler 方法中，我使用这个 Key，尝试从 Informer 维护的缓存中拿到了它所对应的 Network 对象。

可以看到，在这里，我使用了 networksLister 来尝试获取这个 Key 对应的 Network 对象。这个操作，其实就是在访问本地缓存的索引。实际上，在 Kubernetes 的源码中，你会经常看到控制器从各种 Lister 里获取对象，比如：podLister、nodeLister 等等，它们使用的都是 Informer 和缓存机制。

而如果控制循环从缓存中拿不到这个对象（即：networkLister 返回了 IsNotFound 错误），那就意味着这个 Network 对象的 Key 是通过前面的“删除”事件添加进工作队列的。所以，尽管队列里有这个 Key，但是对应的 Network 对象已经被删除了。

这时候，我就需要调用 Neutron 的 API，把这个 Key 对应的 Neutron 网络从真实的集群里删除掉。

**而如果能够获取到对应的 Network 对象，我就可以执行控制器模式里的对比“期望状态”和“实际状态”的逻辑了。**

其中，自定义控制器“千辛万苦”拿到的这个 Network 对象，**正是 APIServer 里保存的“期望状态”**，即：用户通过 YAML 文件提交到 APIServer 里的信息。当然，在我们的例子里，它已经被 Informer 缓存在了本地。

**那么，“实际状态”又从哪里来呢？**

当然是来自于实际的集群了。

所以，我们的控制循环需要通过 Neutron API 来查询实际的网络情况。

比如，我可以先通过 Neutron 来查询这个 Network 对象对应的真实网络是否存在。

- 如果不存在，这就是一个典型的“期望状态”与“实际状态”不一致的情形。这时，我就需要使用这个 Network 对象里的信息（比如：CIDR 和 Gateway），调用 Neutron API 来创建真实的网络。
- 如果存在，那么，我就要读取这个真实网络的信息，判断它是否跟 Network 对象里的信息一致，从而决定我是否要通过 Neutron 来更新这个已经存在的真实网络。

这样，我就通过对比“期望状态”和“实际状态”的差异，完成了一次调协（Reconcile）的过程。

至此，一个完整的自定义 API 对象和它所对应的自定义控制器，就编写完毕了。

### 总结

所谓的 Informer，就是一个自带缓存和索引机制，可以触发 Handler 的客户端库。这个本地缓存在 Kubernetes 中一般被称为 Store，索引一般被称为 Index。

Informer 使用了 Reflector 包，它是一个可以通过 ListAndWatch 机制获取并监视 API 对象变化的客户端封装。

Reflector 和 Informer 之间，用到了一个“增量先进先出队列”进行协同。而 Informer 与你要编写的控制循环之间，则使用了一个工作队列来进行协同。

在实际应用中，除了控制循环之外的所有代码，实际上都是 Kubernetes 为你自动生成的，即：pkg/client/{informers, listers, clientset}里的内容。

而这些自动生成的代码，就为我们提供了一个可靠而高效地获取 API 对象“期望状态”的编程库。

所以，接下来，作为开发者，你就只需要关注如何拿到“实际状态”，然后如何拿它去跟“期望状态”做对比，从而决定接下来要做的业务逻辑即可。
















