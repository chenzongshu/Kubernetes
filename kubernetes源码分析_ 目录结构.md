# kubernetes源码分析: 目录结构

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








