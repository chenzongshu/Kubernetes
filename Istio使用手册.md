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



## 卸载Istio

```
istioctl manifest generate --set profile=demo | kubectl delete -f -
```



# 演示



## 安装Bookinfo

1. 进入 Istio 安装目录。

2. Istio 默认[自动注入 Sidecar](https://istio.io/latest/zh/docs/setup/additional-setup/sidecar-injection/#automatic-sidecar-injection). 请为 `default` 命名空间打上标签 `istio-injection=enabled`：

   ```
   $ kubectl label namespace default istio-injection=enabled
   ```

3. 使用 `kubectl` 部署应用：

   ```
   $ kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
   ```

