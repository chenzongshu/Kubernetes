# 前言

在 Kubernetes 中，Finalizers 是一种机制，用于确保在删除资源之前执行一些清理操作。

比如每当删除 namespace 或 pod 等一些 Kubernetes 资源时，有时资源状态会卡在 `Terminating`，很长时间无法删除，甚至有时增加 `--force` flag 之后还是无法正常删除。这时就需要 `edit` 该资源，将 `finalizers` 字段设置为 []，之后 Kubernetes 资源就正常删除了。

# 简介

Finalizers 字段属于 Kubernetes GC 垃圾收集器，是一种删除拦截机制，能够让控制器实现异步的删除前（Pre-delete）回调。

简单的说，就是你的资源如果存在`Finalizers`字段（注意该字段为slice，可以同时定义多个），这个时候k8s不会去删除资源，只会进入terminating状态，直到别的控制器去把Finalizers字段清空后，k8s才会删除该资源

常见可以增加Finalizers 的 Kubernetes 资源：

- **Pods**

- **Deployments**

- **Services**
- **ConfigMaps**
- **Secrets**
- **PersistentVolumeClaims (PVCs)**
- **Namespaces**
- **Custom Resources (CRDs)**

# 自定义Finalizer

首先，创建一个 Service 并在其元数据中添加 Finalizers

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  finalizers:
    - my.custom/finalizer
spec:
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

然后实现 Finalizer 逻辑：

需要编写一个Controller来处理 Finalizer 逻辑。这个控制器会监视带有特定 Finalizer 的资源，并在资源删除时执行清理操作。

```go
package main

import (
    "context"
    "fmt"
    "time"

    v1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/cache"
    "k8s.io/client-go/tools/clientcmd"
    "k8s.io/client-go/util/workqueue"
)

func main() {
    config, err := clientcmd.BuildConfigFromFlags("", clientcmd.RecommendedHomeFile)
    if err != nil {
        panic(err.Error())
    }

    clientset, err := kubernetes.NewForConfig(config)
    if err != nil {
        panic(err.Error())
    }

    queue := workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())

    informer := cache.NewSharedInformer(
        cache.NewListWatchFromClient(clientset.CoreV1().RESTClient(), "services", metav1.NamespaceAll, nil),
        &v1.Service{},
        0,
    )

    informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        DeleteFunc: func(obj interface{}) {
            service := obj.(*v1.Service)
            for _, finalizer := range service.ObjectMeta.Finalizers {
                if finalizer == "my.custom/finalizer" {
                    fmt.Printf("Cleaning up resources for service %s\n", service.Name)
                    // 在这里添加你的清理逻辑
                    service.ObjectMeta.Finalizers = removeString(service.ObjectMeta.Finalizers, "my.custom/finalizer")
                    _, err := clientset.CoreV1().Services(service.Namespace).Update(context.TODO(), service, metav1.UpdateOptions{})
                    if err != nil {
                        fmt.Printf("Error removing finalizer: %v\n", err)
                    }
                }
            }
        },
    })

    stopCh := make(chan struct{})
    defer close(stopCh)

    go informer.Run(stopCh)

    <-stopCh
}

func removeString(slice []string, s string) []string {
    for i, v := range slice {
        if v == s {
            return append(slice[:i], slice[i+1:]...)
        }
    }
    return slice
}
```

**运行控制器**： 

编译并运行你的控制器。确保它能够访问 Kubernetes API 服务器，并且有足够的权限来读取和更新 Service 资源。

**删除 Service**： 

当你删除带有自定义 Finalizer 的 Service 时，Kubernetes 不会立即删除它，而是将其标记为“正在删除”。你的控制器会检测到这个状态，并执行清理逻辑。清理完成后，控制器需要移除 Finalizer，Kubernetes 才会真正删除这个 Service。

```bash
kubectl delete service my-service
```