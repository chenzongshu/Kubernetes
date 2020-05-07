本文基于Istio 1.5.2来安装使用Istio

在 Istio 1.5 中，饱受诟病的 `Mixer` 终于被废弃了，新版本的 HTTP 遥测默认基于 in-proxy Stats filter，同时可使用 WebAssembly开发 `in-proxy` 扩展。更详细的说明请参考 Istio 1.5 发布公告: https://istio.io/news/releases/1.5.x/announcing-1.5/

# 下载 Istio 部署文件

可以去github下载 https://github.com/istio/istio/releases/tag/1.5.2

也可以执行下载命令

```
curl -L https://istio.io/downloadIstio | sh -
```

下载完成后会得到一个 `istio-1.5.2` 目录，里面包含了：

- `install/kubernetes` : 针对 Kubernetes 平台的安装文件

- `samples` : 示例应用

- `bin` : istioctl 二进制文件，可以用来手动注入 sidecar proxy

  

将 istioctl 拷贝到 `/usr/local/bin/` 中

```
cp bin/istioctl /usr/local/bin/
```

 开启 istioctl 的自动补全功能

将 `tools` 目录中的 `istioctl.bash` 拷贝到 $HOME 目录中：

```bash
cp tools/istioctl.bash ~/
source ~/istioctl.bash
```

在 `~/.bashrc` 中添加一行：

```bash
source ~/istioctl.bash
```

# 部署istio

istioctl 提供了多种安装配置文件，可以通过下面的命令查看：

```
[root@centos-kata istio-1.5.2]# ls install/kubernetes/operator/profiles/
default.yaml  demo.yaml  empty.yaml  minimal.yaml  remote.yaml  separate.yaml
```



它们之间的差异如下：

|                      | default | demo  | minimal | remote |
| -------------------- | ------- | ----- | ------- | ------ |
| **核心组件**         |         |       |         |        |
| istio-egressgateway  |         | **X** |         |        |
| istio-ingressgateway | **X**   | **X** |         |        |
| istio-pilot          | **X**   | **X** | **X**   |        |
| **附加组件**         |         |       |         |        |
| Grafana              |         | **X** |         |        |
| istio-tracing        |         | **X** |         |        |
| kiali                |         | **X** |         |        |
| prometheus           | **X**   | **X** |         | **X**  |

其中标记 **X** 表示该安装该组件。

如果只是想快速试用并体验完整的功能，可以直接使用配置文件 `demo` 来部署。

在正式部署之前，需要先说明两点：

- **1. Istio CNI Plugin**

当前实现将用户 pod 流量转发到 proxy 的默认方式是使用 privileged 权限的 `istio-init` 这个 init container 来做的（运行脚本写入 iptables），需要用到 `NET_ADMIN` capabilities。

Istio CNI 插件的主要设计目标是消除这个 privileged 权限的 init container，换成利用 Kubernetes CNI 机制来实现相同功能的替代方案。具体的原理就是在 Kubernetes CNI 插件链末尾加上 Istio 的处理逻辑，在创建和销毁 pod 的这些 hook 点来针对 istio 的 pod 做网络配置：写入 iptables，让该 pod 所在的 network namespace 的网络流量转发到 proxy 进程。

详细内容请参考[官方文档](https://istio.io/docs/setup/additional-setup/cni/)。

使用 Istio CNI 插件来创建 sidecar iptables 规则肯定是未来的主流方式，不如我们现在就尝试使用这种方法。

- **2. Kubernetes 关键插件（Critical Add-On Pods)**

众所周知，Kubernetes 的核心组件都运行在 master 节点上，然而还有一些附加组件对整个集群来说也很关键，例如 DNS 和 metrics-server，这些被称为**关键插件**。一旦关键插件无法正常工作，整个集群就有可能会无法正常工作，所以 Kubernetes 通过优先级（PriorityClass）来保证关键插件的正常调度和运行。要想让某个应用变成 Kubernetes 的**关键插件**，只需要其 `priorityClassName` 设为 `system-cluster-critical` 或 `system-node-critical`，其中 `system-node-critical` 优先级最高。

> 注意：关键插件只能运行在 `kube-system` namespace 中！

详细内容可以参考[官方文档](https://v1-16.docs.kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/)。



接下来正式安装 Istio，首先部署 `Istio operator`：

```
[root@centos-kata istio-1.5.2]# istioctl operator init
Using operator Deployment image: docker.io/istio/operator:1.5.2

- Applying manifest for component Operator...
✔ Finished applying manifest for component Operator.
Component Operator installed successfully.

*** Success. ***
```

该命令会创建一个 namespace `istio-operator`，并将 Istio operator 部署在此 namespace 中。下载完docker.io/istio/operator:1.5.2镜像之后, 可以看到

```
[root@centos-kata istio-1.5.2]# kubectl -n istio-operator get pod -o wide
NAME                              READY   STATUS    RESTARTS   AGE     IP               NODE          NOMINATED NODE   READINESS GATES
istio-operator-75c4bc6984-49bcm   1/1     Running   0          6m22s   192.168.25.218   centos-kata   <none>           <none>
```

然后创建一个 CR `IstioOperator`：

```yaml
kubectl create ns istio-system
```

```yaml
kubectl apply -f - <<EOF
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
  name: example-istiocontrolplane
spec:
  profile: demo
  components:
    cni:
      enabled: true
      namespace: kube-system
    ingressGateways:
    - enabled: true
      k8s:
        service:
          type: ClusterIP
        strategy:
          rollingUpdate:
            maxUnavailable: 100%
            maxSurge: 0%
        nodeSelector:
          kubernetes.io/hostname: centos-kata
        overlays:
        - apiVersion: apps/v1
          kind: Deployment
          name: istio-ingressgateway
          patches:
          - path: spec.template.spec
            value:
              hostNetwork: true
              dnsPolicy: ClusterFirstWithHostNet
        - apiVersion: v1
          kind: Service
          name: istio-ingressgateway
          patches:
          - path: spec.ports
            value:
            - name: status-port
              port: 15020
              targetPort: 15020
            - name: http2
              port: 80
              targetPort: 80
            - name: https
              port: 443
              targetPort: 443
  values:
    cni:
      excludeNamespaces:
       - istio-system
       - kube-system
       - monitoring
      logLevel: info
EOF
```

其中各个字段的详细含义请参考 [`IstioOperator` API 文档](https://istio.io/docs/reference/config/istio.operator.v1alpha1/)，这里我简要说明一下：

- istio-ingressgateway 的 Service 默认类型为 `LoadBalancer`，需将其改为 `ClusterIP`。
- 为防止集群资源紧张，更新配置后无法创建新的 `Pod`，需将滚动更新策略改为先删除旧的，再创建新的。
- 将 istio-ingressgateway 调度到指定节点。
- 默认情况下除了 `istio-system` namespace 之外，istio cni 插件会监视其他所有 namespace 中的 Pod，然而这并不能满足我们的需求，更严谨的做法是让 istio CNI 插件至少忽略 `kube-system`、`istio-system` 这两个 namespace，如果你还有其他的特殊的 namespace，也应该加上，例如 `monitoring`。

下面着重解释 `overlays` 列表中的字段：

**HostNetwork:**

为了暴露 Ingress Gateway，我们可以使用 `hostport` 暴露端口，并将其调度到某个固定节点。如果你的 CNI 插件不支持 `hostport`，可以使用 `HostNetwork` 模式运行，但你会发现无法启动 ingressgateway 的 Pod，因为如果 Pod 设置了 `HostNetwork=true`，则 dnsPolicy 就会从 `ClusterFirst` 被强制转换成 `Default`。而 Ingress Gateway 启动过程中需要通过 DNS 域名连接 `pilot` 等其他组件，所以无法启动。

我们可以通过强制将 `dnsPolicy` 的值设置为 `ClusterFirstWithHostNet` 来解决这个问题，详情参考：[Kubernetes DNS 高阶指南](https://fuckcloudnative.io/posts/kubernetes-dns/)。

当然你可以部署完成之后再修改 Ingress Gateway 的 `Deployment`，但这种方式还是不太优雅。经过我对 [`IstioOperator` API 文档](https://istio.io/docs/reference/config/istio.operator.v1alpha1/) 的研究，发现了一个更为优雅的方法，那就是直接修改资源对象 `IstioOperator` 的内容，在 `components.ingressGateways` 下面加上么一段：

```yaml
        overlays:
        - apiVersion: apps/v1
          kind: Deployment
          name: istio-ingressgateway
          patches:
          - path: spec.template.spec
            value:
              hostNetwork: true
              dnsPolicy: ClusterFirstWithHostNet
```



具体含义我就不解释了，请看上篇文章。这里只对 IstioOperator 的语法做简单说明：

- `overlays` 列表用来修改对应组件的各个资源对象的 manifest，这里修改的是组件 Ingress Gateway 的 `Deployment`。
- `patches` 列表里是实际要修改或添加的字段，我就不解释了，应该很好理解。



### 只暴露必要端口

从安全的角度来考虑，我们不应该暴露那些不必要的端口，对于 Ingress Gateway 来说，只需要暴露 HTTP、HTTPS 和 metrics 端口就够了。方法和上面一样，直接在 `components.ingressGateways` 的 `overlays` 列表下面加上这么一段：

```yaml
        - apiVersion: v1
          kind: Service
          name: istio-ingressgateway
          patches:
          - path: spec.ports
            value:
            - name: status-port
              port: 15020
              targetPort: 15020
            - name: http2
              port: 80
              targetPort: 80
            - name: https
              port: 443
              targetPort: 443
```



部署完成后，查看各组件状态：

```bash
kubectl -n istio-system get pod
NAME                                    READY   STATUS    RESTARTS   AGE
grafana-5cc7f86765-rt549                1/1     Running   0          3h11m
istio-egressgateway-57999c5b76-59z8v    1/1     Running   0          3h11m
istio-ingressgateway-5b97647565-zjz4k   1/1     Running   0          71m
istio-tracing-8584b4d7f9-2jbjp          1/1     Running   0          3h11m
istiod-86798869b8-jmk9v                 1/1     Running   0          3h11m
kiali-76f556db6d-qnsfj                  1/1     Running   0          3h11m
prometheus-6fd77b7876-c4vzn             2/2     Running   0          3h11m
```



```bash
kubectl -n kube-system get pod -l k8s-app=istio-cni-node

NAME                   READY   STATUS    RESTARTS   AGE
istio-cni-node-4dlfb   2/2     Running   0          3h12m
istio-cni-node-4s9s7   2/2     Running   0          3h12m
istio-cni-node-8g22x   2/2     Running   0          3h12m
istio-cni-node-x2drr   2/2     Running   0          3h12m
```



可以看到 cni 插件已经安装成功，查看配置是否已经追加到 CNI 插件链的末尾：

```bash
cat /etc/cni/net.d/10-calico.conflist

{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
  ...
    {
      "cniVersion": "0.3.1",
      "name": "istio-cni",
      "type": "istio-cni",
      "log_level": "info",
      "kubernetes": {
        "kubeconfig": "/etc/cni/net.d/ZZZ-istio-cni-kubeconfig",
        "cni_bin_dir": "/opt/cni/bin",
        "exclude_namespaces": [
          "istio-system",
          "kube-system",
          "monitoring"
        ]
      }
    }
  ]
}
```