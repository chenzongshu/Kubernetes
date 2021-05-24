# 背景

我们开发`kubernetes`应用的过程中，一般情况下是我们在本地开发调试测试完成以后，再通过`CI/CD`的方式部署到`kubernetes`的集群中，这个过程首先是非常繁琐的，而且效率非常低下，因为你想验证你的每次代码修改，就得提交代码重新走一遍`CI/CD`的流程，我们知道编译打包成镜像这些过程就是很耗时的，即使我们在自己本地搭建一套开发`kubernetes`集群，也同样的效率很低。

所以，各家都推出了应用开发神器，主要还是谷歌和微软

# Skaffold

[Skaffold](https://skaffold.dev/) 是谷歌开源的应用开发神器。它的特点：

- 没有服务器端组件，所以不会增加你的集群开销
- 自动检测源代码中的更改并自动构建/推送/部署
- 自动更新镜像**TAG**，不要担心手动去更改`kubernetes`的 manifest 文件
- 一次性构建/部署/上传不同的应用，因此它对于微服务同样完美适配
- 支持开发环境和生产环境，通过仅一次运行manifest，或者持续观察变更

## 安装

下载最新的`Linux`版本，请运行如下命令：

```shell
$ curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64 && chmod +x skaffold && sudo mv skaffold /usr/local/bin
```

下载最新的`OSX`版本，请运行：

```shell
$ curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-darwin-amd64 && chmod +x skaffold && sudo mv skaffold /usr/local/bin
```

当然如果由于某些原因你不能访问上面的链接的话，则可以前往`Skaffold`的[github release](https://github.com/GoogleCloudPlatform/skaffold/releases)页面下载相应的安装包。

## 使用

前提你有k8s集群和镜像仓库，Skaffold不必在集群里面，但是要能通过kubeconfig文件连接到集群

下面我们来通过官方示例感受下， 下载对应仓库

```
git clone https://github.com/GoogleCloudPlatform/skaffold
```

example下面有很多例子，我们来看最简单的，进入 `examples/getting-started`，里面就是一个简单的打印

```
$ tree .
.
├── Dockerfile
├── k8s-pod.yaml
├── main.go
├── skaffold-gcb.yaml
└── skaffold.yaml
```

里面有两个文件，`skaffold.yaml`表示配置文件，镜像地址修改成自己的仓库

`k8s-pod.yaml`表示k8s部署文件，也修改里面的镜像地址

```
[root@node000006 getting-started]# cat skaffold.yaml
apiVersion: skaffold/v2beta16
kind: Config
build:
  artifacts:
  - image: registry-vpc.cn-shenzhen.aliyuncs.com/chenzongshu/skaffold-example
deploy:
  kubectl:
    manifests:
      - k8s-*
      
[root@node000006 getting-started]# cat k8s-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: getting-started
spec:
  containers:
  - name: getting-started
    image: registry-vpc.cn-shenzhen.aliyuncs.com/chenzongshu/skaffold-example      
```

然后在`getting-started`目录下面执行`skaffold dev`命令就可以了，剩下的skaffold给你全部搞定

> 注意要docker login你的镜像仓库地址

```bash
[root@node000006 getting-started]# skaffold dev
Listing files to watch...
 - registry-vpc.cn-shenzhen.aliyuncs.com/chenzongshu/skaffold-example
Generating tags...
 - registry-vpc.cn-shenzhen.aliyuncs.com/chenzongshu/skaffold-example -> registry-vpc.cn-shenzhen.aliyuncs.com/chenzongshu/skaffold-example:v1.24.1-6-gff5038a-dirty
Checking cache...
 - registry-vpc.cn-shenzhen.aliyuncs.com/chenzongshu/skaffold-example: Not found. Building
Starting build...
Building [registry-vpc.cn-shenzhen.aliyuncs.com/chenzongshu/skaffold-example]...
Sending build context to Docker daemon  3.072kB
Step 1/8 : FROM golang:1.15 as builder
1.15: Pulling from library/golang
······
Successfully built b14df11ecf13
Successfully tagged registry-vpc.cn-shenzhen.aliyuncs.com/chenzongshu/skaffold-example:v1.24.1-6-gff5038a-dirty
The push refers to repository [registry-vpc.cn-shenzhen.aliyuncs.com/chenzongshu/skaffold-example]
······
Starting test...
······
[getting-started] Hello world!
[getting-started] Hello world!
```

最后两行看到，已经可以看到pod打印到标准输出的内容了，k8s里面也能看到

```bash
[root@node000006 getting-started]# kubectl get po
NAME                      READY   STATUS    RESTARTS   AGE
······
getting-started           1/1     Running   0          9m3s

[root@node000006 getting-started]# kubectl logs getting-started
Hello world!
Hello world!
```

可以得知`Skaffold`已经帮我们做了很多事情了：

- 用本地源代码构建 Docker 镜像
- 用它的`sha256`值作为镜像的标签
- 设置`skaffold.yaml`文件中定义的 kubernetes manifests 的镜像地址
- 用`kubectl apply -f`命令来部署 kubernetes 应用，然后把pod输出显示回来

然后我们来修改下程序代码的内容，修改为“hello czs”

```
[root@node000006 getting-started]# vim main.go
······
func main() {
	for {
		fmt.Println("Hello czs!")
······
```

看到输出，里面变了

```
······
[getting-started] Hello czs!
[getting-started] Hello czs!
```

有木有？！是不是让调试变得很简单

## 结束调试

调试模式下直接 `ctrl + c`， 会自动删除部署在集群内的资源，pod那些会删除掉，很方便

## 高级功能

对于应用本身的管理以及全套流程的打通，其实有很多路子，我个人觉得skaffold的价值最大就是减少了很多繁琐的调试步骤，不过我们也来看看它还有哪些功能，下面只简单介绍下，具体可以看官方文档

- 本身有三种模式，`run`、`dev`、`debug`： run模式不会热更新镜像，debug模式和dev很像，但是debug模式禁用了重新构建镜像和sync功能
- 支持其他镜像构建软件
- 镜像tag也支持多种模式：gitCommit ID、日期时间、sha256
- 也可以对接helm
- sync功能：dev模式下可以在某些文件发生改变时，直接将文件复制到容器内，从而省掉制作镜像的步骤，提供效率。这对于静态文件来说，非常有用

```
apiVersion: skaffold/v2beta16
kind: Config
build:
  artifacts:
  - image: registry-vpc.cn-shenzhen.aliyuncs.com/chenzongshu/skaffold-example
    sync:
      '**/*.txt': /
```



# Draft

微软也推出的[draft](https://draft.sh/)，Draft 让面向 Kubernetes 的应用开发变得简单。官方宣称，对于运行在 Kubernetes 上的应用，Draft 这一工具是帮助开发过程而非部署的。Draft 文档中推荐使用 Helm 进行应用部署。

他的目标是：开发人员还在开发调试之中的本地的代码，不经提交到版本控制系统，直接运行到 Kubernetes 集群上。开发人员对 Draft 发布的应用变更满意之后，才提交给版本控制系统。

Draft 不是用来在生产环境上进行部署的，他的用意就是在于快速推进面向 Kubernetes 环境的开发过程。他内部使用 Helm 来进行变更，因此他和 Helm 的集成是非常紧密的。

具体使用过程省略