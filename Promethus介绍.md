# 简介

Prometheus是新一代的云原生监控系统，受启发于Google的Brogmon监控系统。

Prometheus可以用二进制部署，自带UI和PromQL，数据保存在TSDB中。

相比于其他传统监控工具主要有以下几个特点：

- 具有由 metric 名称和键/值对标识的时间序列数据的多维数据模型
- 有一个灵活的查询语言
- 不依赖分布式存储，只和本地磁盘有关
- 通过 HTTP 的服务拉取时间序列数据
- 也支持推送的方式来添加时间序列数据
- 还支持通过服务发现或静态配置发现目标
- 多种图形和仪表板支持

## 监控数据模型

Prometheus 的数据模型支持多维自定义，这个模型下的时序数据有metric名和一组key/value标签组成。 Prometheus的存储都是按时间序列去实现的，相同的metrics名和label组成一条时间序列，如果label不同表示不同的时间序列。

```
<metric name>{<label name>=<label value>, ...} 
```

例如：`api_http_requests_total{method="POST", handler="/messages"}`

- `metric name`即指标名称：我们需要给监测对象的指标取一个名字，例如`api_http_requests_total`。
- `label`即标签，表示对一条时间序列不同维度的识别，例如对于`api_http_requests_total{method="POST", handler="/messages"}`中有method表示请求用的是GET还是POST

相同指标名称的数据模型无论增加或减少标签都会形成一条新的时间序列。对应关系型数据库来理解时序数据：

- 指标名称对应表名
- 标签对应表的字段，另外指标也是一个字段
- timestamp是表数据的主键
- 指标的值就是指标字段数据的值

整个流程如下：

```
┌──────────────────────────┐
│           服务发现        │
└──────────────────────────┘
              │             
              ▼             
┌──────────────────────────┐
│             配置          │
└──────────────────────────┘
              │             
              ▼             
┌──────────────────────────┐
│  重新标记relabel_configs   │
└──────────────────────────┘
              │             
              ▼             
┌──────────────────────────┐
│            抓取           │
└──────────────────────────┘
              │             
              ▼             
┌─────────────────────────────┐
│重新标记metric_relabel_configs│
└─────────────────────────────┘
```

# 安装

有两种方式可以安装Promethus相关的，

一种是二进制安装，可以通过github或者官网 https://prometheus.io/download/ 来下载

一种是通过operator安装，这种方式有个开源项目[kube-Promethus](https://github.com/prometheus-operator/kube-prometheus) 

## 配置

Promethus可以热加载其配置文件和规格文件，前提是你设置`--web.enable-lifecycle`为enable，并且发送一个`SIGHUP`信号到promethus进程或者发送一个HTTP POST 请求到`/-/reload` endpoint 

Promethus启动的时候可以通过参数指定配置和数据库，默认情况下，Promethus进程会去读同目录下面的配置文件`promethes.yml`

Promtheus作为一个时间序列数据库，其采集的数据会以文件的形式存储在本地中，默认的存储路径为`data/`，也可以通过参数`--storage.tsdb.path="data/"`修改本地数据存储的路径

可以用`./prometheus -h`命令查看配置参数

Promethus的配置文件是yaml格式的，二进制包里面会带一个默认的配置文件：

```
global:
  scrape_interval:     15s
  evaluation_interval: 15s

rule_files:
  # - "first.rules"
  # - "second.rules"

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']
```

上面这个配置文件中包含了3个模块：`global`、`rule_files` 和 `scrape_configs`。

- `global` 模块控制 `Prometheus Server` 的全局配置：
  - `scrape_interval`：表示 prometheus 抓取指标数据的频率，默认是15s，我们可以覆盖这个值
  - `evaluation_interval`：用来控制评估规则的频率，prometheus 使用规则产生新的时间序列数据或者产生警报
- `rule_files`：指定了报警规则所在的位置，prometheus 可以根据这个配置加载规则，用于生成新的时间序列数据或者报警信息，当前我们没有配置任何报警规则。
- `scrape_configs` 用于控制 prometheus 监控哪些资源。

### scrape_configs

#### 服务发现

static_configs块可以实现服务发现机制

##### 静态服务发现

```yaml
scrape_configs：
  - job_name: 'node'
      static_configs:
        - targets: ['138.197.26.35:9100','138.197.26.36:9100','138.197.26.37:9100']
```

这种方式每次都要维护一个列表，手动添加，太麻烦

##### 基于文件的服务发现

创建targets和每个作业的目录，然后使用JSON或者Yaml文件格式，包含定义的目标列表，和静态服务一样即可

```yaml
scrape_configs：
  - job_name: 'node'
      file_sd_configs:
        - targets/node/*.json
        refresh_interval: 5m
```

JSON格式：

```json
[{
  "targets": [
    '138.197.26.35:9100',
    '138.197.26.36:9100',
    '138.197.26.37:9100'
  ]
}]
```

或者YAML格式

```yaml
  - targets: 
    - '138.197.26.35:9100'
    - '138.197.26.36:9100'
    - '138.197.26.37:9100'
```

##### 基于API的服务发现

一些平台或者工具内置支持prometheus的服务发现，比如aws，azure，consul、Kubernetes

下面我们重点来讲讲Kubernetes的服务发现

###### Kubernetes服务发现

在 Kubernetes 下，Promethues 通过与 Kubernetes API 集成，主要支持5中服务发现模式，分别是：`Node`、`Service`、`Pod`、`Endpoints`、`Ingress`。 见官方文档： [kubernetes_sd_config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config)

官方模板解析

```yaml
# 在集群可以为空，不然需要指定APIServer地址和端口
[ api_server: <host> ]

# 只能为endpoints, service, pod, node, or ingress
role: <string>

# 认证方式，任选一种，bearer_token和bearer_token_file任选一种，需要base64解码之后的
basic_auth:
  [ username: <string> ]
  [ password: <secret> ]
  [ password_file: <string> ]

[ bearer_token: <secret> ]

[ bearer_token_file: <filename> ]

# Optional proxy URL.
[ proxy_url: <string> ]

# HTTPS需要CA文件，
tls_config:
  [ <tls_config> ]

# 可以指定namespaces，不指定就是全部
namespaces:
  names:
    [ - <string> ]

[ selectors:
  [ - role: <string>
    [ label: <string> ]
    [ field: <string> ] ]]
```

有个实际例子，prometheus以二进制部署在集群外，写一个cadvisor的job

```yaml
scrape_configs:
- job_name: k8s-cadvisor
  honor_timestamps: true
  metrics_path: /metrics
  scheme: https
  kubernetes_sd_configs:  # kubernetes 自动发现
  - api_server: https://10.130.126.144:6443  # apiserver 地址
    role: node  # node 类型的自动发现
    bearer_token_file: token
    tls_config:
      insecure_skip_verify: true
  bearer_token_file: token
  tls_config:
    insecure_skip_verify: true
  relabel_configs:
  - action: labelmap
    regex: __meta_kubernetes_node_label_(.+)
  - separator: ;
    regex: (.*)
    target_label: __address__
    replacement: 10.130.126.144:6443
    action: replace
  - source_labels: [__meta_kubernetes_node_name]
    separator: ;
    regex: (.+)
    target_label: __metrics_path__
    replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
    action: replace
```

##### 基于DNS的服务发现

从DNS发现服务，默认是SRV DNS记录查询，支持A或者AAAA记录

```yaml
scrape_configs：
  - job_name: 'webapp'
      dns_sd_configs:
        - names: ['example.com']
          type: A
          port: 9100
```

## 二进制

下载二进制包之后，解压，得到如下

```
drwxr-xr-x  2 3434 3434     4096 2月  18 00:11 console_libraries
drwxr-xr-x  2 3434 3434     4096 2月  18 00:11 consoles
drwxr-xr-x 10 root root     4096 3月   5 11:00 data
-rw-r--r--  1 3434 3434    11357 2月  18 00:11 LICENSE
-rw-r--r--  1 3434 3434     3420 2月  18 00:11 NOTICE
-rwxr-xr-x  1 3434 3434 91044140 2月  17 22:19 prometheus
-rw-r--r--  1 3434 3434     1473 3月   4 15:40 prometheus.yml
-rwxr-xr-x  1 3434 3434 80948693 2月  17 22:21 promtool
```

promtool是Prometheus附带的一个代码校验工具，可以验证配置文件

## kube-promethus

快速部署的话可以直接使用下面命令

```bash
git clone https://github.com/coreos/kube-prometheus.git
cd kube-prometheus
# Create the namespace and CRDs, and then wait for them to be availble before creating the remaining resources
kubectl create -f manifests/setup
until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done
kubectl create -f manifests/
```

默认会部署了下面的组件：

- prometheus-operator: 让 prometheus 更好的适配 k8s，可直接通过创建 k8s CRD 资源来创建 prometheus 与 alertmanager 实例及其监控告警规则 (默认安装时也会创建这些 CRD 资源，也就是会自动部署 prometheus 和 alertmanager，以及它们的配置)
- prometheus-adapter: 让 prometheus 采集的监控数据来适配 k8s 的 [resource metrics API](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/resource-metrics-api.md) 和 [custom metrics API](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/instrumentation/custom-metrics-api.md)，`kubectl top` 和 HPA 功能都依赖它们。
- node-exporter: 以 DaemonSet 方式部署在每个节点，将节点的各项系统指标暴露成 prometheus 能识别的格式，以便让 prometheus 采集。
- kube-state-metrics: 将 k8s 的资源对象转换成 prometheus 的 metrics 格式以便让 prometheus 采集，比如 Node/Pod 的各种状态。
- grafana: 可视化展示监控数据的界面，并且带了一些监控模板。

# 磁盘数据存储模型

prometheus每2小时数据落盘一次，每次落盘会保留如下类似`01EZY7E62AE49C7GFEFDCX3C7K`的一个文件夹

```
[root@node000006 data]# ls -l
总用量 32
drwxr-xr-x 3 root root  4096 3月   4 17:00 01EZY7EMVXWY2PMPMMEKNM975Y
drwxr-xr-x 3 root root  4096 3月   4 19:00 01EZYEABZD22JK74TDW6KFSCX3
drwxr-xr-x 2 root root  4096 3月   4 19:00 chunks_head
-rw-r--r-- 1 root root     0 3月   1 10:31 lock
-rw-r--r-- 1 root root 20001 3月   4 16:40 queries.active
drwxr-xr-x 3 root root  4096 3月   4 19:00 wal
```

里面的结构如下：

```
[root@node000006 data]# ls -l 01EZYEABZD22JK74TDW6KFSCX3/
总用量 3024
drwxr-xr-x 2 root root    4096 3月   4 19:00 chunks
-rw-r--r-- 1 root root 3080769 3月   4 19:00 index
-rw-r--r-- 1 root root     280 3月   4 19:00 meta.json
-rw-r--r-- 1 root root       9 3月   4 19:00 tombstones
```

- 调用API删除数据的时候，实际并没有删除记录，只是在`tombstones`里面加了一个删除记录
- 内存中会保存2小时数据，并且会把该2小时数据写入wal文件（128M每Segment），如果进程奔溃则从wal从读取出来恢复数据

# 应用监控

 Prometheus 的数据指标是通过一个公开的 HTTP(S) 数据接口获取到的，我们不需要单独安装监控的 agent，只需要暴露一个 metrics 接口，Prometheus 就会定期去拉取数据；对于一些普通的 HTTP 服务，我们完全可以直接重用这个服务，添加一个 `/metrics` 接口暴露给 Prometheus；而且获取到的指标数据格式是非常易懂的，不需要太高的学习成本。

现在很多服务从一开始就内置了一个 `/metrics` 接口，比如 Kubernetes 的各个组件、istio 服务网格都直接提供了数据指标接口。有一些服务即使没有原生集成该接口，也完全可以使用一些 `exporter` 来获取到指标数据，比如 `mysqld_exporter`、`node_exporter`，这些 `exporter` 就有点类似于传统监控服务中的 agent，作为服务一直存在，用来收集目标服务的指标数据然后直接暴露给 Prometheus。

> 一般应用类的exporter可以部署成sidecar，和应用打包在一个Pod里面

这就是应用监控的两种姿势，官方有一个已经提供的 exporter 的列表： 见地址 [EXPORTERS AND INTEGRATIONS](https://prometheus.io/docs/instrumenting/exporters/)

# 容量规划

## 内存

Prometheus在内存中做了很多工作。每个收集的时间序列、查询和记录规则都会消耗进程内存。 关于Prometheus的容量规划的参考数据并不多（特别是自2.0版本发布以来），但一个有用的、粗略的 经验法则是将每秒采集的样本数乘以样本的大小。我们可以使用以下查询语句来查看样本收集率。

```
rate(prometheus_tsdb_head_samples_appended_total[1m])
```

这将显示你在最后一分钟添加到数据库的每秒样本率。如果想知道收集的指标数量，则可以使用 以下语句：

这里使用sum聚合来计算所有匹配的指标的计数和，使用=~运算符和.+的正则表达式来匹配所有 指标。

```
sum( count by (__name__)({__name__=\~"\.\+"}) )
```

每个样本的大小通常为1到2个字节，让我们谨慎一点，按照2个字节计算。假设在12小时内每秒收 集100 000个样本，那我们可以像下面这样计算内存使用情况：

```
100 000 * 2bytes * 43200 seconds
```

结果大概是8.64GB的内存。

你还需要考虑在查询和记录规则方面的内存使用情况。这个不太好计算，并且依赖于许多其他变 量，建议根据内存使用情况灵活调整。你可以通过检查process_resident_memory_bytes指标来查看 Prometheus进程的内存使用情况。

## 磁盘

磁盘使用量受存储的时间序列数量和这些时间序列的保留时间限制。默认情况下，指标会在本地 时间序列数据库中存储15天。数据库的位置和保留时间由命令行选项控制。

- `--storage.tsdb.path`选项：它的默认数据目录位于运行Prometheus的目录中，用于控制时间序列数 据库位置。 

- `--storage.tsdb.retention`选项：控制时间序列的保留期。默认值为15d，代表15天

对于每秒10万个样本的示例，我们知道按时间序列收集的每个样本在磁盘上占用大约1到2个字 节。假设每个样本有2个字节，那么保留15天的时间序列意味着需要大约259 GB的磁盘。

> 建议采用SSD作为时间序列数据库的磁盘。