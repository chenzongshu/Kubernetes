# 概述

Kubernetes部署的服务如果对外提供的是https服务，手动颁发证书在生产是不可接受的， 所以需要一个证书颁发服务在自动给service颁发证书，现在使用的最多的就是cert-manager

cert-manager 是 Kubernetes 上的全能证书管理工具，支持利用 cert-manager 基于 [ACME](https://tools.ietf.org/html/rfc8555) 协议与 [Let's Encrypt](https://letsencrypt.org/) 签发免费证书并为证书自动续期，实现永久免费使用证书。

注：

**`Cert-Manager`只能和`Ingress Controller`或者`Istio`结合来给https服务提供证书**

# 原理

cert-manager在k8s中定义了两个自定义类型资源：`Issuer`和`Certificate`。

其中`Issuer`代表的是证书颁发者，可以定义各种提供者的证书颁发者，当前支持基于`Letsencrypt`、`vault`和`CA`的证书颁发者，还可以定义不同环境下的证书颁发者。

而`Certificate`代表的是生成证书的请求，一般其中存入生成证书的元信息，如域名等等。

一旦在k8s中定义了上述两类资源，部署的`cert-manager`则会根据`Issuer`和`Certificate`生成TLS证书，并将证书保存进k8s的`Secret`资源中，然后在`Ingress`资源中就可以引用到这些生成的`Secret`资源。对于已经生成的证书，还是定期检查证书的有效期，如即将超过有效期，还会自动续期。

> Issuer 与 ClusterIssuer 之间的区别是：Issuer 只能用来签发自身所在 namespace 下的证书，ClusterIssuer 可以签发任意 namespace 下的证书。

证书创建流程：`certificates` -> `certificaterequests` -> `orders` -> `challenges`

## certificaterequests

certificaterequests.certmanager 是cert-manager 产生certificate 过程中会使用的资源，当cert-manager监测到certificate产生后，会产生certificaterequests.certmanager.k8s.io资源，来向issuer 发送request certificate请求。

## orders

orders.certmanager.k8s.io 被ACME 的Issuer 使用，用来管理signed TLD certificate 的ACME order。当一个certificates.certmanager 产生，且需要使勇ACME isser 时，certmanager 会产生orders.certmanager ，来取得certificate

## challenges

challenges.certmanager 资源是ACME Issuer 管理issuing lifecycle 时，用来完成单一个DNS name/identifier authorization 时所使用的。用来确定issue certiticate 的客户端真的是DNS name 的拥有者

当cert-manager 产生order 时，order controller 接到order ，就会为每一个需要DNS certificate 的DNSname ，产生challenges.certmanager

## 过程

- user -> 设定好issuers.certmanager

- user -> 产生certificates.certmanager -> 选择Issuer ->

- cert-manager -> 产生certificaterequest ->

- cert-manager 根据certiticfates.certmanager 产生orders.certmanager ->

- order controller 根据order ，并且跟每一个DNS name target，产生一个challenges.certmanager

- challenges.certmanager 产生后，会开启这个DNS name challenge 的lifecycle

- challenges 状态为queued for processing，在伫列中等待，
- 如果没有别的chellenges 在进行，challenges 状态变成scheduled，这样可以避免多个DNS challenge 同时发生，或是相同名称的DNS challenge 重复
- challenges 与远端的ACME server 'synced' 当前的状态，是否valid
  - 如果ACME 回应这个DNS name 的challenge 还是有效的，则直接把challenges 的状态改成valid，然后移出排程伫列。
  - 如果challenges 状态仍然为pending，challenge controller 会依照设定present 这个challenge，使用HTTP01 或是DNS01，challenges 被标记为presented
  - challenges 先执行self check，确定challenge 状态已经传播给dns servers，如果self check 失败，则会依照interval retry
  - ACME authorization 关联到challenge

# Cert-Manager部署

## 安装

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

## 验证

```yaml
cat <<EOF > test-resources.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-test
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: test-selfsigned
  namespace: cert-manager-test
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-cert
  namespace: cert-manager-test
spec:
  dnsNames:
    - example.com
  secretName: selfsigned-cert-tls
  issuerRef:
    name: test-selfsigned
EOF
```

部署测试资源：

```
kubectl apply -f test-resources.yaml
```

然后查看：

```bash
$ kubectl describe certificate -n cert-manager-test
Name:         selfsigned-cert
······
Events:
  Type    Reason     Age   From          Message
  ----    ------     ----  ----          -------
  Normal  Issuing    16s   cert-manager  Issuing certificate as Secret does not exist
  Normal  Generated  16s   cert-manager  Stored new private key in temporary Secret resource "selfsigned-cert-45hf9"
  Normal  Requested  16s   cert-manager  Created new CertificateRequest resource "selfsigned-cert-825gl"
  Normal  Issuing    16s   cert-manager  The certificate has been successfully issued
```

然后清理测试资源

```
kubectl delete -f test-resources.yaml
```

# 颁发证书

## 自动颁发证书

 cert-manager支持Let’s Encrypt签发证书，先部署一个 Let‘s Encrypt 的ClusterIssuer，然后Ingress部署文件中指定一个`cert-manager.io/cluster-issuer`的`annotations`，cert-manager的组件ingress-shim会自动去生成证书

但是Let‘s Encrypt 之类的在线签发证书有限制：

- 必须开放80端口；
- **证书的签发的域名必须是公网域名**，Let‘s Encrypt会发送http请求到该域名去校验Token



## 手工颁发证书

### 安装cfssl

```bash
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
```

后续增加可执行权限，放到/usr/local/bin/过程省略

### 签发根证书

- CA根证书创建后一般命名是ca.pem。 CA根证书私钥创建后一般命名是ca-key.pem
- CA根证书及其私钥，只需要创建一次即可。后续其他证书都由它签名，CA根证书及其私钥一旦改变，其它证书也就无效了。

#### 配置证书生成策略

配置证书生成策略，让CA软件知道颁发有什么功能的证书。

```bash
#打印config模板文件从而进行修改
cfssl print-defaults config > ca-config.json

vim ca-config.json
{
    "signing": {
        "default": {
            "expiry": "87600h"
        },
        "profiles": {
            "https": {
                "expiry": "87600h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
```

- default默认策略，指定了证书的默认有效期是10年(87600h)
- https：表示该配置(profile)的用途是为https生成证书及相关的校验工作
  - signing：表示该证书可用于签名其它证书；生成的 ca.pem 证书中 CA=TRUE
  - server auth：表示可以该CA 对 server 提供的证书进行验证
  - client auth：表示可以用该 CA 对 client 提供的证书进行验证
- expiry：也表示过期时间，如果不写以default中的为准

#### 生成CA证书和私钥(root 证书和私钥)

```bash
#打印csr模板文件从而进行修改
cfssl print-defaults csr > ca-csr.json

vim ca-csr.json
{
    "CN": "stg-kubernetes-dashboard.czs.cn",
    "key": {
        "algo": "ecdsa",
        "size": 256
    },
    "names": [
        {
            "C": "CN",
            "L": "Shenzhen",
            "ST": "Guangdong"
        }
    ]
}
```

- **CN: Common Name，浏览器使用该字段验证网站是否合法，一般写的是域名。非常重要。浏览器使用该字段验证网站是否合法**
- key：生成证书的算法
- hosts：表示哪些主机名(域名)或者IP可以使用此csr申请的证书，为空或者""表示所有的都可以使用(本例中没有hosts字段)
- names：一些其它的属性
  - C: Country， 国家
  - ST: State，州或者是省份
  - L: Locality Name，地区，城市
  - O: Organization Name，组织名称，公司名称
  - OU: Organization Unit Name，组织单位名称，公司部门

#### 初始化创建CA认证中心

生成运行CA所必需的文件`ca-key.pem`（私钥）和`ca.pem`（证书），还会生成`ca.csr`（证书签名请求），用于交叉签名或重新签名。

```bash
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
2021/04/14 18:07:57 [INFO] generating a new CA key and certificate from CSR
2021/04/14 18:07:57 [INFO] generate received request
2021/04/14 18:07:57 [INFO] received CSR
2021/04/14 18:07:57 [INFO] generating key: ecdsa-256
2021/04/14 18:07:57 [INFO] encoded CSR
2021/04/14 18:07:57 [INFO] signed certificate with serial number 715880542964193816289919874763408450087790700961
```

### 部署

首先将根CA的key及证书文件存入一个secret中，进入刚ca的文件夹，执行

```
kubectl create secret tls ca-key-pair \
   --cert=ca.pem \
   --key=ca-key.pem \
   --namespace=cert-manager
```

然后创建ClusterIssuer

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: ca-issuer
spec:
  ca:
    secretName: ca-key-pair
```

再创建Certificate资源：

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: stg-kubernetes-dashboard.czs.cn
  namespace: kubernetes-dashboard
spec:
  dnsNames:
  - stg-kubernetes-dashboard.czs.cn
  issuerRef:
    kind: ClusterIssuer
    name: ca-issuer
  secretName: stg-kubernetes-dashboard.czs.cn-tls
```

最后创建Ingress规则

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-tls
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  tls:
  - hosts:
    - stg-kubernetes-dashboard.czs.cn
    secretName: stg-kubernetes-dashboard.czs.cn-tls
  rules:
  - host: stg-kubernetes-dashboard.czs.cn
    http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
```

