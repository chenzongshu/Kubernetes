网上很多资料都是很简单的client-go的应用，下面来介绍下怎么获取pod或者node实际cpu和内存数据

# 前置安装

必须要安装kube-state-metrics和metric-server

# API信息查询



```bash
# all node metrics; type []NodeMetrics
/apis/metrics.k8s.io/v1beta1/nodes

# metrics for a specified node; type NodeMetrics
/apis/metrics.k8s.io/v1beta1/nodes/{node}

# all pod metrics within namespace with support for all-namespaces; type []PodMetrics
/apis/metrics.k8s.io/v1beta1/namespaces/{namespace}/pods

# metrics for a specified pod; type PodMetrics
/apis/metrics.k8s.io/v1beta1/namespaces/{namespace}/pods/{pod}
```

先做本地的API转发

```go
$ kubectl proxy
Starting to serve on 127.0.0.1:8001
```

然后查询查询对应的API和返回信息

```go
$ curl 127.0.0.1:8001/apis/metrics.k8s.io/v1beta1/nodes
{
  "kind": "NodeMetricsList",
  "apiVersion": "metrics.k8s.io/v1beta1",
  "metadata": {
    "selfLink": "/apis/metrics.k8s.io/v1beta1/nodes"
  },
  "items": [
    {
      "metadata": {
        "name": "ip-172-31-0-149.cn-northwest-1.compute.internal",
        "selfLink": "/apis/metrics.k8s.io/v1beta1/nodes/ip-172-31-0-149.cn-northwest-1.compute.internal",
        "creationTimestamp": "2021-12-23T10:00:31Z"
      },
      "timestamp": "2021-12-23T09:59:41Z",
      "window": "30s",
      "usage": {
        "cpu": "150421966n",
        "memory": "1146944Ki"
      }
    },
    {
      "metadata": {
        "name": "ip-172-31-1-49.cn-northwest-1.compute.internal",
        "selfLink": "/apis/metrics.k8s.io/v1beta1/nodes/ip-172-31-1-49.cn-northwest-1.compute.internal",
        "creationTimestamp": "2021-12-23T10:00:31Z"
      },
      "timestamp": "2021-12-23T09:59:35Z",
      "window": "30s",
      "usage": {
        "cpu": "39028587n",
        "memory": "779856Ki"
      }
    },
    {
      "metadata": {
        "name": "ip-172-31-23-21.cn-northwest-1.compute.internal",
        "selfLink": "/apis/metrics.k8s.io/v1beta1/nodes/ip-172-31-23-21.cn-northwest-1.compute.internal",
        "creationTimestamp": "2021-12-23T10:00:31Z"
      },
      "timestamp": "2021-12-23T09:59:37Z",
      "window": "30s",
      "usage": {
        "cpu": "115385940n",
        "memory": "1159348Ki"
      }
    },
    {
      "metadata": {
        "name": "ip-172-31-37-99.cn-northwest-1.compute.internal",
        "selfLink": "/apis/metrics.k8s.io/v1beta1/nodes/ip-172-31-37-99.cn-northwest-1.compute.internal",
        "creationTimestamp": "2021-12-23T10:00:31Z"
      },
      "timestamp": "2021-12-23T09:59:33Z",
      "window": "30s",
      "usage": {
        "cpu": "108861706n",
        "memory": "1237348Ki"
      }
    },
    {
      "metadata": {
        "name": "ip-172-31-40-31.cn-northwest-1.compute.internal",
        "selfLink": "/apis/metrics.k8s.io/v1beta1/nodes/ip-172-31-40-31.cn-northwest-1.compute.internal",
        "creationTimestamp": "2021-12-23T10:00:31Z"
      },
      "timestamp": "2021-12-23T09:59:35Z",
      "window": "30s",
      "usage": {
        "cpu": "44752053n",
        "memory": "748412Ki"
      }
    }
  ]
}%
```

# 编码

方法一：

```go
var podMetrics models.PodMetricsList
    data, err := k8sClient.RESTClient().Get().AbsPath("apis/metrics.k8s.io/v1beta1/namespaces/my-namespace/pods").
        Param("labelSelector", "my-label=label1").
        Param("labelSelector", "my-label=label2").DoRaw()
    if err != nil {
        log.WithFields(map[string]interface{}{
            "error":          err.Error(),
        }).Errorf("get pods resource faield")
    }
    err = json.Unmarshal(data, &podMetrics)
    for _, item := range podMetrics.Items {
        retMetric := models.RetMetric{
            PodName: item.Metadata.Name,
            CPU:     item.Containers[0].Usage.CPU,
            Memory:  item.Containers[0].Usage.Memory,
        }
    }
```

方法二：

```go
package main

import (
	"context"
	"flag"
	"fmt"
	"path/filepath"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
	metrics "k8s.io/metrics/pkg/client/clientset/versioned"
)

func main() {
	var kubeconfig *string
	if home := homedir.HomeDir(); home != "" {
		kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file")
	} else {
		kubeconfig = flag.String("kubeconfig", "", "absolute path to the kubeconfig file")
	}
	flag.Parse()

	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		panic(err.Error())
	}

	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		panic(err.Error())
	}

	nodes, err := clientset.CoreV1().Nodes().List(context.TODO(), metav1.ListOptions{})
	if err != nil {
		panic(err)
	}
	for _, nds := range nodes.Items {
		fmt.Printf("NodeName: %s\n", nds.Name)
	}

	mc, err := metrics.NewForConfig(config)
	if err != nil {
		panic(err)
	}

	nodeMetrics, err := mc.MetricsV1beta1().NodeMetricses().List(context.TODO(), metav1.ListOptions{})
	if err != nil {
		fmt.Println("Error:", err)
		return
	}

	for _, nodeMetric := range nodeMetrics.Items {
		nodeCPU := nodeMetric.Usage.Cpu()
		nodeMem := nodeMetric.Usage.Memory()
		fmt.Println("**************")
		fmt.Println("node: ", nodeMetric.Name)
		fmt.Println("node CPU：", nodeCPU)
		fmt.Println("node MEM：", nodeMem)
	}
}
```

