
# 简介

Kubebuilder 是社区推荐的创建kubernetes APIs的框架

# 安装



1. 从github网页  https://github.com/kubernetes-sigs/kubebuilder 下载二进制包
2. 解压

```
tar -C /usr/local -zxvf kubebuilder_2.3.0_linux_amd64.tar.gz
mv /usr/local/kubebuilder_2.3.0_linux_amd64 /usr/local/kubebuilder
```

3. 修改环境变量

```
vim ~/.profile

export PATH=$PATH:/usr/local/kubebuilder/bin

source ~/.profile
```

4. 开启go module和对go mod配置proxy

```
vim ~/.profile

export GO111MODULE=on
export GOPROXY=https://goproxy.io

source ~/.profile
```

4. kubebuilder还需要使用kustomize

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

可以看到一堆文件

```go
├── api
│   └── v1
│       ├── application_types.go  #自定义CRD的地方
│       ├── groupversion_info.go  #GV的通用元数据用于CRD的生成,以及Schema创建方法
│       └── zz_generated.deepcopy.go #包含代码生成的runtime.Object接口实现,DeepCopy是核心
├── bin
│   └── manager
├── config
│   ├── certmanager
│   │   ├── certificate.yaml
│   │   ├── kustomization.yaml
│   │   └── kustomizeconfig.yaml
│   ├── crd   #部署CRD的yaml
│   │   ├── kustomization.yaml
│   │   ├── kustomizeconfig.yaml
│   │   └── patches
│   │       ├── cainjection_in_applications.yaml
│   │       └── webhook_in_applications.yaml
│   ├── default
│   │   ├── kustomization.yaml
│   │   ├── manager_auth_proxy_patch.yaml
│   │   ├── manager_webhook_patch.yaml
│   │   └── webhookcainjection_patch.yaml
│   ├── manager  #部署Controller的yaml
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── prometheus
│   │   ├── kustomization.yaml
│   │   └── monitor.yaml
│   ├── rbac
│   │   ├── application_editor_role.yaml
│   │   ├── application_viewer_role.yaml
│   │   ├── auth_proxy_client_clusterrole.yaml
│   │   ├── auth_proxy_role_binding.yaml
│   │   ├── auth_proxy_role.yaml
│   │   ├── auth_proxy_service.yaml
│   │   ├── kustomization.yaml
│   │   ├── leader_election_role_binding.yaml
│   │   ├── leader_election_role.yaml
│   │   └── role_binding.yaml
│   ├── samples
│   │   └── apps_v1_application.yaml  # CRD示例
│   └── webhook
│       ├── kustomization.yaml
│       ├── kustomizeconfig.yaml
│       └── service.yaml
├── controllers
│   ├── application_controller.go # 自定义Controller逻辑的地方
│   └── suite_test.go
├── Dockerfile # 制作Controller镜像
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
├── main.go # Controller的Entrypoint
├── Makefile # 项目构建时使用
└── PROJECT # 项目配置
```

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

对于中国大陆用户，可能无法访问 Google 镜像仓库 gcr.io，因此需要修改 `config/default/manager_auth_proxy_patch.yaml` 文件中的镜像地址，将其中 `gcr.io/kube-rbac-proxy:v0.5.0` 修改为 `jimmysong/kubebuilder-kube-rbac-proxy:v0.5.0`。

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