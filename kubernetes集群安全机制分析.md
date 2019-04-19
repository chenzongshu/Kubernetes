# kubernetes集群安全机制分析

标签（空格分隔）： kubernetes

---

# 前言

k8s集群在正式生产环境的过程中, 肯定需要对应的安全机制来保证对资源的控制权.

kubenetes 默认在两个端口提供服务：一个是基于 https 安全端口 6443，另一个是基于 http 的非安全端口 8080。其中非安全端口 8080 限制只能本机访问，即绑定的是 localhost。

Kubernetes集群的所有操作基本上都是通过`kube-apiserver`这个组件进行的，它提供HTTP RESTful形式的API供集群内外客户端调用。

需要注意的是：认证授权过程只存在HTTPS形式的API中。也就是说，如果客户端使用HTTP连接到`kube-apiserver`，那么是不会进行认证授权的。

所以说，可以这么设置，在集群内部组件间通信使用HTTP，集群外部就使用HTTPS，这样既增加了安全性，也不至于太复杂。

# 访问安全机制的层次

访问一次 API Server, 我们可以简单的把过程分成 `认证(Authentication)` -> `授权(Authorization)` -> `准入控制(Admission Control)` 的过程.

- 1、`认证(Authentication)`: 解决用户是谁的问题,就是身份认证,谁可以使用API Server
- 2、`授权(Authorization)`: 解决用户能做什么的问题,给不同的用户不同的操作权限
- 3、`Admission Control`: 能在一定程度上提高安全性，不过更多是资源管理方面的作用

# 认证

Kubernetes提供管理三种级别的客户端身份认证方式：

- 最严格的HTTPS证书认证：基于CA根证书签名的双向数字证书认证方式；
- HTTP Token认证：通过一个Token来识别合法用户；
- HTTP Base认证：通过用户名+密码的方式认证；

注: 使用openstack作为IaaS的还可以使用keystone认证, 具体方法可以搜索.

## 证书认证

双向认证步骤:

- 1、HTTPS通信双方的务器端向CA机构申请证书，CA机构是可信的第三方机构，它可以是一个公认的权威的企业，也可以是企业自身。企业内部系统一般都使用企业自身的认证系统。CA机构下发根证书、服务端证书及私钥给申请者；
- 2、HTTPS通信双方的客户端向CA机构申请证书，CA机构下发根证书、客户端证书及私钥给申请者；
- 3、客户端向服务器端发起请求，服务端下发服务端证书给客户端。客户端接收到证书后，通过私钥解密证书，并利用服务器端证书中的公钥认证证书信息比较证书里的消息，例如域名和公钥与服务器刚刚发送的相关消息是否一致，如果一致，则客户端认为这个服务器的合法身份；
- 4、客户端发送客户端证书给服务器端，服务端接收到证书后，通过私钥解密证书，获得客户端的证书公钥，并用该公钥认证证书信息，确认客户端是否合法；
- 5、客户端通过随机秘钥加密信息，并发送加密后的信息给服务端。服务器端和客户端协商好加密方案后，客户端会产生一个随机的秘钥，客户端通过协商好的加密方案，加密该随机秘钥，并发送该随机秘钥到服务器端。服务器端接收这个秘钥后，双方通信的所有内容都都通过该随机秘钥加密；

etcd证书存放目录：`/etc/etcd/ssl/`
kubernetes证书存放目录：`/etc/kubernetes/ssl`

一套集群要有一套CA证书, 其他证书都是根据这个CA证书创建出来的. 可以先在一个节点把所有证书都装好, 然后拷贝过去.

使用证书的组件如下：

- etcd：使用 ca.pem、kubernetes-key.pem、kubernetes.pem；
- kube-apiserver：使用 ca.pem、kubernetes-key.pem、kubernetes.pem；
- kubelet：使用 ca.pem；
- kube-proxy：使用 ca.pem、kube-proxy-key.pem、kube-proxy.pem；
- kubectl：使用 ca.pem、admin-key.pem、admin.pem；
- kube-controller、kube-scheduler 当前需要和 kube-apiserver 部署在同一台机器上且使用非安全端口通信，故不需要证书。

### token.csv

该文件为一个用户的描述文件，基本格式为 `Token,用户名,UID,用户组`；

这个文件在 apiserver 启动时被 apiserver 加载，然后就相当于在集群内创建了一个这个用户；接下来就可以用 RBAC 给他授权

### 准备工作

安装 cfssl（生产证书的工具）

```
curl -s -L -o /usr/bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
curl -s -L -o /usr/bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
curl -s -L -o /usr/bin/cfssl-certinfo https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
chmod +x /usr/bin/cfssl*
```

创建一个临时目录来保存证书

```
mkdir -pv /data/ssl && cd /data/ssl/
```

### 生成CA证书和私钥

创建 CA 配置文件

```
cfssl print-defaults config > config.json
cfssl print-defaults csr > csr.json

tee ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF
```

> - ca-config.json：可以定义多个 profiles，分别指定不同的过期时间、使用场景等参数；后续在签名证书时使用某个 profile；
- signing：表示该证书可用于签名其它证书；生成的 ca.pem 证书中 CA=TRUE；
- server auth：表示client可以用该 CA 对server提供的证书进行验证；
- client auth：表示server可以用该CA对client提供的证书进行验证；

创建 CA 证书签名请求

```
tee ca-csr.json << EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Shenzhen",
      "L": "Shenzhen",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

> - CN：Common Name，kube-apiserver 从证书中提取该字段作为请求的用户名 (User Name)；浏览器使用该字段验证网站是否合法；
- O：Organization，kube-apiserver 从证书中提取该字段作为请求用户所属的组 (Group)；

生成 CA 证书和私钥

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

会生成`ca-key.pem`, `ca.pem`两个文件, 其中一个秘钥，一个证书

### 创建etcd证书

```
tee etcd-csr.json << EOF
{
    "CN": "etcd",
    "hosts": [
      "127.0.0.1",
      "<etcd IP list>"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Shenzhen",
            "L": "Shenzhen",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

生成 etcd 证书和私钥

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
```

最后生成`etcd-key.pem`和`etcd.pem`


### 创建kubernetes证书

```
tee kubernetes-csr.json << EOF
{
    "CN": "kubernetes",
    "hosts": [
      "127.0.0.1",
      "192.168.1.88",
      "10.254.0.1",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "Shenzhen",
            "L": "Shenzhen",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF

```

> - 192.168.1.88: SLB的ip地址
- 10.254.0.1: service-cluster-ip-range 网段的第一个IP
如果配置了etcd集群https, 想使用这个证书, 也需要将etcd所在的服务器ip加入其中

生成 kubernetes 证书和私钥

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
```

生成三个文件`kubernetes.csr`, `kubernetes-key.pem`, `kubernetes.pem`, 其中`kubernetes.csr`是中间过程文件

### 创建admin证书

**注意, 后续配置了RBAC之后, 不使用admin证书**

```
tee admin-csr.json << EOF
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Shenzhen",
      "L": "Shenzhen",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

```

> - 后续 kube-apiserver 使用 RBAC 对客户端(如 kubelet、kube-proxy、Pod)请求进行授权；
- kube-apiserver 预定义了一些 RBAC 使用的 RoleBindings，如 cluster-admin 将 Group system:masters 与 Role cluster-admin 绑定，该 Role 授予了调用kube-apiserver 的所有 API的权限；
- OU 指定该证书的 Group 为 system:masters，kubelet 使用该证书访问 kube-apiserver 时 ，由于证书被 CA 签名，所以认证通过，同时由于证书用户组为经过预授权的 system:masters，所以被授予访问所有 API 的权限

生成 admin 证书和私钥

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
```

同样生成`admin-key.pem`, `admin.pen`

### 创建 kube-proxy 证书

```
tee kube-proxy-csr.json << EOF
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Shenzhen",
      "L": "Shenzhen",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```

> - CN 指定该证书的 User 为 system:kube-proxy；
- kube-apiserver 预定义的 RoleBinding cluster-admin 将User system:kube-proxy 与 Role system:node-proxier 绑定，该 Role 授予了调用 kube-apiserver Proxy 相关 API 的权限；

生成 kube-proxy 客户端证书和私钥

```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

### 查看证书信息

```
cfssl-certinfo -cert kubernetes.pem
```

### 分发证书

把证书scp到各个节点的 `/etc/kubernetes/ssl`

### 组件配置证书

#### kubectl

- `--certificate-authority=/etc/kubernetes/ssl/ca.pem` 设置了该集群的根证书路径， `--embed-certs为true`表示将`--certificate-authority`证书写入到kubeconfig中
- `--client-certificate=/etc/kubernetes/ssl/admin.pem` 指定kubectl证书
- `--client-key=/etc/kubernetes/ssl/admin-key.pem` 指定kubectl私钥

#### kubelet

当成功签发证书后，目标节点的 kubelet 会将证书写入到 --cert-dir= 选项指定的目录中；此时如果不做其他设置应当生成上述除ca.pem以外的4个文件

- `kubelet-client.crt` 该文件在 kubelet 完成 TLS bootstrapping 后生成，此证书是由 controller manager 签署的，此后 kubelet 将会加载该证书，用于与 apiserver 建立 TLS 通讯，同时使用该证书的 CN 字段作为用户名，O 字段作为用户组向 apiserver 发起其他请求
- `kubelet.crt` 该文件在 kubelet 完成 TLS bootstrapping 后并且没有配置 `--feature-gates=RotateKubeletServerCertificate=true` 时才会生成；这种情况下该文件为一个独立于 apiserver CA 的自签 CA 证书，有效期为 1 年；被用作 kubelet 10250 api 端口

#### kube-apiserver

- `--token-auth-file=/etc/kubernetes/token.csv` 指定了token.csv的位置，用于kubelet 组件 第一次启动时没有证书如何连接 apiserver 。 Token 和 apiserver 的 CA 证书被写入了 kubelet 所使用的 bootstrap.kubeconfig 配置文件中；这样在首次请求时，kubelet 使用 bootstrap.kubeconfig 中的 apiserver CA 证书来与 apiserver 建立 TLS 通讯，使用 bootstrap.kubeconfig 中的用户 Token 来向 apiserver 声明自己的 RBAC 授权身份
- `--tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem` 指定kube-apiserver证书地址
- `--tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem` 指定kube-apiserver私钥地址
- `--client-ca-file=/etc/kubernetes/ssl/ca.pem` 指定根证书地址
- `--service-account-key-file=/etc/kubernetes/ssl/ca-key.pem` 包含PEM-encoded x509 RSA公钥和私钥的文件路径，用于验证Service Account的token，如果不指定，则使用--tls-private-key-file指定的文件
- `--etcd-cafile=/etc/kubernetes/ssl/ca.pem` 到etcd安全连接使用的SSL CA文件
- `--etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem` 到etcd安全连接使用的SSL 证书文件
- `--etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem` 到etcd安全连接使用的SSL 私钥文件

#### kube-controller-manager

kubelet 发起的 CSR 请求都是由 kube-controller-manager 来做实际签署的,所有使用的证书都是根证书的密钥对 。由于kube-controller-manager是和kube-apiserver部署在同一节点上，且使用非安全端口通信，故不需要证书

- `--cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem` 指定签名的CA机构根证书，用来签名为 TLS BootStrap 创建的证书和私钥
- `--cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem` 指定签名的CA机构私钥，用来签名为 TLS BootStrap 创建的证书和私钥
- `--service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem` 同上
- `--root-ca-file=/etc/kubernetes/ssl/ca.pem` 根CA证书文件路径 ，用来对 kube-apiserver 证书进行校验，指定该参数后，才会在Pod 容器的 ServiceAccount 中放置该 CA 证书文件
- `--kubeconfig kubeconfig`配置文件路径，在配置文件中包括Master的地址信息及必要认证信息

#### kube-scheduler && kube-proxy

kube-scheduler是和kube-apiserver一般部署在同一节点上，且使用非安全端口通信，故启动参参数中没有指定证书的参数可选 。 若分离部署，可在kubeconfig文件中指定证书，使用kubeconfig认证，kube-proxy类似

```
$ # 设置集群参数
$ kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig
$ # 设置客户端认证参数
$ kubectl config set-credentials kube-proxy \
  --client-certificate=/etc/kubernetes/ssl/kube-proxy.pem \
  --client-key=/etc/kubernetes/ssl/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
$ # 设置上下文参数
$ kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
$ # 设置默认上下文
$ kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
$ mv kube-proxy.kubeconfig /etc/kubernetes/
```

## token认证

token认证使用token对应用户名, 在`kubectl`和`API Server`端分别进行配置之后, 重启对应服务生效, 这样发往`API Server`的http请求头部带有token信息, 能识别有效的用户请求

先生成一个随机token, 在集群中使用

```
head -c 16 /dev/urandom | od -An -t x | tr -d ' '
```

### API Server

```
$ cat > /etc/kubernetes/pki/token_auth_file<<EOF
7176d48e4e66ddb3557a82f2dd316a93,crd-admin,1
 EOF
```

格式为

```
token,user,uid,"group1,group2,group3" #group可选
```

- 第一列为刚刚生成的token, 要与kubectl config里的token一致
- 第二列为user, 要与kubectl config里的use一致
- 第三列为uid, 编号或是序列号

然后添加kube-spiserver启动参数 `--token-auth-file=/etc/kubernetes/pki/token_auth_file`

- 注意地址
- 需要重启kube-apiserver
- 证书验证和token和同时启用的，但是token和用户名密码，不可同时启用

### kubectl config

```
$ # 设置集群参数
$ kubectl config set-cluster kubernetes \
  --insecure-skip-tls-verify=true \
  --server=${KUBE_APISERVER} 
$ # 设置客户端认证参数
$ kubectl config set-credentials crd-admin \
 --token=7176d48e4e66ddb3557a82f2dd316a93 
$ # 设置上下文参数
$ kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=crd-admin  \
  --namespace=crd 
$ # 设置默认上下文
$ kubectl config use-context kubernetes
```

## base认证

即用户名+密码认证, 与上面token使用方法类似.

`API Server`中设置`–basic-auth-file=SOMEFILE`来启用。一旦`API server`服务启动，加载的用户名和密码信息就不会发生改变，任何对源文件的修改必须重启才能生效

静态密码文件是 CSV格式的文件，每行对应一个用户的信息，前面3列为：密码、用户名、用户ID，这些是是必须的；第四列是可选的组名（如果有多个组，必须用双引号）

这方法不灵活也不安全, 没人使用.

## service account

k8s里面有两种用户，一种是`User`，一种就是`service account`，`User`给人用的，`service account`给Pod进程用的，让Pod进程有相关的权限

如果kubernetes开启了`ServiceAccount`（`–admission_control=…,ServiceAccount,… `）那么会在每个namespace下面都会创建一个默认的default的sa。

```
kubectl get sa --all-namespaces

NAMESPACE     NAME          SECRETS   AGE
default       build-robot   1         1d
default       default       1         32d
default       kube-dns      1         31d
kube-public   default       1         32d
kube-system   dashboard     1         31d
kube-system   default       1         32d
kube-system   heapster      1         30d
kube-system   kube-dns      1         31d
```

创建deployment的时候指定使用

```
      volumes:
      - name: kubernetes-dashboard-certs
        secret:
          secretName: kubernetes-dashboard-certs
      - name: tmp-volume
        emptyDir: {}
      serviceAccountName: kubernetes-dashboard
```

如果没有指定`ServiceAccount`, 则会被设置为default

也可以自己创建`ServiceAccount`

```
$ cat > /tmp/serviceaccount.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
EOF

$ kubectl create -f /tmp/serviceaccount.yaml
serviceaccount "build-robot" created
```


# 授权

`Webhook`和`Custom Modules`略

- `ABAC` : 基于属性的访问控制, 使用用户配置的授权规则去匹配用户的请求
- `RBAC` : 基于账号的资源控制，权限与角色相关联，用户通过成为适当角色的成员而得到这些角色的权限

> 对于ABAC，Kubernetes在实现上是比较难用的，而且需要Master Node的SSH和根文件系统访问权限，授权策略发生变化后还需要重启API Server, 这里不做详细介绍

## RBAC

如果要使用RBAC, 在启动API Server的时候使用参数 `--authorization-mode=RBAC`

API则使用 `rbac.authorization.k8s.io/v1`

RBAC中有两组四种概念:

- `Role`/`ClusterRole`
- `RoleBinding`/`ClusterRoleBinding`

`Role`是一系列的权限的集合，但只能授予单个namespace 中资源的访问权限，例如一个Role可以包含读取 Pod 的权限和列出 Pod 的权限， 
`ClusterRole` 跟 Role 类似，但是可以在集群中全局使用
`RoleBinding`/`ClusterRoleBinding` 就是把对应的权限赋给某个User

### Role

创建一个叫pod-reader的角色

```
[root@master1 ~]# cat pod-reader.yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

```
kubectl create -f pod-reader.yaml
```

创建一个角色绑定，把pod-reader角色绑定到User上

```
[root@master1 ~]# cat devuser-role-bind.yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: kube-system
subjects:
- kind: User
  name: devuser   # 目标用户
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader  # 角色信息
  apiGroup: rbac.authorization.k8s.io
```

```
kubectl create -f devuser-role-bind.yaml
```

### clusterRole

ClusterRoleBinding可以用于集群中所有命名空间中授予权限。以下ClusterRoleBinding允许组“manager”中的任何用户在任何命名空间中读取secrets。

```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  # "namespace" omitted since ClusterRoles are not namespaced
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```

以下ClusterRoleBinding允许组“manager”中的任何用户在任何命名空间中读取secrets

```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

API Server会创建一组默认的`ClusterRole`和`ClusterRoleBinding`对象。其中许多都是`system:`前缀，表示这些资源属于系统基础组件。修改这些资源对象可能会引起一些未知错误。例如：是system:nodeClusterRole，此角色定义了`kubelet`的权限，如果角色被修改，将会引起`kubelet`无法工作。

# Admission Control

通过认证和鉴权之后，客户端并不能得到API Server的真正响应，这个请求还需通过`Admission Control`所控制的一个“准入控制链”的层层考验。

`Admission Control`配备有一个“准入控制器”的插件列表，发送给API Server的任何请求都需要通过列表中每一个准入控制器的检查，检查不通过API Server拒绝此调用请求

在API Server上设置--admission-control参数，即可定制我们需要的准入控制链。如果启用多种准入控制选项，1.9版本建议的设置（含加载顺序）如下

```
--admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota
```

> - AlwaysAdmit：允许所有请求；
- AlwaysPullmages：在启动容器之前总去下载镜像，相当于在每个容器的配置项imagePullPolicy=Always
- AlwaysDeny：禁止所有请求，一般用于测试；
- Service Account：这个plug-in将ServiceAccount实现了自动化，默认启用，如果你想使用ServiceAccount对象，那么强烈推荐使用它。
- ResourceQuota：用于资源配额管理目的，作用于namespace上，它会观察所有请求，确保在namespace上的配额不会超标。推荐在Admission Control参数列表中将这个插件安排在最后一个，以免可能被其他插件拒绝的Pod被过早分配资源。
- LimitRanger：用于资源限制管理，作用于namespace上，确保对Pod进行资源限制。启用该插件还会为未设置资源限制的Pod进行默认设置，例如为namespace "default"中所有的Pod设置0.1CPU的资源请求。
- NamespaceLifecycle：如果尝试在一个不存在的namespace中创建资源对象，则该创建请求将被拒绝。当删除一个namespace时，系统将会删除该namespace中所有对象，保存Pod，Service等。
- DefaultStorageClass：为了实现共享存储的动态供应，为未指定StorageClass或PV的PVC尝试匹配默认的StorageClass，尽可能减少用户在申请PVC时所需了解的后端存储细节。





























