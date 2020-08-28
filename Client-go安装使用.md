# 简介

Client-go是官方开源的操作kubernetes集群的SDK

官方地址为:  https://github.com/kubernetes/client-go/

# 安装

## golang安装

这个不多介绍， 需要安装golang并配置

需要注意的是需要配置GOPROXY

```
export GOPROXY=https://goproxy.io
```

原来代理可能访问不了, 可以ping一下试试

如果是golang 1.13以上, 可以设置

```
go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.io,direct
```

## Client-go安装

需要先打开go modules

```
export GO111MODULE=on
```

然后下载client-go (如果kubernetes版本大于1.17.0, 如果小于使用另外的命令, 见官网)

```
go get k8s.io/client-go@v0.18.0
go: downloading k8s.io/client-go v0.18.0
```

如果想要运行成功, 还需要另外两个依赖库 `k8s.io/client-go/vendor` 和 `glog`，

```
go get -u k8s.io/apimachinery/...
```

> 说明一下，为什么要使用 `-u` 参数来拉取最新的该依赖库呢？那是因为最新的 client-go 库只能保证跟最新的 `apimachinery` 库一起运行

# Demo

创建一个文件夹`cgtest`, 在里面先执行

```
go mod init cgtest
```

创建`main.go`

```
package main

import (
	"flag"
	"fmt"
	"log"
	"os"
	"path/filepath"
        "context"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {
	var ns string
	flag.StringVar(&ns, "namespace", "", "namespace")

	// Bootstrap k8s configuration from local 	Kubernetes config file
	kubeconfig := filepath.Join(os.Getenv("HOME"), ".kube", "config")
	log.Println("Using kubeconfig file: ", kubeconfig)
	config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
	if err != nil {
		log.Fatal(err)
	}

	// Create an rest client not targeting specific API version
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		log.Fatal(err)
	}

	pods, err := clientset.CoreV1().Pods("").List(context.TODO(), metav1.ListOptions{})
	if err != nil {
		log.Fatalln("failed to get pods:", err)
	}

	// print pods
	for i, pod := range pods.Items {
		fmt.Printf("[%d] %s\n", i, pod.GetName())
	}
}

```

然后执行

```
go run main.go
```

有可能报错

```
go: found golang.org/x/oauth2 in golang.org/x/oauth2 v0.0.0-20200107190931-bf48bf16ab8d
# k8s.io/client-go/rest
../pkg/mod/k8s.io/client-go@v11.0.0+incompatible/rest/request.go:598:31: not enough arguments in call to watch.NewStreamWatcher
	have (*versioned.Decoder)
	want (watch.Decoder, watch.Reporter)
# k8s.io/client-go/tools/clientcmd/api/v1
../pkg/mod/k8s.io/client-go@v11.0.0+incompatible/tools/clientcmd/api/v1/conversion.go:29:15: scheme.AddConversionFuncs undefined (type *runtime.Scheme has no field or method AddConversionFuncs)
../pkg/mod/k8s.io/client-go@v11.0.0+incompatible/tools/clientcmd/api/v1/conversion.go:31:12: s.DefaultConvert undefined (type conversion.Scope has no field or method DefaultConvert)

```

修改`go.mod`

```
require (
    k8s.io/client-go v0.18.2
    k8s.io/apimachinery v0.18.6
) 
```

然后再执行, 可以看到执行成功

```
[root@localhost cgtest]# go run main.go
2020/08/26 02:44:54 Using kubeconfig file:  /root/.kube/config
[0] my-dashboard-kubernetes-dashboard-5c885b88dc-b2rnx
[1] calico-kube-controllers-578894d4cd-ncvjr
[2] calico-node-9wvmt
[3] calico-node-9xjg2
[4] coredns-7ff77c879f-cpzt4
[5] coredns-7ff77c879f-dq68c
[6] etcd-k8s-master
[7] kube-apiserver-k8s-master
[8] kube-controller-manager-k8s-master
[9] kube-proxy-lmmz2
[10] kube-proxy-spx2j
[11] kube-scheduler-k8s-master
[12] ks-installer-848b4d7c9c-gw5wb
```



