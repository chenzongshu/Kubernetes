# 架构图

放一张官方架构图

![kubeedge架构图](https://docs.kubeedge.io/en/latest/_images/kubeedge_arch.png)

从官方的架构图可以清晰地看到，kubeedge整体分Cloud和Edge两部分：

1. Cloud部分 是kubernetes api server与Edge部分的桥梁，负责将kubernetes的指令下发到Edge，同时将Edge的状态和事件同步到的kubernetes api server；
2. Edge部分 接受并执行Cloud部分下发的指令，管理各种负载，并将Edge部分负载的状态和事件同步到Cloud部分；

除了官方架构图展示的Cloud和Edge部分外，还有横跨Cloud和Edge的部分，具体如下：

1. Edgemesh 基于Istio的横跨Cloud和Edge的服务网格解决方案；
2. Edgesite 为满足在边缘需要完整集群功能的场景，定制的在边缘搭建既能管理、编排又能运行负载的完整集群解决方案；

# 代码目录

代码位于GitHub ： https://github.com/kubeedge/kubeedge

| 模块     | 作用                    |      |
| -------- | ----------------------- | ---- |
| build    | 部署的一些脚本          |      |
| cloud    | cloud端各功能模块的集合 |      |
| common   |                         |      |
| edge     | edge端各功能模块的集合  |      |
| edgemesh |                         |      |
| edgesite | 边缘独立集群解决方案    |      |
| hack     |                         |      |
| keadm    | kubeedge的一键部署工具  |      |
| mappers  | 物联网协议实现包        |      |
| pkg      |                         |      |
| staging  |                         |      |

# 总体分析

- beehive是kubeedge中的一个通信框架，整个kubeEdge的cloud和edge模块都依赖于beehive框架。
- 各模块的`Register`函数调用beehive的`Register`函数将模块注册到beehive中。根据配置文件中模块是否被启用（`modules.enabled`），beehive将模块加入内部的`modules` map或者`disabledModules` map中加以管理
- edgecore或edgesite在完成模块注册之后，会调用beehive的`Run`函数启动各模块（`StartModules`）并监听信号（`GracefulShutdown`）以准备好正常关闭。