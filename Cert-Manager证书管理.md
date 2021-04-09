# 概述

Kubernetes部署的服务如果对外提供的是https服务，手动颁发证书在生产是不可接受的， 所以需要一个证书颁发服务在自动给service颁发证书，现在使用的最多的就是cert-manager

cert-manager 是 Kubernetes 上的全能证书管理工具，支持利用 cert-manager 基于 [ACME](https://tools.ietf.org/html/rfc8555) 协议与 [Let's Encrypt](https://letsencrypt.org/) 签发免费证书并为证书自动续期，实现永久免费使用证书。

# 原理

cert-manager在k8s中定义了两个自定义类型资源：`Issuer`和`Certificate`。

其中`Issuer`代表的是证书颁发者，可以定义各种提供者的证书颁发者，当前支持基于`Letsencrypt`、`vault`和`CA`的证书颁发者，还可以定义不同环境下的证书颁发者。

而`Certificate`代表的是生成证书的请求，一般其中存入生成证书的元信息，如域名等等。

一旦在k8s中定义了上述两类资源，部署的`cert-manager`则会根据`Issuer`和`Certificate`生成TLS证书，并将证书保存进k8s的`Secret`资源中，然后在`Ingress`资源中就可以引用到这些生成的`Secret`资源。对于已经生成的证书，还是定期检查证书的有效期，如即将超过有效期，还会自动续期。

> Issuer 与 ClusterIssuer 之间的区别是：Issuer 只能用来签发自身所在 namespace 下的证书，ClusterIssuer 可以签发任意 namespace 下的证书。

# 安装

有yaml模板和helm chart两种模式，用yaml来部署一下

> Cert-Manager 1.2.0版本以上需要Kubernetes 1.16.0版本以上。

```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.3.0/cert-manager.yaml
```

部署完成会看到三个pod

```
[root@ALY-HN1-ACK-Agent-STG-01 cert-manager]# kubectl -n cert-manager get po
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5678fc99dd-46pjp              1/1     Running   0          7m30s
cert-manager-cainjector-7cdccb66f9-ds78j   1/1     Running   0          15s
cert-manager-webhook-68d9ddd8bd-mvk8v      1/1     Running   0          15s
```

# 颁发证书

