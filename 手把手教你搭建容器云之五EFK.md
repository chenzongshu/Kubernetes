# 前言

在私有云平台中，日志系统也是重要的一部分，采集收集归纳是必不可少的，在kubernetes容器云中，我们采用标准的EFK来集成。



日志系统有多种套件，ELK（Elasticsearch、Logstash、Kibana）和EFK（Elasticsearch、Filebeat/Fluent、Kibana）, 其中ES是日志存储分析的核心引擎，在搜索领域应用也非常广泛， Logstash/Filebeat/Fluent是日志的采集端，不过Filebeat/Fluent更加轻量级，对资源消耗更小，各大公有云厂商都是自研的采集插件



在大型的日志系统中，日志量过大时，ES前端往往会集成kafka或者redis，用于缓存日志数据。在私有云中，我们只需要ES配合采集端加上Kibana显示就可以了



# ES安装



为了简化过程我们通过Helm来安装ES, 官方chart地址为  https://github.com/elastic/helm-charts ， 可以从helm的repo源下载也可以直接从github上下载chart

增加官方repo源地址

```
helm repo add elastic https://helm.elastic.co
```

下载chart

```
helm fetch elastic/elasticsearch
```

或者从github下载对应分支，这里采用github上下载的chart

```
git chone https://github.com/elastic/helm-charts.git
```

查看目录，有如下目录，里面有多个chart，包括kibana，filebeat

```
drwxrwxr-x 5 root root  4096 12月  3 17:04 apm-server
-rw-r--r-- 1 root root 10734 12月  3 17:04 CONTRIBUTING.md
drwxrwxr-x 5 root root  4096 12月  3 18:02 elasticsearch
drwxrwxr-x 5 root root  4096 12月  3 17:41 filebeat
drwxrwxr-x 4 root root  4096 12月  3 17:04 helpers
drwxrwxr-x 5 root root  4096 12月  4 15:08 kibana
-rw-r--r-- 1 root root 11357 12月  3 17:04 LICENSE
drwxrwxr-x 5 root root  4096 12月  3 17:04 logstash
-rw-r--r-- 1 root root    26 12月  3 17:04 Makefile
drwxrwxr-x 5 root root  4096 12月  3 17:04 metricbeat
-rw-r--r-- 1 root root  6898 12月  3 17:04 README.md
-rw-r--r-- 1 root root   173 12月  3 17:04 requirements.txt
```

查看es的chart, 修改value.yaml文件参数, 主要是修改卷大小, 从30G改为5G

```
volumeClaimTemplate:
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 5Gi
```

使用localpath方式创建几个PV （生产环境不能使用locallocalpath）

```
apiVersion: "v1"
kind: "PersistentVolume"
metadata:
  name: "elasticsearch-master-elasticsearch-master-0"
spec:
  capacity:
    storage: "5Gi"
  accessModes:
    - "ReadWriteOnce"
  persistentVolumeReclaimPolicy: Recycle
  hostPath:
    path: /root/esdata
```

- 使用名字自动匹配，建三个pv，分别是 xxx-0，xxx-1，xxx-2；
- 每个节点新建`/root/esdata`，**并修改权限 `chown -R 1000:1000 esdata/`**，否则会报错

安装，可以自建ns，也可以挂在default的ns下面

```
# 第一个elasticsearch是helm中的名字，第二个参数是目录
helm install elasticsearch ./es-chart/elasticsearch
```

可以看到Pod启动成功

```
[root@node000006 ~]# kubectl get po
NAME                     READY   STATUS    RESTARTS   AGE
elasticsearch-master-0   1/1     Running   0          17m
elasticsearch-master-1   1/1     Running   0          17m
elasticsearch-master-2   1/1     Running   0          17m
```

# Kibana

如果是github上下载的，自带了kibana的chart

为了方便访问kibana，把service的方式修改成nodePort方式，修改其values.yaml文件

```
service:
  type: NodePort
  loadBalancerIP: ""
  port: 5601
  nodePort: "30001"
```

然后安装

```
helm install kibana ./es-chart/kibana
```

可以看到成功运行的Pod

```
[root@node000006 ~]# kubectl get po|grep kibana
kibana-kibana-5679f8777d-bzgzr   1/1     Running   0          13m
```

# Filebeat

## 安装

如果ES和filebeat不在一个集群，需要修改es的地址，这里在一起就不修改了

```
helm install filebeat es-chart/filebeat
```

可以看到po已经运行

```
[root@node000006 ~]# kubectl get po -l app=filebeat-filebeat
NAME                      READY   STATUS    RESTARTS   AGE
filebeat-filebeat-4gxvl   1/1     Running   0          29m
filebeat-filebeat-dzcns   1/1     Running   0          29m
filebeat-filebeat-sgg8n   1/1     Running   0          29m
filebeat-filebeat-vqq6x   1/1     Running   0          29m
filebeat-filebeat-xtz98   1/1     Running   0          29m
```

查看任意一个Pod的日志，可以看到启动了文件的harvester

```
2020-12-04T07:41:43.066Z	INFO	log/harvester.go:302	Harvester started for file: /var/log/containers/dns-autoscaler-565c5546-tcknc_kube-system_autoscaler-cca6f27e730e8afcec56ade7d4e2cefb2dd41606cb166603fd65c2f733c7e510.log
2020-12-04T07:41:43.067Z	INFO	log/harvester.go:302	Harvester started for file: /var/log/containers/kube-apiserver-node1_kube-system_kube-apiserver-537540b9cedaec95cf0b2c32cdf0519b419c8c61b8c4ed5b8472390f33cc336a.log
```



## 配置

https://arch-long.cn/articles/elasticsearch/FileBeat.html

filebeat的配置文件，在Pod里面的 `/usr/share/filebeat/filebeat.yml`位置，是configmap映射过去的

```
[root@filebeat-filebeat-vqq6x /]# cat /usr/share/filebeat/filebeat.yml
filebeat.inputs:
- type: container
  paths:
    - /var/log/containers/*.log   #要监控的目录位置
  processors:
  - add_kubernetes_metadata:
      host: ${NODE_NAME}
      matchers:
      - logs_path:
          logs_path: "/var/log/containers/"

output.elasticsearch:
  host: '${NODE_NAME}'
  hosts: '${ELASTICSEARCH_HOSTS:elasticsearch-master:9200}'
```

这个配置是比较简单的，在生产上，需要增加更多的参数：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config
  namespace: logging
  labels:
    k8s-app: filebeat
    kubernetes.io/cluster-service: "true"
data:
  filebeat.yml: |-
    filebeat.config:
    #  inputs:
    #    path: ${path.config}/inputs.d/*.yml
    #    reload.enabled: true
      modules:
        path: ${path.config}/modules.d/*.yml
        reload.enabled: true

    filebeat.autodiscover:
      providers:
        - type: kubernetes
          hints.enabled: true
          include_annotations: ["artifact.spinnaker.io/name","ad.datadoghq.com/tags"]
          include_labels: ["app.kubernetes.io/name"]
          labels.dedot: true
          annotations.dedot: true
          templates:
            - condition:
                equals:
                  kubernetes.namespace: myapp   #Set the namespace in which your app is running, can add multiple conditions in case of more than 1 namespace.
              config:
                - type: docker
                  containers.ids:
                    - "${data.kubernetes.container.id}"
                  multiline:
                    pattern: '^[A-Za-z ]+[0-9]{2} (?:[01]\d|2[0123]):(?:[012345]\d):(?:[012345]\d)'.   #Timestamp regex for the app logs. Change it as per format. 
                    negate: true
                    match: after
            - condition:
                equals:
                  kubernetes.namespace: elasticsearch
              config:
                - type: docker
                  containers.ids:
                    - "${data.kubernetes.container.id}"
                  multiline:
                    pattern: '^\[[0-9]{4}-[0-9]{2}-[0-9]{2}|^[0-9]{4}-[0-9]{2}-[0-9]{2}T'
                    negate: true
                    match: after
                    
    processors:
      - add_cloud_metadata: ~
      - drop_fields:
          when:
            has_fields: ['kubernetes.labels.app']
          fields:
            - 'kubernetes.labels.app'

    output.elasticsearch:
      hosts: ['${ELASTICSEARCH_HOST:elasticsearch}:${ELASTICSEARCH_PORT:9200}']
```

- **hints.enabled**： 这将激活Filebeat的Kubernetes的hint模块。通过使用这个，我们可以使用pod注释直接将config传递给Filebeat pod。我们可以指定不同的多行模式和其他各种类型的配置。更多关于这方面的内容可以访问以下链接：https://www.elastic.co/guide/en/beats/filebeat/current/configuration-autodiscover-hints.html
- **include_annotations**： 将此设置为 “true”，可以让Filebeat保留特定日志条目的任何pod注释。这些注释可以在以后用于在Kibana控制台中过滤日志。
- **include_labels:**  将此设置为 “true”，可以让Filebeat保留特定日志条目的任何pod标签，这些标签以后可以用于在Kibana控制台中过滤日志。
- 我们还可以针对特定的命名空间过滤日志，然后可以对日志条目进行相应的处理。这里使用的是docker日志处理器。我们也可以针对不同的命名空间使用不同的多行模式。

### 参数

#### processor

filebeat对于收集的每行日志都封装成event， event 发送到 output 之前，可在配置文件中定义processors去处理 event。如果是一系列的processor，会按顺序执行。processor作用：

- 减少导出的字段
- 添加其他metadata
- 执行额外的处理和解码

processor 有两个基本构建块。`Indexers`和`Matchers`。

`Indexers` 接收pod元数据并根据pod元数据构建索引。例如：`ip_port`indexer可以使用kubernetes pod并根据所有`pod_ip:container_port`组合、索引元数据。

`Matchers` 用于构造查询索引的查找键。例如：当字段匹配器将`["metricset.host"]`作为查找字段时，它将用字段`metricset.host`的值构造一个查找键。

filebeat默认带了`add_kubernetes_metadata`的processor，详细配置见官方文档：[add_kubernetes_metadata](https://www.elastic.co/guide/en/beats/filebeat/current/add-kubernetes-metadata.html)

#### autodiscover

`autodiscover`是filebeat针对容器的配置，定义在`filebeat.yml`配置文件中。

它提供了3种provider，包括docker和kubernetes，分别针对的观测项有所不同。官方链接地址是 [autodiscover](https://www.elastic.co/guide/en/beats/filebeat/current/configuration-autodiscover.html)

#### 输出

默认是输出到es，如果我要输出到其他端，可以配置其他输出，见官方链接 [filebeat输出配置](https://www.elastic.co/guide/en/beats/filebeat/current/configuring-output.html)

例如我们要输出到kafka

```
output.kafka:
  # initial brokers for reading cluster metadata
  hosts: ["kafka1:9092", "kafka2:9092", "kafka3:9092"]

  # message topic selection + partitioning
  topic: '%{[fields.log_topic]}'
  partition.round_robin:
    reachable_only: false

  required_acks: 1
  compression: gzip
  max_message_bytes: 1000000
```

