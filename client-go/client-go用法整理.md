# 资源更新方式

> 涉及到资源的操作，就不得不提k8s资源更新的机制，方便理解下面的代码和写法

## patch

当用户对某个资源对象提交一个 patch 请求时，kube-apiserver 不会考虑版本问题，而是“无脑”地接受用户的请求（只要请求发送的 patch 内容合法），也就是将 patch 打到对象上、同时更新版本号

`vendor/k8s.io/apimachinery/pkg/types/patch.go`

```go
const (
    JSONPatchType           PatchType = "application/json-patch+json"
    MergePatchType          PatchType = "application/merge-patch+json"
    StrategicMergePatchType PatchType = "application/strategic-merge-patch+json"
    ApplyPatchType          PatchType = "application/apply-patch+yaml"
)
```

有四种patch合并方式：

- json

- 合并

- 策略性合并

- apply

### json

在RFC6902协议的定义中，JSON Patch是执行在资源对象上的一系列操作

```go
{
    "op": "add",
    "path": "/spec/containers/0/image",
    "value": "busybox:latest"
}
```

- **op**: 表示操作，有“**test**”，“**remove**”，“**add**”，“**replace**”，“**move**”，“**copy**”这六种操作。
- **path**: 表示被作资源对象的路径. 例如`/spec/containers/0/image`表示要操作的对象是“spec.containers[0].image”
- **value**: 表示操作的具体的值

### Merge

一般merge用的比较少

merge patch 必须包含一个对资源对象的部分描述，json对象。该json对象被提交到服务端，并和服务端的当前对象进行合并，从而创建新的对象。完整的替换列表，也就是说，新的列表定义会替换原有的定义。

这个策略并不适合我们对一些列表深层的字段做更新，更适用于大片段的覆盖更新。不过对于 labels/annotations 这些 map 类型的元素更新，merge patch 是可以单独指定 key-value 操作的，相比于 json patch 方便一些，写起来也更加直观

```yaml
kubectl patch deployment/foo --type='merge' -p '{"metadata":{"labels":{"test-key":"foo"}}}'
```

注意：

- 如果value的值为null,表示要删除对应的键

- merge patch **无法单独更新一个列表(数组)中的某个元素**，因此不管我们是要在 containers 里新增容器、还是修改已有容器的 image、env 等字段，都要用整个 containers 列表(数组)来提交 patch：

### StrategicMerge

patch 策略并没有一个通用的 RFC 标准，而是 K8s 独有的，不过相比前两种而言却更为强大

> strategic 策略只能用于原生 K8s 资源以及 Aggregated API 方式的自定义资源，对于 CRD 定义的资源对象，是无法使用的。

它在 K8s 原生资源的数据结构定义中额外定义了一些的策略注解

比如我们要更新container的一些信息，merge patch要指定数组序号来更新，但是strategic merge path就可以根据container name来找到对应的容器。类似下面命令

```go
kubectl patch deployment/foo -p \
  '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:mainline"}]}}}}'
```

对于策略性合并 patch，列表可以根据其 patch 策略进行替换或合并。 patch 策略由 Kubernetes 源代码中字段标记中的 `patchStrategy` 键的值指定，指定了就是合并，没有的字段就是替换，比如`Tolerations`就是替换

```go
type PodSpec struct {
  ...
  Containers []Container `json:"containers" patchStrategy:"merge" patchMergeKey:"name" ...`

  Tolerations []Toleration `json:"tolerations,omitempty" protobuf:"bytes,22,opt,name=tolerations"`
  ...
}
```

### apply

apply patch 分为client-side apply和服务器端的server-side apply，主要是kubectl等k8s组件使用的

在使用默认参数执行 apply 时，触发的是 client-side apply，里面封装了多步操作，

server-side apply是k8s v1.18以上的特性。什么是Server-side Apply呢？简单的说就是多个Controller控制一个资源, 通过`managedFields`来记录哪个Field被哪个资源控制，例如 WorkloadController只能修改image相关的操作，而ScaleController 只能修改副本数。

相比于client-side apply 用 last-applied annotations的方式，服务器端(server-side) apply 新提供了一种声明式 API (叫 ManagedFields) 来明确指定谁管理哪些资源字段。当使用server-side apply时，尝试着去改变一个被其他人管理的字段， 会导致请求被拒绝（在没有设置强制执行时，参见[冲突](https://kubernetes.io/zh/docs/reference/using-api/server-side-apply/#conflicts)）

## update

对于patch请求， 我们只需要将对象中某些字段的修改提交给k8s；而update请求,我们需要将整个修改后的对象提交给k8s而对于patch请求

所以update请求必须带有resourceVersion，来做版本控制，如果版本不是最新，则会返回`"the object has been modified; please apply your changes to the latest version and try again"` 这种报错

# 范例

## 连接集群

作为Pod跑在集群内和集群外略有不同，集群外需要去找kubeconfig文件

```go
func creatClient() {
    var config *rest.Config
    var err error
    var kubeconfig *string
    if env == "in-cluster" {
        // creates the in-cluster config
        config, err = rest.InClusterConfig()
        if err != nil {
            panic(err.Error())
        }
        // creates the clientset
        clientset, err = kubernetes.NewForConfig(config)
        if err != nil {
            panic(err.Error())
        }
        // return clientset, metricset
    } else {
        if home := homedir.HomeDir(); home != "" {
            kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file")
        } else {
            kubeconfig = flag.String("kubeconfig", "", "absolute path to the kubeconfig file")
        }
        flag.Parse()

        // use the current context in kubeconfig
        config, err = clientcmd.BuildConfigFromFlags("", *kubeconfig)
        if err != nil {
            panic(err.Error())
        }

        clientset, err = kubernetes.NewForConfig(config)
        if err != nil {
            panic(err.Error())
        }
        // return clientset, metricset
    }
}
```

## 获取pod信息

## cordon/uncordon

```go
type patchStringValue struct {
    Op    string `json:"op"`
    Path  string `json:"path"`
    Value bool   `json:"value"`
}

func NodeCordon(ctx context.Context, client *kubernetes.Clientset, node *v1.Node) error {
    payload := []patchStringValue{{
        Op:    "replace",
        Path:  "/spec/unschedulable",
        Value: true,
    }}
    payloadBytes, _ := json.Marshal(payload)
    _, err := client.CoreV1().Nodes().Patch(ctx, node.Name, types.JSONPatchType, payloadBytes, metav1.PatchOptions{})
    if err != nil {
        klog.InfoS("Node ", node.Name, " [NodeCordon()] have error: ", err)
    }
    return err
}


func UnNodeCordon(ctx context.Context, client *kubernetes.Clientset, node *v1.Node) error {
    payload := []patchStringValue{{
        Op:    "replace",
        Path:  "/spec/unschedulable",
        Value: false,
    }}
    payloadBytes, _ := json.Marshal(payload)
    _, err := client.CoreV1().Nodes().Patch(ctx, node.Name, types.JSONPatchType, payloadBytes, metav1.PatchOptions{})
    if err != nil {
        klog.ErrorS(err, "[UnNodeCordon()] have error: ", "node:", node.Name)
    }
    return err
}
```

## 节点新增label

### Json

```go
type patchLabelValue struct {
    Op    string      `json:"op"`
    Path  string      `json:"path"`
    Value interface{} `json:"value"`
}

func AddLabelNodeScheduler(ctx context.Context, client *kubernetes.Clientset, node *v1.Node) error {
    payload := []patchLabelValue{{
        Op:    "add",
        Path:  "/metadata/labels/hll-descheduler",
        Value: "true",
    }}

    payloadBytes, _ := json.Marshal(payload)
    _, err := client.CoreV1().Nodes().Patch(ctx, node.Name, types.JSONPatchType, payloadBytes, metav1.PatchOptions{})

    return err
}
```

### 策略性合并

```go
func AddLabelNodeScheduler(ctx context.Context, client *kubernetes.Clientset, node *v1.Node) error {
    labels := node.Labels
    labels["descheduler"] = "true"
    patchData := map[string]interface{}{"metadata": map[string]map[string]string{"labels": labels}}
    playLoadBytes, _ := json.Marshal(patchData)

    _, err := client.CoreV1().Nodes().Patch(ctx, node.Name, types.StrategicMergePatchType, playLoadBytes, metav1.PatchOptions{})
    return err
}
```

## 节点删除label

### Json

```go
func RemoveLabelNodeScheduler(ctx context.Context, client *kubernetes.Clientset, node *v1.Node) error {
    payload := []patchLabelValue{{
        Op:    "remove",
        Path:  "/metadata/labels/hll-descheduler",
        Value: "true",
    }}
    payloadBytes, _ := json.Marshal(payload)
    _, err := client.CoreV1().Nodes().Patch(ctx, node.Name, types.JSONPatchType, payloadBytes, metav1.PatchOptions{})
    return err
}
```

### 策略性合并

value置为null就可以删除

```go
func RemoveLabelNodeScheduler(ctx context.Context, client *kubernetes.Clientset, node *v1.Node) error {
    labels := node.Labels
    labels["descheduler"] = nil
    patchData := map[string]interface{}{"metadata": map[string]map[string]string{"labels": labels}}
    playLoadBytes, _ := json.Marshal(patchData)

    _, err := client.CoreV1().Nodes().Patch(ctx, node.Name, types.StrategicMergePatchType, playLoadBytes, metav1.PatchOptions{})
    return err
}
```

### update

- update之前注意缓存的node没被修改过

```go
func RemoveLabelOffNode(c clientset.Interface, nodeName string, labelKeys []string) error {
    var node *v1.Node
    var err error
    for attempt := 0; attempt < retries; attempt++ {
        node, err = c.CoreV1().Nodes().Get(context.TODO(), nodeName, metav1.GetOptions{})
        if err != nil {
            return err
        }
        if node.Labels == nil {
            return nil
        }
        for _, labelKey := range labelKeys {
            if node.Labels == nil || len(node.Labels[labelKey]) == 0 {
                break
            }
            delete(node.Labels, labelKey)
        }
        _, err = c.CoreV1().Nodes().Update(context.TODO(), node, metav1.UpdateOptions{})
        if err != nil {
            if !apierrors.IsConflict(err) {
                return err
            } else {
                klog.V(2).Infof("Conflict when trying to remove a labels %v from %v", labelKeys, nodeName)
            }
        } else {
            break
        }
        time.Sleep(100 * time.Millisecond)
    }
    return err
}
```

## 获取某节点上的Pod

```go
func getNodePods(client k8sClient.Interface, node v1.Node) (*v1.PodList, error) {
    fieldSelector, err := fields.ParseSelector("spec.nodeName=" + node.Name +
        ",status.phase!=" + string(v1.PodSucceeded) +
        ",status.phase!=" + string(v1.PodFailed))

    if err != nil {
        return nil, err
    }

    return client.CoreV1().Pods(v1.NamespaceAll).List(context.TODO(), metaV1.ListOptions{
        FieldSelector: fieldSelector.String(),
    })
}
```

## 节点操作annotation

### 新增

和上面操作label类似

```go
	payload := []patchLabelValue{{
		Op:    "add",
		Path:  "/metadata/annotations/{key}",  //key-value是annotation要填写的值
		Value: "{value}",
	}}

	payloadBytes, _ := json.Marshal(payload)
	_, err := client.CoreV1().Nodes().Patch(ctx, node.Name, types.JSONPatchType, payloadBytes, metav1.PatchOptions{})
	if err != nil {
		klog.ErrorS(err, "return error: ", "node:", node.Name)
	}
```

### 删除

```go
	payload := []patchLabelValue{{
		Op:    "remove",
		Path:  "/metadata/annotations/{key}",  //key-value是annotation要填写的值
		Value: "{value}",
	}}

	payloadBytes, _ := json.Marshal(payload)
	_, err := client.CoreV1().Nodes().Patch(ctx, node.Name, types.JSONPatchType, payloadBytes, metav1.PatchOptions{})
	if err != nil {
		klog.ErrorS(err, "return error: ", "node:", node.Name)
	}
```



# Informer

有各种类型： `NewInformer`、`NewIndexInfomer`、`NewShareInformer` 、`NewShareIndexInformer`  和 `NewSharedInformerFactoryWithOptions`

## 前置基础

Controller或者client-go这个SDK里面，涉及到的概念有

- Reflector(反射器) 通过 http trunk 协议监听 K8s apiserver 服务的资源变更事件 , 事件主要分为三个动作 `ADD`、`UPDATE`、`DELETE`；

- Reflector(反射器) 将事件添加到 Delta 队列中等待；

- Informer 从队列获取新的事件；

- Informer 调用 Indexer (索引器 , 该索引器内包含 Store 对象), 默认索引器是以 namespace 和 name 作为每种资源的索引名；

- Indexer 通过调用 Store 存储对象按资源分类存储；

## NewInformer

Informer 会向我们返回两个对象：`Store` 和 `Controller`。其中，Controller 主要用于控制监听事件的循环过程，而 Store 对象实际上与之前所讲的内容相同，我们可以直接从本地缓存中获取我们所监听的资源。

```go
...
func main () {
        cliset := config.NewK8sConfig().InitClient()
        // 获取configmap
        listWatcher := cache.NewListWatchFromClient(
                cliset.CoreV1().RESTClient(),
                "configmaps",
                "default",
                fields.Everything(),
        )
        // CmdHandler 和上述的 EventHandler (参考 3.3.5)
        store, controller := cache.NewInformer(listWatcher, &v1.ConfigMap{}, 0, &CmdHandler{})
        // 开启一个goroutine 避免主线程堵塞
        go controller.Run(wait.NeverStop)
        // 等待3秒 同步缓存
        time.Sleep(3 * time.Second)
        // 从缓存中获取监听到的 configmap 资源
        fmt.Println(store.List())

}

// Output:
// Add:  kube-root-ca.crt
// Add:  istio-ca-root-cert
// [... configmap 对象]
```

## NewIndexInfomer

在 NewInformer 基础上接收 Indexer

```go
import (
    "fmt"
    "k8s-clientset/config"
    "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/meta"
    "k8s.io/apimachinery/pkg/fields"
    "k8s.io/apimachinery/pkg/util/wait"
    "k8s.io/client-go/tools/cache"
    "time"
)

...

// LabelsIndexFunc 用作给出可检索的索引值
func LabelsIndexFunc(obj interface{}) ([]string, error) {
        metaD, err := meta.Accessor(obj)
        if err != nil {
                return []string{""}, fmt.Errorf("object has no meta: %v", err)
        }
        return []string{metaD.GetLabels()["app"]}, nil
}

func main () {
        cliset := config.NewK8sConfig().InitClient()
        // 获取configmap
        listWatcher := cache.NewListWatchFromClient(
                cliset.CoreV1().RESTClient(),
                "configmaps",
                "default",
                fields.Everything(),
        )
        // 创建索引其并指定名字
        myIndexer := cache.Indexers{"app": LabelsIndexFunc}
        // CmdHandler 和上述的 EventHandler (参考 3.3.5)
        i, c := cache.NewIndexerInformer(listWatcher, &v1.Pod{}, 0, &CmdHandler{}, myIndexer)
        // 开启一个goroutine 避免主线程堵塞
        go controller.Run(wait.NeverStop)
        // 等待3秒 同步缓存
        time.Sleep(3 * time.Second)
        // 通过 IndexStore 指定索引器获取我们需要的索引值
        // busy-box 索引值是由于 在某个 pod 上打了一个 label 为 app: busy-box
        objList, err := i.ByIndex("app", "busy-box")
        if err != nil {
                panic(err)
        }

        fmt.Println(objList[0].(*v1.Pod).Name)

}

// Output:
// Add:  cloud-enterprise-7f84df95bc-7vwxb
// Add:  busy-box-6698d6dff6-jmwfs
// busy-box-6698d6dff6-jmwfs
//
```

## NewShareInformer

Share Informer 和 Informer 的主要区别就是可以添加多个 EventHandler

```go
// 只展示重要部分
...
func main() {
        cliset := config.NewK8sConfig().InitClient()
        listWarcher := cache.NewListWatchFromClient(
                cliset.CoreV1().RESTClient(),
                "configmaps",
                "default",
                fields.Everything(),
        )
        // 全量同步时间
        shareInformer := cache.NewSharedInformer(listWarcher, &v1.ConfigMap{}, 0)
        // 可以增加多个Event handler
        shareInformer.AddEventHandler(&handlers.CmdHandler{})
        shareInformer.AddEventHandler(&handlers.CmdHandler2{})
        shareInformer.Run(wait.NeverStop)
}
```

## NewShareIndexInformer

`NewSharedIndexInformer` 和 `NewSharedInformer` 的区别就是可以添加 Indexer

## NewSharedInformerFactoryWithOptions

使用该方法可以创建一个 Informer 工厂对象，在该工厂对象启动前，我们可以向其中添加任意 Kubernetes 内置的资源以及任意 Indexer

```go
package main

import (
        "fmt"
        "k8s-clientset/config"
        "k8s-clientset/dc/handlers"
        "k8s.io/apimachinery/pkg/labels"
        "k8s.io/apimachinery/pkg/runtime/schema"
        "k8s.io/apimachinery/pkg/util/wait"
        "k8s.io/client-go/informers"
)

func main() {

        cliset := config.NewK8sConfig().InitClient()
        informerFactory := informers.NewSharedInformerFactoryWithOptions(
                cliset,
                0,
                // 指定的namespace 空间，如果需要所有空间，则不指定该参数
                informers.WithNamespace("default"),
        )
        // 添加 ConfigMap 资源
        cmGVR := schema.GroupVersionResource{
                Group:    "",
                Version:  "v1",
                Resource: "configmaps",
        }
        cmInformer, _ := informerFactory.ForResource(cmGVR)
        // 增加对 ConfigMap 事件的处理
        cmInformer.Informer().AddEventHandler(&handlers.CmdHandler{})

        // 添加 Pod 资源
        podGVR := schema.GroupVersionResource{
                Group:    "",
                Version:  "v1",
                Resource: "pods",
        }
        _, _ = informerFactory.ForResource(podGVR)

        // 启动 informerFactory
        informerFactory.Start(wait.NeverStop)
        // 等待所有资源完成本地同步
        informerFactory.WaitForCacheSync(wait.NeverStop)
        
        // 打印资源信息
        listConfigMap, _ := informerFactory.Core().V1().ConfigMaps().Lister().List(labels.Everything())
        fmt.Println("Configmap:")
        for _, obj := range listConfigMap {
                fmt.Printf("%s/%s \n", obj.Namespace, obj.Name)
        }
        fmt.Println("Pod:")
        listPod, _ := informerFactory.Core().V1().Pods().Lister().List(labels.Everything())
        for _, obj := range listPod {
                fmt.Printf("%s/%s \n", obj.Namespace, obj.Name)
        }
        select {}
}

// Ouput:

// Configmap:
// default/istio-ca-root-cert 
// default/kube-root-ca.crt 
// default/my-config 
// Pod:
// default/cloud-enterprise-7f84df95bc-csdqp 
// default/busy-box-6698d6dff6-42trb 
```

如果想监听所有可操作的内部资源，可以使用 `DiscoveryClient` 去获取当前集群的资源版本再调用 `InformerFactory` 进行资源缓存
