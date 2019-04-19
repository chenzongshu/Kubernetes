# Helm

标签（空格分隔）： kubernetes

---
# 背景

每个成功的软件平台都有一个优秀的打包系统，比如 Debian、Ubuntu 的 apt，Redhat、Centos 的 yum。而 Helm 则是 Kubernetes 上的包管理器。

K8s缺乏一个高层次的应用打包工具

比如一个MySQL服务, 会需要定义一个 Service的Yaml, 让外界能够访问到 MySQL; 定义Secret的Yaml，保存MySQL 的密码; 定义PersistentVolumeClaim的Yaml，为 MySQL 申请持久化存储空间;定义一个Deployment的Yaml，部署 MySQL Pod，并使用上面的这些支持对象

可以将上面这些配置保存到对象各自的文件中，或者集中写进一个配置文件，然后通过 `kubectl apply -f` 部署

如果服务达到几十上百,而且服务之间还有前后关联的话,组织管理就很困难了.

Helm提供的功能:

- 从零创建新 chart。
- 与存储 chart 的仓库交互，拉取、保存和更新 chart。
- 在 Kubernetes 集群中安装和卸载 release。
- 更新、回滚和测试 release。

# 架构

Helm 有两个重要的概念：

- chart 
> chart 是创建一个应用的信息集合，包括各种 Kubernetes 对象的配置模板、参数定义、依赖关系、文档说明等。chart 是应用部署的自包含逻辑单元。可以将 chart 想象成 apt、yum 中的软件安装包。
- release。
> release 是 chart 的运行实例，代表了一个正在运行的应用。当 chart 被安装到 Kubernetes 集群，就生成一个 release。chart 能够多次安装到同一个集群，每次安装都是一个 release。

Helm 是包管理工具，这里的包就是指的 chart。

Helm 包含两个组件：Helm 客户端 和 Tiller 服务器

![](https://images2018.cnblogs.com/blog/775365/201804/775365-20180429075127290-274922778.png)

Helm 客户端是终端用户使用的命令行工具，用户可以：

1、在本地开发 chart。
2、管理 chart 仓库。
3、与 Tiller 服务器交互。
4、在远程 Kubernetes 集群上安装 chart。
5、查看 release 信息。
6、升级或卸载已有的 release。

Tiller 服务器运行在 Kubernetes 集群中，它会处理 Helm 

1、客户端的请求，与 Kubernetes API Server 交互。Tiller 服务器负责：
2、监听来自 Helm 客户端的请求。
3、通过 chart 构建 release。
4、在 Kubernetes 中安装 chart，并跟踪 release 的状态。
5、通过 API Server 升级或卸载已有的 release。

简单的讲：**Helm 客户端负责管理 chart；Tiller 服务器负责管理 release。**

# 部署

## 客户端

通常，我们将 Helm 客户端安装在能够执行 kubectl 命令的节点上，只需要下面一条命令：

```
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get | bash
```

执行 `helm version` 验证

## Tiller 服务器

Tiller 服务器安装非常简单，只需要执行 `helm init`

Tiller 本身也是作为容器化应用运行在 Kubernetes Cluster 中的

# 使用

Helm 安装成功后，可执行 `helm search` 查看当前可安装的 chart

Helm 安装时已经默认配置好了两个仓库：`stable` 和 `local`。`stable` 是官方仓库，`local` 是用户存放自己开发的 chart 的本地仓库。

`helm search` 会显示 chart 位于哪个仓库，比如 `local/cool-chart` 和 `stable/acs-engine-autoscaler`。

用户可以通过 `helm repo add` 添加更多的仓库，比如企业的私有仓库

## 安装chart

```
helm install stable/mysql
```

如果有报错:`no available release name found`,说明可能tiller没权限

```
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```

# chart

chart 是 Helm 的应用打包格式。chart 由一系列文件组成，这些文件描述了 Kubernetes 部署应用时所需要的资源，比如 Service、Deployment、PersistentVolumeClaim、Secret、ConfigMap 等。

chart 将这些文件放置在预定义的目录结构中，通常整个 chart 被打成 tar 包，而且标注上版本信息，便于 Helm 部署。

## chart目录结构

一旦安装了某个 chart，我们就可以在 ~/.helm/cache/archive 中找到 chart 的 tar 包。解压之后可以看到目录

```
examples/
  Chart.yaml   # Yaml文件，用于描述Chart的基本信息，包括名称版本等
  LICENSE      # [可选] 协议
  README.md    # [可选] 当前Chart的介绍
  values.yaml         # Chart的默认配置文件
  requirements.yaml   # [可选] 用于存放当前Chart依赖的其它Chart的说明文件
  charts/             # [可选]: 该目录中放置当前Chart依赖的其它Chart
  templates/          # [可选]: 部署文件模版目录，模版使用的值来自values.yaml和由Tiller提供的值
  templates/NOTES.txt # [可选]: 放置Chart的使用指南
```

### Chart.yaml

```
name: [必须] Chart的名称
version: [必须] Chart的版本号，版本号必须符合 SemVer 2：http://semver.org/
description: [可选] Chart的简要描述
keywords:
  -  [可选] 关键字列表
home: [可选] 项目地址
sources:
  - [可选] 当前Chart的下载地址列表
maintainers: # [可选]
  - name: [必须] 名字
    email: [可选] 邮箱
engine: gotpl # [可选] 模版引擎，默认值是gotpl
icon: [可选] 一个SVG或PNG格式的图片地址
```

### requirements.yaml 和 charts目录

Chart支持两种方式表示依赖关系，可以使用requirements.yaml或者直接将依赖的Chart放置到charts目录中。

```
dependencies:
  - name: example
    version: 1.2.3
    repository: http://example.com/charts
  - name: Chart名称
    version: Chart版本
    repository: 该Chart所在的仓库地址
```

### templates 目录

templates目录中存放了Kubernetes部署文件的模版。

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: deis-database
  namespace: deis
  labels:
    heritage: deis
spec:
  replicas: 1
  selector:
    app: deis-database
  template:
    metadata:
      labels:
        app: deis-database
    spec:
      serviceAccount: deis-database
      containers:
        - name: deis-database
          image: {{.Values.imageRegistry}}/postgres:{{.Values.dockerTag}}
          imagePullPolicy: {{.Values.pullPolicy}}
          ports:
            - containerPort: 5432
          env:
            - name: DATABASE_STORAGE
              value: {{default "minio" .Values.storage}}
```

模版语法扩展了 golang/text/template的语法















