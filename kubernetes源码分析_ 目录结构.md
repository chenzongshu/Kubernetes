# kubernetes源码分析: 目录结构与编译

标签（空格分隔）： kubernetes

---

# 主目录结构

|目录|功能|
| --- | --- |
|api|输出接口文档用，基本是json源码|
|build|构建脚本|
|cmd|所有的二进制可执行文件入口代码，也就是各种命令的接口代码。|
|pkg|项目主目录，cmd只是接口，这里是具体实现。cmd类似业务代码，pkg类似核心|
|plugin|插件
|test|测试相关的工具|
|third_party|第三方工具|
|docs|文档|
|example|使用例子|
|vendor|项目依赖的Go的第三方包，比如docker客户端sdk，rest等|
|hack|工具箱，各种编译，构建，校验的脚本都在这|
|cluster|主要放一些配置脚本|

# pkg目录结构

|包名|	用途|
| :--- | ---|
|api|	kubernetes api主要包括最新版本的Rest API接口的类，并提供数据格式验证转换工具类，对应版本号文件夹下的文件描述了特定的版本如何序列化存储和网络|	
|client|	Kubernetes 中公用的客户端部分，实现对对象的具体操作增删该查操作|	
|cloudprovider|	kubernetes 提供对aws、azure、gce、cloudstack、mesos等云供应商提供了接口支持，目前包括负载均衡、实例、zone信息、路由信息等|	
|controller|	kubernetes controller主要包括各个controller的实现逻辑，为各类资源如replication、endpoint、node等的增删改等逻辑提供派发和执行|	
|credentialprovider|kubernetes credentialprovider 为docker 镜像仓库贡献者提供权限认证|	
|generated|	kubernetes generated包是所有生成的文件的目标文件，一般这里面的文件日常是不进行改动的|	
|kubectl|kuernetes kubectl模块是kubernetes的命令行工具，提供apiserver的各个接口的命令行操作，包括各类资源的增删改查、扩容等一系列命令工具|	
|kubelet|kuernetes kubelet模块是kubernetes的核心模块，该模块负责node层的pod管理，完成pod及容器的创建，执行pod的删除同步等操作等等|	
|master	|kubernetes master负责集群中master节点的运行管理、api安装、各个组件的运行端口分配、NodeRegistry、PodRegistry等的创建工作, 并启动kubernetes自身相关服务, 服务的ClusterIP分配和Port分配|	
|runtime|kubernetes runtime实现不同版本api之间的适配，实现不同api版本之间数据结构的转换|

# 编译

注意Kubernetes所需要的golang的版本, 配置好golang环境, 注意GOROOT和GOPATH是否正确

## 二级制

以1.13版本举例, 将kubernetes源码拉取到本地 `GOPATH` 工作路径，并指定好分支

```
$ mkdir -p $GOPATH/src/k8s.io && cd $GOPATH/src/k8s.io
$ git clone  https://github.com/kubernetes/kubernetes -b release-1.13
$ cd kubernetes
```

我们从 `kubernetes/hack/lib/golang.sh` 代码中可以看到，该脚本会检测当前系统环境来安装对应的必要的服务，默认不指定 `KUBE_BUILD_PLATFORMS`（当前构建平台环境类型）的话，它会分别编译多种环境，编译时间很长，而且没有必要。这里我们可以直接修改文件中 `KUBE_SERVER_PLATFORMS` `KUBE_NODE_PLATFORMS` `KUBE_CLIENT_PLATFORMS` `KUBE_TEST_PLATFORMS` 配置，只保留当前系统对应环境，例如我们的系统环境为 `linux/amd64`。另一种方式就是在编译时，直接指定 `KUBE_BUILD_PLATFORMS` 参数即可。编译命令如下：

```
$ KUBE_BUILD_PLATFORMS=linux/amd64 make all GOFLAGS=-v GOGCFLAGS="-N -l"
```

对参数简单说明一下：

- `KUBE_BUILD_PLATFORMS=linux/amd64` 指定当前编译平台环境类型为 linux/amd64。
- `make all` 表示在本地环境中编译所有组件。
- `GOFLAGS=-v` 编译参数，开启 verbose 日志。
- `GOGCFLAGS="-N -l"` 编译参数，禁止编译优化和内联，减小可执行程序大小。

注意：执行以上命令时，需要机器可用内存在 4G 左右，否则编译时会报内存不够错误。若我们只想编译某个组件，例如，只想编译 kube-apiserver ，那么可以执行 `make WHAT=cmd/kube-apiserver` 命令。这里可选组件有很多，详细可参考 `kubernetes/cmd/` 目录下所有组件。

稍等片刻，全部组件编译耗时较长，执行完毕后，二进制可执行文件默认生成到 `kubernetes/_output/bin` 目录下。

上边讲到单独对某个组件执行编译，除了上述办法之外，还可以进入到 `kubernetes/cmd/` 目录下对应组件下执行，例如我们编译 `kube-apiserver` 组件。

```
$ cd kubernetes/cmd/kube-apiserver
$ go build -v
$ ls 
apiserver.go  app  kube-apiserver  BUILD  OWNERS
```

## 镜像编译

鉴于国内环境, 可以先把核心镜像下载到本地, 主要包含几个核心基础镜像的制作，包括 `kube-cross`、`pause-amd64`、`debian-iptables-amd64`、`debian-base-amd64`、`debian-hyperkube-base-amd64`

接下来需要修改构建策略，忽略 `--pull` 参数，不然每次构建还是会去外网拉取基础镜像，要让它直接读取本地镜像。修改 `kubernetes/build/lib/release.sh` 文件如下：

```
"${DOCKER[@]}" build --pull -q -t "${docker_image_tag}" "${docker_build_path}" >/dev/null
修改为
"${DOCKER[@]}" build -q -t "${docker_image_tag}" "${docker_build_path}" >/dev/null
```

其次修改 `kubernetes/hack/lib/version.sh` 文件，将变量 `KUBE_GIT_TREE_STATE="dirty"` 修改为 `KUBE_GIT_TREE_STATE="clean"`，dirty 表示 Git 提交 ID 之后的源代码有更改，clean 表示 Git 提交 ID 之后没有更改，为了确保版本号干净，都修改为 clean。

接下来直接执行如下编译命令

```
$ KUBE_BUILD_PLATFORMS=linux/amd64 KUBE_BUILD_CONFORMANCE=n KUBE_BUILD_HYPERKUBE=n make release-images GOFLAGS=-v GOGCFLAGS="-N -l"
```

- `UBE_BUILD_CONFORMANCE=n` 和 `KUBE_BUILD_HYPERKUBE=n` 参数配置是否构建 hyperkube-amd64 和 conformance-amd64 镜像，默认是 y 构建，这里设置为 n 暂时不需要构建了。
- `make release-images` 表示执行编译并生成镜像 tar 包。

编译的 kubernetes 组件 docker 镜像以 tar 包的形式发布在 `kubernetes/_output/release-tars/amd64` 目录中。







