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