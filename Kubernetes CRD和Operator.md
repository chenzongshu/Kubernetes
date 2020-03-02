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








