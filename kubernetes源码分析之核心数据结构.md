# 资源

看一个常见的yaml模板

```yaml
# CronJob就是这个 API 对象的资源类型（Resource）
# batch就是它的组（Group）
# v2alpha1 就是它的版本（Version）
apiVersion: batch/v2alpha1 
kind: CronJob
...
```

Kubernetes将资源再次分组和版本化，形成Group（资源组）、Version（资源版本）、Resource（资源）、Kind（资源种类）

- **Group** ：被称为资源组，在Kubernetes API Server中也可称其为APIGroup。

- **Version** ：被称为资源版本，在Kubernetes API Server中也可称其为APIVersions。

- **Resource** ：被称为资源，在Kubernetes API Server中也可称其为APIResource。

- **Kind** ：资源种类，描述Resource的种类，与Resource为同一级别。比如Deployment

Kubernetes系统支持多个Group，每个Group支持多个Version，每个Version支持多个Resource，其中部分资源同时会拥有自己的子资源（即SubResource）。例如，Deployment资源拥有Status子资源。

> resources（资源） 只是 API 中的一个 Kind 的使用方式。通常情况下，Kind 和 resources 之间有一个一对一的映射。 例如，`pods` 资源对应于 `Pod` 种类。但是有时，同一类型可能由多个资源返回。例如，`Scale` Kind 是由所有 `scale` 子资源返回的，如 `deployments/scale` 或 `replicasets/scale`。这就是允许 Kubernetes HorizontalPodAutoscaler(HPA) 与不同资源交互的原因。然而，使用 CRD，每个 Kind 都将对应一个 resources。

资源组、资源版本、资源、子资源的完整表现形式为`<group>/<version>/<resource>/<subresource>`。以常用的Deployment资源为例，`apps/v1/deployments/status`

> 可以通过 Group、Version、Resource、Kind 任意组合的数据结构来明确标识一个资源
> 常见的资源结构如下： GVR, GV, GR, GVK, GV, GK, GVS。



## 版本

| 版本       | 稳定性   | 是否开启 | 命名规则                     |
| ---------- | -------- | -------- | ---------------------------- |
| **Alpha**  | 不稳定   | 默认禁用 | v1alpha1、v1alpha2、v2alpha1 |
| **Beta**   | 相对稳定 | 默认开启 | v1beta1、v1beta2、v2beta1    |
| **Stable** | 正式版本 | 默认开启 | v1、v2、v3                   |



## 内部版本与外部版本

每一个资源都至少有两个版本，分别是外部版本（External Version）和内部版本（Internal Version）。外部版本用于对外暴露给用户请求的接口所使用的资源对象。内部版本不对外暴露，仅在Kubernetes API Server内部使用

资源的外部版本代码定义在`pkg/apis/<group>/<version>/`目录下，资源的内部版本代码定义在`pkg/apis/<group>/`目录下。例如，Deployment资源，它的外部版本定义在`pkg/apis/apps/{v1，v1beta1，v1beta2}/`目录下，它的内部版本定义在`pkg/apis/apps/`目录下（内部版本一般与资源组在同一级目录下）

```go
// 省略掉了一些文件，最外层的就是内部版本的资源代码
├── BUILD
├── doc.go
├── fuzzer
│   └── fuzzer.go
├── install
│   └── install.go
├── register.go
├── types.go
├── v1
│   ├── conversion.go
│   ├── defaults.go
│   ├── doc.go
│   ├── register.go
│   ├── zz_generated.conversion.go
│   └── zz_generated.defaults.go
├── v1beta1
······
├── v1beta2
······
├── validation
│   ├── validation.go
└── zz_generated.deepcopy.go
```

- **doc.go** ：GoDoc文件，定义了当前包的注释信息。在Kubernetes资源包中，它还担当了代码生成器的全局Tags描述文件。
- **register.go** ：定义了资源组、资源版本及资源的注册信息。
- **types.go** ：定义了在当前资源组、资源版本下所支持的资源类型。
- **v1** 、**v1beta1** 、**v1beta2** ：定义了资源组下拥有的资源版本的资源（即外部版本）。
- **install** ：把当前资源组下的所有资源注册到资源注册表中。
- **validation** ：定义了资源的验证方法。
- **zz_generated.deepcopy.go** ：定义了资源的深复制操作，该文件由代码生成器自动生成。

通过register.go代码文件定义所属的资源组和资源版本，内部版本资源对象通过**`runtime.APIVersionInternal`**（即__internal）标识

外部版本的资源代码多了几个文件，含义如下：

-  **conversion.go** ：定义了资源的转换函数（默认转换函数），并将默认转换函数注册到资源注册表中。
-  **zz_generated.conversion.go** ：定义了资源的转换函数（自动生成的转换函数），并将生成的转换函数注册到资源注册表中。该文件由代码生成器自动生成。
-  **defaults.go** ：定义了资源的默认值函数，并将默认值函数注册到资源注册表中。
-  **zz_generated.defaults.go** ：定义了资源的默认值函数（自动生成的默认值函数），并将生成的默认值函数注册到资源注册表中。该文件由代码生成器自动生成。

外部版本与内部版本资源类型相同，都通过register.go代码文件定义所属的资源组和资源版本，外部版本资源对象通过资源版本（Alpha、Beta、Stable）标识

## 资源对象描述文件

一个资源对象需要用5个字段来描述它，分别是Group/Version、Kind、MetaData、Spec、Status。这些字段定义在YAML或JSON文件中，这就是我们常用来部署的yaml文件了。

- **apiVersion** ：指定创建资源对象的资源组和资源版本，其表现形式为`<group>/<version>`，若是core资源组（即核心资源组）下的资源对象，其表现形式为`<version>`。
- **kind** ：指定创建资源对象的种类。
- **metadata** ：描述创建资源对象的元数据信息，例如名称、命名空间等。
- **spec** ：包含有关Deployment资源对象的核心信息，告诉Kubernetes期望的资源状态、副本数量、环境变量、卷等信息。
- **status** ：包含有关正在运行的Deployment资源对象的信息。这是由Kubernetes系统提供和更新的

# Schema资源注册表

**`legacyscheme.Scheme`**是kube-apiserver组件的全局资源注册表，Kubernetes的所有资源信息都交给资源注册表统一管理。`core.AddToScheme`函数注册core资源组内部版本的资源。`v1.AddToScheme`函数注册core资源组外部版本的资源。`scheme.SetVersionPriority`函数注册资源组的版本顺序，如有多个资源版本，排在最前面的为资源首选版本。

Scheme资源注册表支持两种资源类型（Type）的注册，分别是UnversionedType和KnownType资源类型

- **UnversionedType** ：无版本资源类型，这是一个早期Kubernetes系统中的概念，它主要应用于某些没有版本的资源类型，该类型的资源对象并不需要进行转换。在目前的Kubernetes发行版本中，无版本类型已被弱化，几乎所有的资源对象都拥有版本，但在metav1元数据中还有部分类型，它们既属于meta.k8s.io/v1又属于UnversionedType无版本资源类型，例如metav1.Status、metav1.APIVersions、metav1.APIGroupList、metav1.APIGroup、metav1.APIResourceList。

- **KnownType** ：是目前Kubernetes最常用的资源类型，也可称其为“拥有版本的资源类型”。

在Scheme资源注册表中，UnversionedType资源类型的对象通过`scheme.AddUnversionedTypes`方法进行注册，KnownType资源类型的对象通过`scheme.AddKnownTypes`方法进行注册。

## Scheme资源注册表数据结构

Scheme资源注册表数据结构主要由4个map结构组成，它们分别是gvkToType、typeToGVK、unversionedTypes、unversionedKinds

```go
file:// k8s.io/apimachinery/pkg/runtime/scheme.go
type Scheme struct {
	gvkToType map[schema.GroupVersionKind]reflect.Type
	typeToGVK map[reflect.Type][]schema.GroupVersionKind
	unversionedTypes map[reflect.Type]schema.GroupVersionKind
	unversionedKinds map[string]reflect.Type
	fieldLabelConversionFuncs map[schema.GroupVersionKind]FieldLabelConversionFunc
	defaulterFuncs map[reflect.Type]func(interface{})
	converter *conversion.Converter
	versionPriority map[string][]string
	observedVersions []schema.GroupVersion
	schemeName string
}
```

- **gvkToType** ：存储GVK与Type的映射关系。

- **typeToGVK** ：存储Type与GVK的映射关系，一个Type会对应一个或多个GVK。

- **unversionedTypes** ：存储UnversionedType与GVK的映射关系。

- **unversionedKinds** ：存储Kind（资源种类）名称与UnversionedType的映射关系。

Scheme资源注册表通过Go语言的map结构实现映射关系，这些映射关系可以实现高效的正向和反向检索，从Scheme资源注册表中检索某个GVK的Type，它的时间复杂度为*O* （1）

