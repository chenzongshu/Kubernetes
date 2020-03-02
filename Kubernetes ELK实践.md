# 概述

对于Kubernetes集群来说, 一个集中式的日志系统而言是必须的, 现在Kubernetes相关的就是ELK(EFK), 但是对于不同规模的集群, 使用的形式应该不同

对于日志量较小的私有集群(日均日志10G以下), 可以把ELK放到集群内部, 这是Kubernetes官方自带的方式, 通过statusfulset来部署es的, 具体yaml文件见: https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/fluentd-elasticsearch

对于大型Kubernetes集群, ES集群应该是独立的集群, 而且是高可用设置的, 这种方案网上有很多详细描述, 这种不重复.

# elk一体化镜像

网上找到了一些文章, 有作者提供了 sebp/elk 一体化镜像, 包含了elk三个组件, 具体使用文档可见 https://elk-docker.readthedocs.io/

问题:
1. filebeat.yml使用的服务器地址必须使用域名，不能使用IP地址，否则会报错
2. 自带证书

