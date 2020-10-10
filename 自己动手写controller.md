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