本文基于Istio 1.7.4来安装使用Istio

在 Istio 1.5 以上，饱受诟病的 `Mixer` 终于被废弃了，新版本的 HTTP 遥测默认基于 in-proxy Stats filter，同时可使用 WebAssembly开发 `in-proxy` 扩展。更详细的说明请参考 Istio 1.5 发布公告: https://istio.io/news/releases/1.5.x/announcing-1.5/

# Istio部署

## 下载 Istio 部署文件

可以去github下载 https://github.com/istio/istio/releases/tag/1.7.4

也可以执行下载命令

```
curl -L https://istio.io/downloadIstio | sh -
```

下载完成后会得到一个 `istio-1.7.4` 目录，里面包含了：

- `install/kubernetes` : 针对 Kubernetes 平台的安装文件

- `samples` : 示例应用

- `bin` : istioctl 二进制文件，可以用来手动注入 sidecar proxy

  

将 istioctl 拷贝到 `/usr/local/bin/` 中 或者把文件夹中的bin目录添加到系统变量

```
cp bin/istioctl /usr/local/bin/
或者
export PATH=$PWD/bin:$PATH
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

## 部署

istioctl 提供了多种安装配置文件，可以在下面位置查看：

```
ls manifests/profiles/
default.yaml  demo.yaml  empty.yaml  minimal.yaml  preview.yaml  remote.yaml
```

也可以使用istioctl命令来看

```
[root@localhost istio-1.7.4]# istioctl profile list
Istio configuration profiles:
    preview
    remote
    default
    demo
    empty
    minimal
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



最简单的安装方式， 安装default模板， 可以使用如下命令

```
[root@localhost istio-1.7.4]# istioctl manifest install
This will install the default Istio profile into the cluster. Proceed? (y/N) y
Detected that your cluster does not support third party JWT authentication. Falling back to less secure first party JWT. See https://istio.io/docs/ops/best-practices/security/#configure-third-party-service-account-tokens for details.
✔ Istio core installed
✔ Istiod installed
✔ Ingress gateways installed
✔ Installation complete
```

如果要安装其他配置文件， 可以使用如下命令

```
istioctl manifest install --set profile=demo
```

default模板安装完成可以看到启动的Pod

```
[root@localhost istio-1.7.4]# kubectl get pods -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
istio-ingressgateway-55f67b4b7f-722gs   1/1     Running   0          14h
istiod-7c487bdcd7-8qzn6                 1/1     Running   0          14h
```

## 检测安装是否成功

使用verify-install来检测每个CRD, Service, Deployment是否安装成功, 不过检测之前需要先生成安装清单

```
[root@localhost istio-1.7.4]# istioctl manifest generate --set profile=demo > $HOME/generated-manifest.yaml

[root@localhost istio-1.7.4]# istioctl verify-install -f $HOME/generated-manifest.yaml
CustomResourceDefinition: adapters.config.istio.io.default checked successfully
......
Service: istiod.istio-system checked successfully
Checked 21 custom resource definitions
Checked 2 Istio Deployments
Istio is installed successfully
```





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



## 只暴露必要端口

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
[root@centos-kata istio-1.5.2]# kubectl -n istio-system get pod
NAME                                   READY   STATUS    RESTARTS   AGE
grafana-5cc7f86765-j9xng               1/1     Running   0          15h
istio-egressgateway-65d4884678-ptdtb   1/1     Running   0          15h
istio-tracing-8584b4d7f9-b5n4b         1/1     Running   0          15h
istiod-55dc75dfb5-bpbwq                1/1     Running   0          15h
kiali-696bb665-8g2bz                   1/1     Running   0          19m
prometheus-77b9c64b9c-swhkf            2/2     Running   0          15h
```



```bash
[root@centos-kata istio-1.5.2]# kubectl -n kube-system get pod -l k8s-app=istio-cni-node -o wide
NAME                   READY   STATUS    RESTARTS   AGE   IP               NODE          NOMINATED NODE   READINESS GATES
istio-cni-node-8thqh   2/2     Running   1          71s   172.16.104.131   vm1           <none>           <none>
istio-cni-node-ttnv7   2/2     Running   0          16h   172.16.104.171   centos-kata   <none>           <none>
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

查看刚拉起的service

```
[root@centos-kata ~]# kubectl get svc -n istio-system
NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                    AGE
grafana                     ClusterIP   10.96.43.108    <none>        3000/TCP                                                   20h
istio-egressgateway         ClusterIP   10.96.27.71     <none>        80/TCP,443/TCP,15443/TCP                                   20h
istio-ingressgateway        ClusterIP   10.96.21.20     <none>        15020/TCP,80/TCP,443/TCP                                   20h
istio-pilot                 ClusterIP   10.96.148.126   <none>        15010/TCP,15011/TCP,15012/TCP,8080/TCP,15014/TCP,443/TCP   20h
istiod                      ClusterIP   10.96.0.37      <none>        15012/TCP,443/TCP                                          20h
jaeger-agent                ClusterIP   None            <none>        5775/UDP,6831/UDP,6832/UDP                                 20h
jaeger-collector            ClusterIP   10.96.99.190    <none>        14267/TCP,14268/TCP,14250/TCP                              20h
jaeger-collector-headless   ClusterIP   None            <none>        14250/TCP                                                  20h
jaeger-query                ClusterIP   10.96.31.5      <none>        16686/TCP                                                  20h
kiali                       ClusterIP   10.96.96.177    <none>        20001/TCP                                                  20h
prometheus                  ClusterIP   10.96.131.15    <none>        9090/TCP                                                   20h
tracing                     ClusterIP   10.96.171.96    <none>        80/TCP                                                     20h
zipkin                      ClusterIP   10.96.199.146   <none>        9411/TCP                                                   20h
```



## 暴露dashboard



通过ingress来暴露服务(前提已经安装ingress服务, 如nginx-ingress或者treafik)

创建 `istio-ingress.yaml`文件, 内容如下: 

````
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: jaeger-query
  namespace: istio-system
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: istio.jaeger-query.com
    http:
      paths:
      - path: /
        backend:
          serviceName: jaeger-query
          servicePort: 16686

---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: prometheus
  namespace: istio-system
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: istio.prometheus.com
    http:
      paths:
      - path: /
        backend:
          serviceName: prometheus
          servicePort: 9090

---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: grafana
  namespace: istio-system
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: istio.grafana.com
    http:
      paths:
      - path: /
        backend:
          serviceName: grafana
          servicePort: 3000

---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kiali
  namespace: istio-system
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: istio.kiali.com
    http:
      paths:
      - path: /
        backend:
          serviceName: kiali
          servicePort: 20001
````

然后创建



## 卸载Istio

```
istioctl manifest generate --set profile=demo | kubectl delete -f -
```

