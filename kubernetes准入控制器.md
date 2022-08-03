# 什么是准入控制器

准入控制（Admission Controller）是在对象持久化之前用于对Kubernetes API Server 用于拦截请求的一种手段。`Admission` 可以做到对请求的资源对象进行校验，修改。**`Istio` 利用的就是 mutating webhooks 来自动将`Envoy`这个 sidecar 容器注入到 Pod 中去的。**

kubernetes官方准入控制器列表见： https://kubernetes.io/zh/docs/reference/access-authn-authz/admission-controllers/#what-does-each-admission-controller-do

# 动态准入控制

官方文档链接见： [kubernetes官方-动态准入控制器](https://kubernetes.io/zh/docs/reference/access-authn-authz/extensible-admission-controllers/)

kubernetes提供的准入控制器需要和APIServer一起编译， 但是也提供了一种webhook的扩展机制

- `MutatingAdmissionWebhook` ：处理资源更改
- `ValidatingAdmissionWebhook` ：处理验证

## MutatingAdmissionWebhook

`MutatingAdmissionWebhook` 需要三个对象才能运行

### MutatingWebhookConfiguration

`MutatingAdmissionWebhook` 需要根据 `MutatingWebhookConfiguration` 向 apiserver 注册。在注册过程中，`MutatingAdmissionWebhook` 需要说明：

1. 如何连接 `webhook admission server`；
2. 如何验证 `webhook admission server`；
3. `webhook admission server` 的 URL path；
4. webhook 需要操作对象满足的规则；
5. `webhook admission server` 处理时遇到错误时如何处理。

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

### MutatingAdmissionWebhook 本身

`MutatingAdmissionWebhook` 是一种插件形式的 `admission controller` ，且可以配置到 apiserver 中。`MutatingAdmissionWebhook` 插件可以从 `MutatingWebhookConfiguration` 中获取所有感兴趣的 `admission webhooks`。

然后 `MutatingAdmissionWebhook` 监听 apiserver 的请求，拦截满足条件的请求，并并行执行。

### Webhook Admission Server

`Webhook Admission Server` 只是一个附着到 k8s apiserver 的 http server。对于每一个 apiserver 的请求，`MutatingAdmissionWebhook` 都会发送一个 `admissionReview` 到相关的 `webhook admission server`。`webhook admission server` 再决定如何更改资源。

## 认证和信任

Admission webhook 权限较大， 生产最好要取得APIServer的身份认证， 其可以部署在集群内， 也可以部署在集群外

有两种方法，一是使用kubeconfig文件，另一种是使用类似于[cert-manager](https://www.qikqiak.com/post/automatic-kubernetes-ingress-https-with-lets-encrypt)之类的工具来自动处理 TLS 证书

具体方法见官网

## webhook server实例

可参加别人的实例 ： https://github.com/banzaicloud/admission-webhook-example

kubernetes官方测试用例中，[admission webhook 服务器](https://github.com/kubernetes/kubernetes/blob/v1.13.0/test/images/webhook/main.go) 