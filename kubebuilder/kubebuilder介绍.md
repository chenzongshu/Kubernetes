
# 简介

Kubebuilder 是社区推荐的创建kubernetes APIs的框架

- 提供脚手架工具初始化 CRDs 工程，自动生成 boilerplate 代码和配置；
- 提供代码库封装底层的 K8s go-client；

## GV & GVK & GVR

- GV: Api Group & Version
  - API Group 是相关 API 功能的集合
  - 每个 Group 拥有一或多个 Versions
- GVK: Group Version Kind
  - 每个 GV 都包含 N 个 api 类型，称之为 `Kinds`，不同 `Version` 同一个 `Kinds` 可能不同
- GVR: Group Version Resource
  - `Resource` 是 `Kind` 的对象标识，一般来 `Kind` 和 `Resource` 是 `1:1` 的，但是有时候存在 `1:n` 的关系，不过对于 Operator 来说都是 `1:1` 的关系

```yaml
apiVersion: apps/v1 # 这个是 GV，G 是 apps，V 是 v1
kind: Deployment    # 这个就是 Kind
sepc:               # 加上下放的 spec 就是 Resource了
  ...
```

# 安装

> kubebuilder从3.0开始，下载包只含二进制文件

- 下载二进制包并解压

```bash
curl -L -o kubebuilder https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)
chmod +x kubebuilder && mv kubebuilder /usr/local/bin/
```

> 或者从github网页  https://github.com/kubernetes-sigs/kubebuilder 

- 注意开启go module和对go mod配置proxy

```
vim ~/.profile

export GO111MODULE=on
export GOPROXY=https://goproxy.io

source ~/.profile
```

- kubebuilder还需要使用kustomize

```
go get sigs.k8s.io/kustomize/kustomize/v3@v3.8.5
```

# 使用

## 初始化

先创建一个文件夹`kubebuilder-test`

先初始化go mod

```
 go mod init czs.io
```

然后执行kubebuilder初始化命令

```
kubebuilder init --domain czs.io
kubebuilder create api --group apps --version v1 --kind Application
```

### 项目工程结构

```bash
.
├── Dockerfile
├── Makefile # 项目构建时使用
├── PROJECT  # 项目配置
├── api
│   └── v1
│       ├── application_types.go  #自定义CRD的地方
│       ├── groupversion_info.go  #GV的通用元数据用于CRD的生成,以及Schema创建方法
│       └── zz_generated.deepcopy.go #包含代码生成的runtime.Object接口实现,DeepCopy是核心
├── bin
│   └── controller-gen
├── config
│   ├── crd   # 自动生成的crd文件，不用修改这里，只需要修改了v1中的go文件之后执行make generate即可
│   │   ├── kustomization.yaml
│   │   ├── kustomizeconfig.yaml
│   │   └── patches
│   │       ├── cainjection_in_applications.yaml
│   │       └── webhook_in_applications.yaml
│   ├── default
│   │   ├── kustomization.yaml
│   │   ├── manager_auth_proxy_patch.yaml
│   │   └── manager_config_patch.yaml
│   ├── manager  #部署Controller的yaml
│   │   ├── controller_manager_config.yaml
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── prometheus
│   │   ├── kustomization.yaml
│   │   └── monitor.yaml
│   ├── rbac  #Controller运行需要的RBAC
│   │   ├── application_editor_role.yaml
│   │   ├── application_viewer_role.yaml
│   │   ├── auth_proxy_client_clusterrole.yaml
│   │   ├── auth_proxy_role.yaml
│   │   ├── auth_proxy_role_binding.yaml
│   │   ├── auth_proxy_service.yaml
│   │   ├── kustomization.yaml
│   │   ├── leader_election_role.yaml
│   │   ├── leader_election_role_binding.yaml
│   │   ├── role_binding.yaml
│   │   └── service_account.yaml
│   └── samples
│       └── apps_v1_application.yaml
├── controllers
│   ├── application_controller.go  # 自定义Controller逻辑的地方
│   └── suite_test.go
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
└── main.go  # Controller的Entrypoint
```

#### Scheme

每一组 Controllers 都需要一个 Scheme，提供了 Kinds 与对应 Go types 的映射，也就是说给定 Go type 就知道他的 GVK，给定 GVK 就知道他的 Go type，比如说我们给定一个 Scheme: "tutotial.kubebuilder.io/api/v1".CronJob{} 这个 Go type 映射到 batch.tutotial.kubebuilder.io/v1 的 CronJob GVK，那么从 Api Server 获取到下面的 JSON:

```
{
  "kind": "CronJob",
  "apiVersion": "batch.tutorial.kubebuilder.io/v1",
  ...
}
```


就能构造出对应的 Go type了，通过这个 Go type 也能正确地获取 GVR 的一些信息，控制器可以通过该 Go type 获取到期望状态以及其他辅助信息进行调谐逻辑。

#### Manager

Kubebuilder 的核心组件，具有 3 个职责：

- 负责运行所有的 Controllers；
- 初始化共享 caches，包含 listAndWatch 功能；
- 初始化 clients 用于与 Api Server 通信。

#### Cache

Kubebuilder 的核心组件，负责在 Controller 进程里面根据 Scheme 同步 Api Server 中所有该 Controller 关心 GVKs 的 GVRs，其核心是 GVK -> Informer 的映射，Informer 会负责监听对应 GVK 的 GVRs 的创建/删除/更新操作，以触发 Controller 的 Reconcile 逻辑。

#### Controller

Kubebuidler 为我们生成的脚手架文件，我们只需要实现 Reconcile 方法即可。

#### Clients

在实现 Controller 的时候不可避免地需要对某些资源类型进行创建/删除/更新，就是通过该 Clients 实现的，其中查询功能实际查询是本地的 Cache，写操作直接访问 Api Server。

#### Index

由于 Controller 经常要对 Cache 进行查询，Kubebuilder 提供 Index utility 给 Cache 加索引提升查询效率。

#### Finalizer

在一般情况下，如果资源被删除之后，我们虽然能够被触发删除事件，但是这个时候从 Cache 里面无法读取任何被删除对象的信息，这样一来，导致很多垃圾清理工作因为信息不足无法进行，K8s 的 Finalizer 字段用于处理这种情况。在 K8s 中，只要对象 ObjectMeta 里面的 Finalizers 不为空，对该对象的 delete 操作就会转变为 update 操作，具体说就是 update deletionTimestamp 字段，其意义就是告诉 K8s 的 GC“在deletionTimestamp 这个时刻之后，只要 Finalizers 为空，就立马删除掉该对象”。
所以一般的使用姿势就是在创建对象时把 Finalizers 设置好（任意 string），然后处理 DeletionTimestamp 不为空的 update 操作（实际是 delete），根据 Finalizers 的值执行完所有的 pre-delete hook（此时可以在 Cache 里面读取到被删除对象的任何信息）之后将 Finalizers 置为空即可。

#### OwnerReference

K8s GC 在删除一个对象时，任何 ownerReference 是该对象的对象都会被清除，与此同时，Kubebuidler 支持所有对象的变更都会触发 Owner 对象 controller 的 Reconcile 方法。

### 注释

在生成的代码当中我们可以看到很多 `//+kubebuilder:xxx` 开头的注释，这些注释是给对应的代码生成器服务的，在 Go 中有一个比较常用的套路就是利用 `go gennerate`生成对应的 go 代码

kubebuilder 使用 [controller-gen](https://cloudnative.to/reference/controller-gen.html) 生成代码和对应的 yaml 文件，这其中主要包含 CRD 生成、验证、处理还有 WebHook 的 RBAC 的生成功能

- CRD 生成
  - `//+kubebuilder:subresource:status` 开启 status 子资源，添加这个注释之后就可以对 `status`进行更新操作了
  - `//+groupName=nodes.lailin.xyz` 指定 groupname
  - `//+kubebuilder:printcolumn` 为 `kubectl get xxx` 添加一列，这个挺有用的
  - ……
- CRD 验证，利用这个功能，我们只需要添加一些注释，就给可以完成大部分需要校验的功能
  - `//+kubebuilder:default:=<any>` 给字段设置默认值
  - `//+kubebuilder:validation:Pattern:=string` 使用正则验证字段
  - ……
- Webhook
  - `//+kubebuilder:webhook` 用于指定 webhook 如何生成，例如我们可以指定只监听 `Update` 事件的 webhook
- RBAC 用于生成 rbac 的权限
  - `//+kubebuilder:rbac`

## 安装CRD

```
[root@localhost kubebuilder-test]# make install
/root/go/bin/controller-gen "crd:trivialVersions=true" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
kustomize build config/crd | kubectl apply -f -
customresourcedefinition.apiextensions.k8s.io/applications.apps.czs.io created

# 可以看到已有对应CRD
[root@localhost kubebuilder-test]# kubectl get crd
NAME                                                  CREATED AT
applications.apps.czs.io                              2020-10-13T07:35:26Z

```

看下生成的yaml

```
[root@localhost kubebuilder-test]# cat config/samples/apps_v1_application.yaml 
apiVersion: apps.czs.io/v1
kind: Application
metadata:
  name: application-sample
spec:
  # Add fields here
  foo: bar
```

创建对应资源

```
[root@localhost kubebuilder-test]# kubectl apply -f config/samples/
application.apps.czs.io/application-sample created
```

查看是否生成

```
[root@localhost kubebuilder-test]# kubectl get applications.apps.czs.io
NAME                 AGE
application-sample   49s
```

## 部署controller

**修改使用 gcr.io 镜像仓库的镜像地址**

对于中国大陆用户，可能无法访问 Google 镜像仓库 gcr.io，因此需要修改 `config/default/manager_auth_proxy_patch.yaml` 文件中的镜像地址，将其中 `gcr.io/kube-rbac-proxy:v0.8.0` 修改为 `registry.cn-shenzhen.aliyuncs.com/chenzongshu/kube-rbac-proxy:v0.8.0`。

有两种方式运行 controller：

- 本地运行，用于调试
- 部署到 Kubernetes 上运行，作为生产使用

**本地运行 controller**

要想在本地运行 controller，只需要执行下面的命令。

```bash
make run
```

你将看到 controller 启动和运行时输出。

**将 controller 部署到 Kubernetes**

执行下面的命令部署 controller 到 Kubernetes 上，这一步将会在本地构建 controller 的镜像，并推送到 DockerHub 上，然后在 Kubernetes 上部署 Deployment 资源。

首先需要修改 DockerFile，以方便国内下载

- 在 `COPY go.sum` 下加上 `ENV GOPROXY=https://goproxy.cn,direct`
- 将 `FROM gcr.io/distroless/static:nonroot` 替换成 `FROM golang:1.13`
- 删除 `USER nonroot:nonroot`

```bash
make docker-build docker-push IMG=chenzongshu/infra-controller  #先docker login

make deploy IMG=chenzongshu/infra-controller
/root/go/bin/controller-gen "crd:trivialVersions=true" rbac:roleName=manager-role webhook paths="./..." output:crd:artifacts:config=config/crd/bases
cd config/manager && kustomize edit set image controller=chenzongshu/infra-controller
kustomize build config/default | kubectl apply -f -
namespace/kubebuilder-test-system created
customresourcedefinition.apiextensions.k8s.io/applications.apps.czs.io configured
role.rbac.authorization.k8s.io/kubebuilder-test-leader-election-role created
clusterrole.rbac.authorization.k8s.io/kubebuilder-test-manager-role created
clusterrole.rbac.authorization.k8s.io/kubebuilder-test-proxy-role created
clusterrole.rbac.authorization.k8s.io/kubebuilder-test-metrics-reader created
rolebinding.rbac.authorization.k8s.io/kubebuilder-test-leader-election-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/kubebuilder-test-manager-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/kubebuilder-test-proxy-rolebinding created
service/kubebuilder-test-controller-manager-metrics-service created
deployment.apps/kubebuilder-test-controller-manager created

```

在初始化项目时，kubebuilder 会自动根据项目名称创建一个 Namespace，如本文中的 `kubebuilder-test-system`，查看 Deployment 对象和 Pod 资源。

```
[root@localhost kubebuilder-test]# kubectl get deployment -n kubebuilder-test-system
NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
kubebuilder-test-controller-manager   1/1     1            1           2m23s
[root@localhost kubebuilder-test]# 
[root@localhost kubebuilder-test]# kubectl -n kubebuilder-test-system get po
NAME                                                  READY   STATUS    RESTARTS   AGE
kubebuilder-test-controller-manager-855944f5b-7r92b   2/2     Running   0          3m13s
```



# 增加业务逻辑

## 修改CRD对象数据参数

查看对象的yaml文件,  可以看到这里参数有 `foo:bar`

```
[root@localhost kubebuilder-test]# cat config/samples/apps_v1_application.yaml 
apiVersion: apps.czs.io/v1
kind: Application
metadata:
  name: application-sample
spec:
  # Add fields here
  foo: bar
```

如果要增加, 需要修改`api/v1/application_types.go`文件 (比如增加一个cpu,内存信息)

```
type ApplicationSpec struct {
    Foo string `json:"foo,omitempty"`
	// 下面是增加的
	CPU    string `json:"cpu"`
	Memory string `json:"memory"`
}
// 这里可以加个状态
type ApplicationStatus struct {

}
```

后面可以修改yaml文件, 增加对应字段

## Reconcile

Reconcile 函数是 Operator 的核心逻辑, 位于`controllers/application_controller.go`

具体省略

# 删除

删除CRD

```
make uninstall
```

想要删除Controller, 没有现成的命令, 但是我们可以观察Makefile文件, 发现部署的时候是使用`kustomize`

```
# Deploy controller in the configured Kubernetes cluster in ~/.kube/config
deploy: manifests
	cd config/manager && kustomize edit set image controller=${IMG}
	kustomize build config/default | kubectl apply -f -
```

我们就使用`kustomize`来删除好了

```
kustomize build config/default | kubectl delete -f -
```

# 其他用法

其他比如crt, webhook, finalizer等用法见官方文档： https://book.kubebuilder.io/