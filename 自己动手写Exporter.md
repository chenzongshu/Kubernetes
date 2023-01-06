# 基本概念

## exporter是什么

现在的监控体系中，Promethus应用得越来越多。

exporter就是一个应用指标的采集器，采集完对应指标之后，把数据转换成Promethus识别的metric格式，然后提供一个http server，供给Promethus采集

## Promethus指标类型

- Counter (累加指标)：Counter 一个累加指标数据，这个值随着时间只会逐渐的增加，比如程序完成的总任务数量，运行错误发生的总次数。常见的还有交换机中snmp采集的数据流量也属于该类型，代表了持续增加的数据包或者传输字节累加值。

- Gauge (测量指标)： Gauge代表了采集的一个单一数据，这个数据可以增加也可以减少，比如CPU使用情况，内存使用量，硬盘当前的空间容量等等

- Summary (概略图)

- Histogram (直方图)

这些都有Vec版本，差别在于带不带Label

# 通用写法

官方推荐简单的入门示例 ： [haproxy-exporter](https://github.com/prometheus/haproxy_exporter)

## 启动HTTP Server

```go
package main

import (
    "log"
    "net/http"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)
func main() {
    http.Handle("/metrics", promhttp.Handler())
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

`promhttp.Handler()`封装了本身的 go 运行时 metrics，并按照metircs后接value的格式在前端输出。

## 定义指标

```go
    cpuTemp = prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "cpu_temperature_celsius",
        Help: "Current temperature of the CPU.",
    })
    hdFailures = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "hd_errors_total",
            Help: "Number of hard-disk errors.",
        },
        []string{"device"},
    )
```

## 注册指标

```go
func init() {
    // Metrics have to be registered to be exposed:
    prometheus.MustRegister(cpuTemp)
    prometheus.MustRegister(hdFailures)
}
```

使用prometheus.MustRegister是将数据直接注册到Default Registry，这个Default Registry不需要额外的任何代码就可以将指标传递出去。注册后既可以在程序层面上去使用该指标了，我们可以使用之前定义的指标提供的API（Set和With().Inc）去改变指标的数据内容

## 采集更新指标

有两种方式

- 在自己代码里面定时刷新值

- 在 Prometheus 抓取的时候实时获取值

这两种方式本质上没有差别，但是在采集时有一些差别：

- 方式一可以以几乎忽略的延迟返回监控数据, 但无法保证数据是最新值.
- 方式二可能会因为一些难以获取的值而超时或者很久才会返回值

### 代码采集

在代码逻辑里面使用Set和With().Inc来改变指标数据内容，比如

```go
cpuTemp.Set(65.3)

metrics.PodsCPUEvicted.With(map[string]string{"podName": podname, "nodeName": nodeName, "namespace": namespace}).Inc()
metrics.Counts.WithLabelValues(ClusterName, AutoScalingGroupName, InstanceType, NodeName)
```

> Counter接口的定义包含了Counter本身的特性-只能增加即Inc和Add. Inc就是加1，Add可以增加任意float64

### Promethus实时采集

给exporter定义2个方法：

我们需要替这个exporter定义两个方法

- Collect: 实际收集metrics的function
- Describe: 描述metrics

这两个方法不能省略，一定要有

Describe理论上不用做什么特别的事，只要让exporter metrics呼叫Describe方法就好

而Collect则是要实作对metrics的收集

# 最佳实践

可以参看Promethus官方文档：[Writing exporters | Prometheus](https://prometheus.io/docs/instrumenting/writing_exporters/) 总结如下：

- 做到开箱即用(默认配置就可以直接开始用)
- 推荐使用 YAML 作为配置格式
- 指标使用下划线命名
- 为指标提供 HELP String (指标上的 `# HELP` 注释, 事实上这点大部分 exporter 都没做好)
- 为 Exporter 本身的运行状态提供指标
- 可以提供一个落地页

## 可配置化

可能需要采集很多指标，但是只要是指标的采集，就有可能有资源的消耗，所以必须可以通过配置来决定哪些指标可以采集和上报

## Info指标

针对指标标签(Label), 我们考虑两点: “唯一性” 和 “可读性”:

**唯一性**：对于指标, 我们应当只提供有”唯一性” 的(Label), 比如说我们暴露出 “ECS 的内存使用” 这个指标. 这时, “ECS ID” 这个标签就可以唯一区分所有的指标. 这时我们假如再加入 “IP”, “操作系统”, “名字” 这样的标签并不会增加额外的区分度, 反而会在某些状况下造成一些问题. 比方说某台 ECS 的名字变了, 那么在 Prometheus 内部就会重新记录一个时间序列, 造成额外的开销和部分 PromQL 计算的问题

```
序列A {id="foo", name="旧名字"} ..................
序列B {id="foo", name="新名字"} .................
```

**可读性**：上面的论断有一个例外, 那就是当标签涉及”可读性”时, 即使它不贡献额外的区分度, 也可以加上. 比如 “IP” 这样的标签, 假如我们只知道 ECS ID 而不知道 IP, 那么根本对不上号, 排查问题也会异常麻烦.

可以看到, 唯一性和可读性之间其实有一些权衡

答案就是 Info 指标(Info Metric). 单独暴露一个指标, 用 label 来记录实例的”额外信息”, 比如

```
ecs_info{id="foo", name="DIO", os="linux", region="hangzhou", cpu="4", memory="16GB", ip="188.188.188.188"} 1
```

这类指标的值习惯上永远为 1, 它们并记录实际的监控值, 仅仅记录 ecs 的一些额外信息. 而在使用的时候, 我们就可以通过 PromQL 的 “Join”(`group_left`) 语法将这些信息加入到最后的查询结果中

## 记录 Exporter 本身的信息

任何时候元监控(或者说自监控)都是首要的, 我们不可能依赖一个不被监控的系统去做监控. 因此了解怎么监控 exporter 并在编写时考虑到这点尤为重要.

首先, 所有的 Prometheus 抓取目标都有一个 `up` 指标用来表明这个抓取目标能否被成功抓取. 因此, **假如 exporter 挂掉或无法正常工作了**, 我们是可以从相应的 `up` 指标立刻知道并报警的.

但 `up` 成立的条件仅仅是指标接口返回 200 并且内容可以被解析, 这个粒度太粗了. 假设我们用 exporter 监控了好几个不同的模块, 其中有几个模块的指标无法正常返回了, 这时候 `up` 就帮不上忙了.

因此一个 BP 就是针对各个子模块, 甚至于各类指标, 记录细粒度的 `up` 信息, 比如阿里云 exporter 就选择了为每类指标都记录 `up` 信息:

```
aliyun_acs_rds_dashboard_MemoryUsage{id="foo"} 1233456
aliyun_acs_rds_dashboard_MemoryUsage{id="bar"} 3215123
aliyun_acs_rds_dashboard_MemoryUsage_up 1
```

当 `aliyun_acs_rds_dashboard_MemoryUsage_up` 这个指标出现 0 的时候, 我们就能知道 aliyun rds 内存信息的抓取不正常, 需要报警出来人工介入处理了.

## 设计落地页

用过 `node_exporter` 的会知道, 当访问它的主页, 也就是根路径 `/` 时, 它会返回一个简单的页面, 这就是 exporter 的落地页(Landing Page).

落地页什么都可以放, 我认为最有价值的是放文档和帮助信息(或者放对应的链接). 而文档中最有价值的莫过于对于每个指标项的说明, 没有人理解的指标没有任何价值.
