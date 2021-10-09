# 前言

下面来使用kubebuilder实际动手写一个控制器，省略kubebuilder介绍和安装过程

# 初始化

新建项目文件夹，然后初始化，域名选

```bash
chenzongshu@chenzongshudeMacBook-Pro k8s-inspect % go mod init myhll.cn
go: creating new go.mod: module myhll.cn
chenzongshu@chenzongshudeMacBook-Pro k8s-inspect % kubebuilder init --domain myhll.cn
```

创建API

```bash
kubebuilder create api --group apps --version v1 --kind Application
```

# 逻辑编写

## Status

`api/v1/application_types.go`里面定义了CRD的`schema`和对应的`Spec`还有`Status`，三个结构体

- `Status`：真实状态，比如正在运行的deployment的Pod数量
- `Spec`：期望状态，例如deployment在创建时指定了pod有三个副本

## 增加权限

Reconcile方法前面有一些+kubebuilder:rbac前缀的注释，这些是用来确保controller在运行时有对应的资源操作权限，这里我们增加Deployment和Pod的读权限

```
//+kubebuilder:rbac:groups=apps,resources=deployment,verbs=get;list;watch
func (r *ApplicationReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
```

