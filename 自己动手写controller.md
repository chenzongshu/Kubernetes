# Controller

控制器，就是监控kubernetes中各种资源的变化，并把当前状态变成期望状态

kubernetes社区提供了一个简单例子： [sample-controller](https://github.com/kubernetes/sample-controller)

![controller设计图](./pic/controller.png)

本节介绍一些前置知识

## CRD

CRD就是自定义资源，官方链接 [CRD](https://kubernetes.io/zh/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)

CRD 可以是名字空间作用域的，也可以 是集群作用域的，取决于 CRD 的 `scope` 字段设置。

```
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # 名字必需与下面的 spec 字段匹配，并且格式为 '<名称的复数形式>.<组名>'
  name: crontabs.stable.example.com
spec:
  # 组名称，用于 REST API: /apis/<组>/<版本>
  group: stable.example.com
  # 列举此 CustomResourceDefinition 所支持的版本
  versions:
    - name: v1
      # 每个版本都可以通过 served 标志来独立启用或禁止
      served: true
      # 其中一个且只有一个版本必需被标记为存储版本
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
  # 可以是 Namespaced 或 Cluster
  scope: Namespaced
  names:
    # 名称的复数形式，用于 URL：/apis/<组>/<版本>/<名称的复数形式>
    plural: crontabs
    # 名称的单数形式，作为命令行使用时和显示时的别名
    singular: crontab
    # kind 通常是单数形式的驼峰编码（CamelCased）形式。你的资源清单会使用这一形式。
    kind: CronTab
    # shortNames 允许你在命令行使用较短的字符串来匹配资源
    shortNames:
    - ct
```

## 代码生成器

自动代码生成工具将controller之外的事情都做好了，我们只要专注于controller的开发就好

- `API Machinery`: 定义API等级的方案，类型（Typing），编码（Encoding），解码（Decoding），验证（Validate），类型转换与相关工具等等功能。当我们要实现一个新的API资源时，就必须通过`API Machinery`来注册Scheme，另外A也定义TypeMeta，ObjectMeta，ListMeta，Label与Selector等等，而这些几乎在每个Kubernetes API的基础上使用。
- `API`:  主要提供Kubernetes原生的API资源类型的Scheme，这包含命名空间，Pod等等。该函数库也提供了每个API资源类型，当前所支持的版本，如：v1，v1beta1。而其中一些API资源都依功能导向被分组化
- `gengo`: 主要用于通过Go语言文件产生各种系统与API所需的文件，例如说Protobuf。而该应用也包含了Set，Deep-copy，Defaulter等等产生器（Generator），这些会被用作产生定制化客户函式库
- `code-generator`: 主要用于产生 kubernetes-style API types的Client, Deep-copy, Informer, Lister等功能的程序. 这是因为Go语言中没有泛型(Generic)的概念, 因此不同的API资源类型, 都要写一次上诉这些功能会有大量重复的代码, 因此kubernetes采用定义好的结构后, 才通过这个工具产生相关代码.



# 编写controller

我们来定义一个管理自定义资源VM的控制器

自己编写controller有三步：

- 定义CRD

- 生成自定义资源的Clientset、Informers、Listers等

- 编写Controller等代码


先建立如下目录

```
controller
├── LICENSE
├── README.md
├── deploy # 部署 Controller 的相关文件，如 Deployment、CRD、RBAC。
├── go.mod # Go mod package 
├── go.sum # Go mod package 
├── hack   # 存放一些常使用到的脚本
└── pkg    # 控制器相关程序
```

## 定义资源

创建CRD文件

```
[root@localhost controller]# cat deploy/crd.yaml 
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: virtualmachines.cloudnative.czs
spec:
  group: cloudnative.czs
  version: v1alpha1
  names:
    kind: VirtualMachine
    singular: virtualmachine
    plural: virtualmachines
    shortNames:
    - vm
  scope: Namespaced
```

创建资源

```
[root@localhost controller]# kubectl apply -f deploy/crd.yaml
customresourcedefinition.apiextensions.k8s.io/virtualmachines.cloudnative.czs created
[root@localhost controller]# 
[root@localhost controller]# kubectl get crd
NAME                                                  CREATED AT
virtualmachines.cloudnative.czs                       2020-10-10T05:38:43Z
[root@localhost controller]# 
[root@localhost controller]# kubectl get vm
error: the server doesn't have a resource type "vm"
```

然后创建VM

```
[root@localhost controller]# cat deploy/vm.yaml 
apiVersion: cloudnative.czs/v1alpha1
kind: VirtualMachine 
metadata: 
  name: test-vm
spec:
  action: active
  resource:
    cpu: 2
    memory: 4G
    rootDisk: 40G
```

## 自动生成代码

跟官方例子一样，我们也使用「[code generator](https://github.com/kubernetes/code-generator)」这个工具，基于已经定义好的CRD，自动生成Controller基础代码。先来下载`code-generator`

```go
go get -u k8s.io/code-generator/
go get -u k8s.io/apimachinery/pkg/apis/meta/v1
```

可以看到下载的目录里面有`generate-groups.sh`文件了

```shell
[root@localhost code-generator@v0.19.2]# pwd
/root/go/pkg/mod/k8s.io/code-generator@v0.19.2
[root@localhost code-generator@v0.19.2]# ls
cmd                 _examples                    Godeps  hack     pkg                third_party
code-of-conduct.md  generate-groups.sh           go.mod  LICENSE  README.md          tools.go
CONTRIBUTING.md     generate-internal-groups.sh  go.sum  OWNERS   SECURITY_CONTACTS

```

再看下需要准备的代码框架：

```
└── pkg
    └── apis # APIs 定义
        └── cloudnative # 提供该 Package 的 API Group Name。
            ├── register.go
            └── v1alpha1 # API 各版本结构定义。Kubernetes API 是支援多版本的。
                ├── doc.go
                ├── register.go
                └── types.go
```

### v1alpha1/doc.go

定义code-generator 的 Global tags. 可标识当前版本Package中

```go
//下面两行是用来帮助生成Controller代码的
// +k8s:deepcopy-gen=package
// +groupName=cloudnative.czs

// Package v1alpha1 是定义Controller的v1alpha1版本
// 所以你可以定义多个版本的Controller
package v1alpha1
```

### v1alpha1/types.go

定义CRD中资源的结构, 以及定义code-generator的local tags. 

```go
package v1alpha1

import (
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

type VirtualMachine struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   VirtualMachineSpec   `json:"spec"`
	Status VirtualMachineStatus `json:"status"`
}

type VirtualMachineSpec struct {
	Resource corev1.ResourceList `json:"resource"`
}

type VirtualMachinePhase string

const (
	VirtualMachineNone        VirtualMachinePhase = ""
	VirtualMachineCreating    VirtualMachinePhase = "Creating"
	VirtualMachineActive      VirtualMachinePhase = "Active"
	VirtualMachineFailed      VirtualMachinePhase = "Failed"
	VirtualMachineTerminating VirtualMachinePhase = "Terminating"
	VirtualMachineUnknown     VirtualMachinePhase = "Unknown"
)

type ResourceUsage struct {
	CPU    float64 `json:"cpu"`
	Memory float64 `json:"memory"`
}

type ServerStatus struct {
	ID    string        `json:"id"`
	State string        `json:"state"`
	Usage ResourceUsage `json:"usage"`
}

type VirtualMachineStatus struct {
	Phase          VirtualMachinePhase `json:"phase"`
	Reason         string              `json:"reason,omitempty"`
	Server         ServerStatus        `json:"server,omitempty"`
	LastUpdateTime metav1.Time         `json:"lastUpdateTime"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

type VirtualMachineList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata"`

	Items []VirtualMachine `json:"items"` 
}
```

### v1alpha1/register.go

用于将刚建立的新API版本与新资源类型注册到API Group Schema中, 以便API Server能识别

> Scheme: 用于API 资源群组之间的的序列化, 反序列化与版本转换

```go
[root@localhost v1alpha1]# cat register.go 
package v1alpha1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"

	"controller/pkg/apis/cloudnative"
)

var SchemeGroupVersion = schema.GroupVersion{Group: cloudnative.GroupName, Version: "v1alpha1"}

func Kind(kind string) schema.GroupKind {
	return SchemeGroupVersion.WithKind(kind).GroupKind()
}

func Resource(resource string) schema.GroupResource {
	return SchemeGroupVersion.WithResource(resource).GroupResource()
}

var (
	SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
	AddToScheme = SchemeBuilder.AddToScheme
)

func addKnownTypes(scheme *runtime.Scheme) error {
	scheme.AddKnownTypes(SchemeGroupVersion,
		&VirtualMachine{},
		&VirtualMachineList{},
	)
	metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
	return nil
}

```



### register.go

这个文件很简单, 主要是给CRD取个groupname

```go
package cloudnative

// GroupName is the group name used in this package
const (
	GroupName = "cloudnative.czs"
)
```

### 生成代码

如果`hack`下面没有`boilerplate.go.txt`文件, 创建一个, 里面就是license模板

#### 方法1

在`controller`文件夹执行shell生成代码

```
[root@localhost controller]# /root/go/pkg/mod/k8s.io/code-generator@v0.19.2/generate-groups.sh "deepcopy,client,informer,lister" controller/pkg/client controller/pkg/apis "cloudnative:v1alpha1"  --output-base /home/czs/ --go-header-file /home/czs/controller/hack/boilerplate.go.txt

Generating deepcopy funcs
Generating clientset for cloudnative:v1alpha1 at controller/pkg/client/clientset
Generating listers for cloudnative:v1alpha1 at controller/pkg/client/listers
Generating informers for cloudnative:v1alpha1 at controller/pkg/client/informers

```

每个参数含义是

```
/root/go/pkg/mod/k8s.io/code-generator@v0.19.2/generate-groups.sh \
# 期望生成的函数列表
"deepcopy,client,informer,lister" \
# 生成代码的目标目录
controller/pkg//client \
# CRD所在目录
controller/pkg/apis \ 
# CRD的group name和version
"cloudnative:v1alpha1" \
# 指定输出文件夹, 默认是GOPATH/Src
--output-base /home/czs/
# 这个文件里面其实是开源授权说明，但如果没有这个入参，该命令无法执行
--go-header-file /home/czs/controller/hack/boilerplate.go.txt
```

生成之后代码为

```
[root@localhost controller]# tree .
.
├── deploy
│   ├── crd.yaml
│   └── vm.yaml
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
└── pkg
    ├── apis
    │   └── cloudnative
    │       ├── register.go
    │       └── v1alpha1
    │           ├── doc.go
    │           ├── register.go
    │           ├── types.go
    │           └── zz_generated.deepcopy.go
    └── client
        ├── clientset
        │   └── versioned
        │       ├── clientset.go
        │       ├── doc.go
        │       ├── fake
        │       │   ├── clientset_generated.go
        │       │   ├── doc.go
        │       │   └── register.go
        │       ├── scheme
        │       │   ├── doc.go
        │       │   └── register.go
        │       └── typed
        │           └── cloudnative
        │               └── v1alpha1
        │                   ├── cloudnative_client.go
        │                   ├── doc.go
        │                   ├── fake
        │                   │   ├── doc.go
        │                   │   ├── fake_cloudnative_client.go
        │                   │   └── fake_virtualmachine.go
        │                   ├── generated_expansion.go
        │                   └── virtualmachine.go
        ├── informers
        │   └── externalversions
        │       ├── cloudnative
        │       │   ├── interface.go
        │       │   └── v1alpha1
        │       │       ├── interface.go
        │       │       └── virtualmachine.go
        │       ├── factory.go
        │       ├── generic.go
        │       └── internalinterfaces
        │           └── factory_interfaces.go
        └── listers
            └── cloudnative
                └── v1alpha1
                    ├── expansion_generated.go
                    └── virtualmachine.go

23 directories, 32 files
```

#### 方法2

`hack`目录下创建对应的生成代码的脚本

 **`tools.go`**

```
package tools

import (
	_ "k8s.io/code-generator"
)
```

 **`update-generated.sh`**

主要的执行脚本, 生成对应代码

```
#!/usr/bin/env bash
set -o errexit
set -o nounset
set -o pipefail

SCRIPT_ROOT=$(dirname "${BASH_SOURCE[0]}")/..
CODEGEN_PKG=${CODEGEN_PKG:-$(cd "${SCRIPT_ROOT}"; ls -d -1 ./vendor/k8s.io/code-generator 2>/dev/null || echo ../code-generator)}

bash "${CODEGEN_PKG}"/generate-groups.sh "deepcopy,client,informer,lister" \
  k8s.io/sample-controller/pkg/generated k8s.io/sample-controller/pkg/apis \
  samplecontroller:v1alpha1 \
  --output-base "$(dirname "${BASH_SOURCE[0]}")/../../.." \
  --go-header-file "${SCRIPT_ROOT}"/hack/boilerplate.go.txt
```

**`verify-codegen.sh`**

通过diff检测当前代码是否已经依据apis定义的内容产生对应代码

```
#!/usr/bin/env bash
set -o errexit
set -o nounset
set -o pipefail

SCRIPT_ROOT=$(dirname "${BASH_SOURCE[0]}")/..

DIFFROOT="${SCRIPT_ROOT}/pkg"
TMP_DIFFROOT="${SCRIPT_ROOT}/_tmp/pkg"
_tmp="${SCRIPT_ROOT}/_tmp"

cleanup() {
  rm -rf "${_tmp}"
}
trap "cleanup" EXIT SIGINT

cleanup

mkdir -p "${TMP_DIFFROOT}"
cp -a "${DIFFROOT}"/* "${TMP_DIFFROOT}"

"${SCRIPT_ROOT}/hack/update-codegen.sh"
echo "diffing ${DIFFROOT} against freshly generated codegen"
ret=0
diff -Naupr "${DIFFROOT}" "${TMP_DIFFROOT}" || ret=$?
cp -a "${TMP_DIFFROOT}"/* "${DIFFROOT}"
if [[ $ret -eq 0 ]]
then
  echo "${DIFFROOT} up to date."
else
  echo "${DIFFROOT} is out of date. Please run hack/update-codegen.sh"
  exit 1
fi
```

然后执行下面命令来生成代码

```
$ go mod vendor
$ ./hack/k8s/update-generated.sh
```

## 编写Controller

在上面生成了代码之后, 下面只需要编写controller的逻辑了

在目录下新增下面目录和文件, 也可以和官方用例 simple-controller一样直接放到根目录下

```
├── cmd
│   └── main.go
├── example
│   └── test-vm.yml 
└── pkg
    ├── controller
    │   └── controller.go
    └── version
        └── version.go
```

- cmd/main.go: 控制器的主程序。
- example/test-vm.yml: 用于控制器的 VirtualMachine 的用例
- pkg/controller/controller.go: VirtualMachine 控制器核心程序。
- pkg/version/version.go:  用于Go build 时加入版本号

### controller.go

利用client-go和code-generator生成的代码来完成控制器的核心功能, 通常在写一个控制器的时候, 会建一个controller struct, 并包含下面元素

- Clientset: 控制器与Kubernetes API Server进行互动，以操作VirtualMachine资源。
- Informer:控制器的SharedInformer，用于接收API事件，并呼叫回调函数。
- InformerSynced: 确认SharedInformer的储存是否以获得至少一次完整list通知。
- Lister: 用于列出或获取缓存中的VirtualMachine资源。
- Workqueue:控制器的资源处理队列，都Informer收到事件时，会将物件推到这个队列，并在协调程序取出处理。当发生错误时，可以用于Requeue当前物件。 

```go
package controller

import (
	"context"
	"encoding/json"
	"fmt"
	"time"

	cloudnative "controller/pkg/generated/clientset/versioned"
	cloudnativeinformer "controller/pkg/generated/informers/externalversions"
	listerv1alpha1 "controller/pkg/generated/listers/cloudnative/v1alpha1"
	"github.com/golang/glog"
	"k8s.io/apimachinery/pkg/api/errors"
	utilruntime "k8s.io/apimachinery/pkg/util/runtime"
	"k8s.io/apimachinery/pkg/util/wait"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/util/workqueue"
	"k8s.io/klog"
)

const (
	resouceName = "VirtualMachine"
)

type Controller struct {
	clientset cloudnative.Interface
	informer  cloudnativeinformer.SharedInformerFactory
	lister    listerv1alpha1.VirtualMachineLister
	synced    cache.InformerSynced
	queue     workqueue.RateLimitingInterface
}

func New(clientset cloudnative.Interface, informer cloudnativeinformer.SharedInformerFactory) *Controller {
	vmInformer := informer.Cloudnative().V1alpha1().VirtualMachines()
	controller := &Controller{
		clientset: clientset,
		informer:  informer,
		lister:    vmInformer.Lister(),
		synced:    vmInformer.Informer().HasSynced,
		queue:     workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), resouceName),
	}

	vmInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: controller.enqueue,
		UpdateFunc: func(old, new interface{}) {
			controller.enqueue(new)
		},
	})
	return controller
}

func (c *Controller) Run(ctx context.Context, threadiness int) error {
	go c.informer.Start(ctx.Done())
	klog.Info("Starting the controller")
	klog.Info("Waiting for the informer caches to sync")
	if ok := cache.WaitForCacheSync(ctx.Done(), c.synced); !ok {
		return fmt.Errorf("failed to wait for caches to sync")
	}

	for i := 0; i < threadiness; i++ {
		go wait.Until(c.runWorker, time.Second, ctx.Done())
	}
	klog.Info("Started workers")
	return nil
}

func (c *Controller) Stop() {
	glog.Info("Stopping the controller")
	c.queue.ShutDown()
}

func (c *Controller) runWorker() {
	defer utilruntime.HandleCrash()
	for c.processNextWorkItem() {
	}
}

// 取数据处理
func (c *Controller) processNextWorkItem() bool {
	obj, shutdown := c.queue.Get()
	if shutdown {
		return false
	}

	err := func(obj interface{}) error {
		defer c.queue.Done(obj)
		key, ok := obj.(string)
		if !ok {
			c.queue.Forget(obj)
			utilruntime.HandleError(fmt.Errorf("Controller expected string in workqueue but got %#v", obj))
			return nil
		}

		if err := c.syncHandler(key); err != nil {
			c.queue.AddRateLimited(key)
			return fmt.Errorf("Controller error syncing '%s': %s, requeuing", key, err.Error())
		}

		c.queue.Forget(obj)
		glog.Infof("Controller successfully synced '%s'", key)
		return nil
	}(obj)

	if err != nil {
		utilruntime.HandleError(err)
		return true
	}
	return true
}

// 数据先放入缓存，再入队列
func (c *Controller) enqueue(obj interface{}) {
	key, err := cache.MetaNamespaceKeyFunc(obj)
	if err != nil {
		utilruntime.HandleError(err)
		return
	}
	c.queue.Add(key)
}

// 处理
func (c *Controller) syncHandler(key string) error {
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		utilruntime.HandleError(fmt.Errorf("invalid resource key: %s", key))
		return err
	}
    // 从缓存中取对象
	vm, err := c.lister.VirtualMachines(namespace).Get(name)
	if err != nil {
	    // 如果对象被删除了，就会走到这里，所以应该在这里加入执行
		if errors.IsNotFound(err) {
			utilruntime.HandleError(fmt.Errorf("virtualmachine '%s' in work queue no longer exists", key))
			return err
		}
		return err
	}

	data, err := json.Marshal(vm)
	if err != nil {
		return err
	}

	klog.Infof("Controller get %s/%s object: %s", namespace, name, string(data))
	return nil
}
```



### main.go 

初始化控制器的主程序, 可以提供flag来设置参数

```go
package main

import (
	"context"
	goflag "flag"
	"fmt"
	"os"
	"os/signal"
	"syscall"
	"time"

	"controller/pkg/controller"
	cloudnative "controller/pkg/generated/clientset/versioned"
	cloudnativeinformer "controller/pkg/generated/informers/externalversions"
	"controller/pkg/version"
	flag "github.com/spf13/pflag"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/klog"
)

const defaultSyncTime = time.Second * 30

var (
	kubeconfig  string
	threads     int
)

func parseFlags() {
	flag.StringVarP(&kubeconfig, "kubeconfig", "", "", "Absolute path to the kubeconfig file.")
	flag.IntVarP(&threads, "threads", "", 2, "Number of worker threads used by the controller.")
	flag.BoolVarP(&showVersion, "version", "", false, "Display the version.")
	flag.CommandLine.AddGoFlagSet(goflag.CommandLine)
	flag.Parse()
}

func restConfig(kubeconfig string) (*rest.Config, error) {
	if kubeconfig != "" {
		cfg, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
		if err != nil {
			return nil, err
		}
		return cfg, nil
	}

	cfg, err := rest.InClusterConfig()
	if err != nil {
		return nil, err
	}
	return cfg, nil
}

func main() {
	parseFlags()

	k8scfg, err := restConfig(kubeconfig)
	if err != nil {
		klog.Fatalf("Error to build rest config: %s", err.Error())
	}

	clientset, err := cloudnative.NewForConfig(k8scfg)
	if err != nil {
		klog.Fatalf("Error to build cloudnative clientset: %s", err.Error())
	}

	informer := cloudnativeinformer.NewSharedInformerFactory(clientset, defaultSyncTime)
	controller := controller.New(clientset, informer)
	ctx, cancel := context.WithCancel(context.Background())
	signalChan := make(chan os.Signal, 1)
	signal.Notify(signalChan, syscall.SIGINT, syscall.SIGTERM)

	if err := controller.Run(ctx, threads); err != nil {
		klog.Fatalf("Error to run the controller instance: %s.", err)
	}

	<-signalChan
	cancel()
	controller.Stop()
}
```

其中`restConfig（）`用于建立`RESTClient Config`，如果有指定Kubeconfig时，会透过client-go/tools/clientcmd解析Kubeconfig内容以产生Config内容；若没有的话，则表示该控制器可能被透过Pod部署在Kubernetes中，因此使用InClusterConfig方式建立Config。

### 执行

简单的可以直接执行

```
go run cmd/main.go --kubeconfig=$HOME/.kube/config -v=2 --logtostderr
```

## 高可用

kubernetes的管理组件Scheduler与Controller Manager是以Lease机制实现Active-Passive构架. 我们自己写的controller当然也要支持高可用才行

这机制的实践方式有很多种，比如基于Redis、Zookeeper、Consul、etcd，或是数据库的分布式锁（Distributed Lock）。而Kubernetes则是是采用资源锁（Resource Lock）概念来实现，基本上就是建立Kubernetes API资源ConfigMap、Endpoint或Lease来维护分布式锁的状态。

> Kubernetes从v1.15版本开始推荐使用Lease资源实现，而ConfigMap、Endpoint已经被弃用。

看一下Kubernetes Controller Manager实际运作状况，当Controller Manager被启动时，预设会透过--leader-elect=true来开启HA功能。当正确启动后，在kube-system底下，就会看到被新增了一个用于维护分布式锁状态的Endpoint资源：

```go
[root@k8s-master .kube]# kubectl -n kube-system get ep kube-controller-manager -o yaml
apiVersion: v1
kind: Endpoints
metadata:
  annotations:
    control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"k8s-master_0a23a5f6-ada5-4b09-a647-5c8a93e4d439","leaseDurationSeconds":15,"acquireTime":"2020-10-09T08:23:48Z","renewTime":"2020-10-12T09:35:20Z","leaderTransitions":15}'
  creationTimestamp: "2020-07-11T05:43:17Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:control-plane.alpha.kubernetes.io/leader: {}
    manager: kube-controller-manager
    operation: Update
    time: "2020-10-12T09:35:20Z"
  name: kube-controller-manager
  namespace: kube-system
  resourceVersion: "5911729"
  selfLink: /api/v1/namespaces/kube-system/endpoints/kube-controller-manager
  uid: 53c032fd-ffc3-42d9-9e3a-b7bf702f8e5f
```

然后可以在该资源的`metadata.annotations`看到用于储存状态的`control-plane.alpha.kubernetes.io/leader`字段。

其中`holderIdentity`用于表示当前拥有者，`acquireTime`为拥有者取得持有权的时间，`renewTime`为当前拥有者上一次活跃时间。

而更换Leader条件是当renewTime与自己当下时间计算超过leaseDurationSeconds时进行。

**Kubernetes client-go提供了Leader Election功能**,  在 client-go 中，提供了 [Leader Election Example](https://github.com/kubernetes/client-go/tree/master/examples/leader-election) 

修改main.go

```go
package main

import (
	"context"
	goflag "flag"
	"fmt"
	"os"
	"os/signal"
	"syscall"
	"time"

	"controller/pkg/controller"
	cloudnative "controller/pkg/generated/clientset/versioned"
	cloudnativeinformer "controller/pkg/generated/informers/externalversions"
	"controller/pkg/version"
	flag "github.com/spf13/pflag"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	clientset "k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/tools/leaderelection"
	"k8s.io/client-go/tools/leaderelection/resourcelock"
	"k8s.io/klog"
)

const defaultSyncTime = time.Second * 30

var (
	kubeconfig         string
	showVersion        bool
	threads            int
	leaderElect        bool
	id                 string
	leaseLockName      string
	leaseLockNamespace string
)

func parseFlags() {
	flag.StringVarP(&kubeconfig, "kubeconfig", "", "", "Absolute path to the kubeconfig file.")
	flag.IntVarP(&threads, "threads", "", 2, "Number of worker threads used by the controller.")
	flag.StringVarP(&id, "holder-identity", "", os.Getenv("POD_NAME"), "the holder identity name")
	flag.BoolVarP(&leaderElect, "leader-elect", "", true, "Start a leader election client and gain leadership before executing the main loop. ")
	flag.StringVar(&leaseLockName, "lease-lock-name", "controller101", "the lease lock resource name")
	flag.StringVar(&leaseLockNamespace, "lease-lock-namespace", "", "the lease lock resource namespace")
	flag.BoolVarP(&showVersion, "version", "", false, "Display the version.")
	flag.CommandLine.AddGoFlagSet(goflag.CommandLine)
	flag.Parse()
}

func restConfig(kubeconfig string) (*rest.Config, error) {
	if kubeconfig != "" {
		cfg, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
		if err != nil {
			return nil, err
		}
		return cfg, nil
	}

	cfg, err := rest.InClusterConfig()
	if err != nil {
		return nil, err
	}
	return cfg, nil
}

func main() {
	parseFlags()

	if showVersion {
		fmt.Fprintf(os.Stdout, "%s\n", version.GetVersion())
		os.Exit(0)
	}

	k8scfg, err := restConfig(kubeconfig)
	if err != nil {
		klog.Fatalf("Error to build rest config: %s", err.Error())
	}

	k8sclientset := clientset.NewForConfigOrDie(k8scfg)
	clientset, err := cloudnative.NewForConfig(k8scfg)
	if err != nil {
		klog.Fatalf("Error to build cloudnative clientset: %s", err.Error())
	}

	informer := cloudnativeinformer.NewSharedInformerFactory(clientset, defaultSyncTime)
	controller := controller.New(clientset, informer)
	ctx, cancel := context.WithCancel(context.Background())
	signalChan := make(chan os.Signal, 1)
	signal.Notify(signalChan, syscall.SIGINT, syscall.SIGTERM)

	if leaderElect {
		lock := &resourcelock.LeaseLock{
			LeaseMeta: metav1.ObjectMeta{
				Name:      leaseLockName,
				Namespace: leaseLockNamespace,
			},
			Client: k8sclientset.CoordinationV1(),
			LockConfig: resourcelock.ResourceLockConfig{
				Identity: id,
			},
		}
		go leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
			Lock:            lock,
			ReleaseOnCancel: true,
			LeaseDuration:   60 * time.Second,
			RenewDeadline:   15 * time.Second,
			RetryPeriod:     5 * time.Second,
			Callbacks: leaderelection.LeaderCallbacks{
				OnStartedLeading: func(ctx context.Context) {
					if err := controller.Run(ctx, threads); err != nil {
						klog.Fatalf("Error to run the controller instance: %s.", err)
					}
					klog.Infof("%s: leading", id)
				},
				OnStoppedLeading: func() {
					controller.Stop()
					klog.Infof("%s: lost", id)
				},
			},
		})
	} else {
		if err := controller.Run(ctx, threads); err != nil {
			klog.Fatalf("Error to run the controller instance: %s.", err)
		}
	}

	<-signalChan
	cancel()
	controller.Stop()
}
```

然后启动三个终端

```
# first terminal 
$ POD_NAME=test1 go run cmd/main.go --kubeconfig=$HOME/.kube/config -v=3 --logtostderr --lease-lock-namespace=default

# second terminal 
$ POD_NAME=test2 go run cmd/main.go --kubeconfig=$HOME/.kube/config -v=3 --logtostderr --lease-lock-namespace=default

# third terminal
$ POD_NAME=test3 go run cmd/main.go --kubeconfig=$HOME/.kube/config -v=3 --logtostderr --lease-lock-namespace=default

```



## 资源回收

Deployment, Job, DaemonSet等资源, 在删除时候, 其相关的Pod都会被删除, 这种机制就是kubernetes的[垃圾收集器](https://kubernetes.io/zh/docs/concepts/workloads/controllers/garbage-collection/).

但是Kubernetes的垃圾收集器仅能删除Kubernetes API的资源。Kubernetes对于删除级联资源提供了2种模式：

- Background:在这模式下，Kubernetes会直接删除属主资源，然后再由垃圾收集器在后台删除相关的API资源

- Foreground:在这模式下，Owner资源会透过设定`metadta.deletionTimestamp`字段来表示"正在删除中"。这时Owner资源依然存在于集群中，并且能透过REST API查看到相关信息。该资源被删除条件是当移除了`metadata.finalizers`字段后，才会真正的从集群中移除。这样机制形成了预删除挂钩（Pre-delete hook），因此我们能在正在删除的期间，开始回收相关的资源（如虚拟机或其他Kubernetes API资源等等），当回收完后，再将该资源删除。

  测试方法: 可以先在yaml中定义`metadata.finalizers`字段, 然后删除, 会发现kubectl卡在删除命令, 这个时候打开另外一个终端查看, 发现资源还存在, 但`metadata.deletionTimestamp`被设置了时间，这表示该资源已经处于预删除阶段, 这时候再使用`kubectl edit`把`metadata.finalizers`字段删除即可.

### Finalizer

可以在`controller.go`里面加入Finalizers机制来确保资源被正确删除, 做法也比较简单, 只需要在对应的创建函数中对其对应的资源设置`metadata.finalizers`即可

```go
func (c *Controller) syncHandler(key string) error {
    ......
	case v1alpha1.VirtualMachineTerminating:
		if err := c.deleteServer(vm); err != nil {
			return err
		}
}

func (c *Controller) createServer(vm *v1alpha1.VirtualMachine) error {
	vmCopy := vm.DeepCopy()
	ok, _ := c.vm.IsServerExist(vm.Name)
	if !ok {
		...
		addFinalizer(&vmCopy.ObjectMeta, finalizerName)
		if err := c.updateStatus(vmCopy, v1alpha1.VirtualMachineActive, nil); err != nil {
			return err
		}
	}
	return nil
}

func (c *Controller) deleteServer(vm *v1alpha1.VirtualMachine) error {
	vmCopy := vm.DeepCopy()
	if err := c.vm.DeleteServer(vmCopy.Name); err != nil {
		// Requeuing object to workqueue for retrying
		return err
	}

	removeFinalizer(&vmCopy.ObjectMeta, finalizerName)
	if err := c.updateStatus(vmCopy, v1alpha1.VirtualMachineTerminating, nil); err != nil {
		return err
	}
	return nil
}
```

> 而addFinalizer()基本上就是传入API物件的ObjectMeta（metadata）与Finalizer名称来设定。

```go
func addFinalizer(meta *metav1.ObjectMeta, finalizer string) {
	if !funk.ContainsString(meta.Finalizers, finalizer) {
		meta.Finalizers = append(meta.Finalizers, finalizer)
	}
}

func removeFinalizer(meta *metav1.ObjectMeta, finalizer string) {
	meta.Finalizers = funk.FilterString(meta.Finalizers, func(s string) bool {
		return s != finalizer
	})
}
```





## 部署到集群内

controller部署到集群的时候, 使用admin的kubeconfig文件肯定存在权限过大的问题, 这个时候就需要自己定义SA和RBAC

> 虽然Kubernetes在建立Namespace时，预设也会自动建立一个名称为default的Service Account，但这个Service Account通常会被用于该Namespace下的所有Pod，因此不建议将RBAC权限赋予给这个Service Account。

### sa.yaml

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: controller
  namespace: kube-system
```

### rbac.yaml

```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: controller-role
rules:
- apiGroups:
  - cloudnative.czs
  resources:
  - "virtualmachines"
  verbs:
  - "*"
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: controller-rolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: controller-role
subjects:
- kind: ServiceAccount
  namespace: kube-system
  name: controller
```

