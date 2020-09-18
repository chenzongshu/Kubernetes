# 什么是准入控制器

准入控制（Admission Controller）是在对象持久化之前用于对Kubernetes API Server 用于拦截请求的一种手段。`Admission` 可以做到对请求的资源对象进行校验，修改。**`service mesh` 最近很火的项目 `Istio` 天生支持 Kubernetes，利用的就是 mutating webhooks 来自动将`Envoy`这个 sidecar 容器注入到 Pod 中去的。**



Kubernetes 1.10 之前的版本可以使用 `--admission-control` 打开准入控制。同时 `--admission-control` 的顺序决定 Admission 运行的先后。其实这种方式对于用户来讲其实是挺复杂的，因为这要求用户对所有的准入控制器需要完全了解。

如果使用 Kubernetes 1.10 之后的版本，`--admission-control` 已经废弃，建议使用
`--enable-admission-plugins` 和 `--disable-admission-plugins` 指定需要打开或者关闭的准入控制器。 同时**用户指定的顺序并不影响实际准入控制器的执行顺序**，对用户来讲非常友好。

值得一提的是，有些准入控制器可能会使用 `Alpha` 版本的 API，这时必须首先使能其使用的 API 版本。否则准入控制器不能工作，可能会影响系统功能。

kubernetes官方准入控制器列表见： https://kubernetes.io/zh/docs/reference/access-authn-authz/admission-controllers/#what-does-each-admission-controller-do



# 动态准入控制

官方文档链接见： [kubernetes官方-动态准入控制器](https://kubernetes.io/zh/docs/reference/access-authn-authz/extensible-admission-controllers/)

kubernetes提供的准入控制器需要和APIServer一起编译， 但是也提供了一种webhook的扩展机制

- `MutatingAdmissionWebhook` ：在对象持久化之前进行修改
- `ValidatingAdmissionWebhook` ：在对象持久化之前进行校验

## 注册

这两种类型的 Webhook Admission 插件都需要在 API 中注册，所有 API servers（`kube-apiserver` 和所有扩展 API servers ）都共享一个通用配置。在注册过程中，一个 Webhook Admission 插件描述了以下信息：

- 如何连接到 Webhook Admission Server
- 如何验证 Webhook Admission Server（是否是我们期望的 server）
- 数据应该发送到 Server 的哪个 URL 路径
- 它将处理**哪些资源**和哪些 HTTP 动词
- API server 在连接失败后应该做什么（例如如果 Webhook Admission Server 停止服务了）

```yaml
apiVersion: admissionregistration.k8s.io/v1beta1
kind: ValidatingWebhookConfiguration
metadata:
  name: namespacereservations.admission.online.openshift.io
webhooks:
- name: namespacereservations.admission.online.openshift.io #1
  clientConfig:  #2
    service:
      namespace: default
      name: kubernetes
      path: /apis/admission.online.openshift.io/v1alpha1/namespacereservations
    caBundle: KUBE\_CA\_HERE  #5
  rules:  #3
  - apiGroups:
    - ""
    apiVersions:
    - ""
    operations:
    - CREATE
    resources:
    - namespaces
  failurePolicy: Fail  #4
```

1、**name** : Webhook 的名称。mutating Webhooks 会根据名称进行排序。

2、**clientConfig** : 提供关于如何连接、信任以及发送数据给 Webhook Admission Server 的信息。

3、**rules** : 用来描述 API server 应该在什么时候调用 Admission 插件。在这个例子中，只有创建 `Namespace` 的时候才触发。你可以指定任何资源，例如 serviceinstances.servicecatalog.k8s.io 的 create 操作也是可行的。

4、**failurePolicy** : 如果 Webhook Admission Server 无法连接时如何处理。有两个选项分别是 “Ignore”（故障时开放） 和 “Fail”（故障时关闭）。“故障时开放”可能会导致无法预测的行为。

5、**caBundle** : 注意 API server 调用 Webhook 时一定是通过 TLS 认证的，所以 MutatingWebhookConfiguration 中一定要配置 caBundle。

## 认证和信任

Admission webhook 权限较大， 生产最好要取得APIServer的身份认证， 其可以部署在集群内， 也可以部署在集群外

有两种方法，一是使用kubeconfig文件，另一种是使用类似于[cert-manager](https://www.qikqiak.com/post/automatic-kubernetes-ingress-https-with-lets-encrypt)之类的工具来自动处理 TLS 证书

具体方法见官网



## webhook server实例

可参加别人的实例 ： https://github.com/banzaicloud/admission-webhook-example

kubernetes官方测试用例中，[admission webhook 服务器](https://github.com/kubernetes/kubernetes/blob/v1.13.0/test/images/webhook/main.go) 