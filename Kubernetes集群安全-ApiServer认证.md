# Kubernetes提供了三种级别的客户端认证方式：

- HTTPS证书认证，是基于CA根证书签名的双向数字证书认证方式，是最严格的认证
- HTTP Token认证，通过Token识别每个合法的用户
- HTTP Basic认证

HTTP Token认证和Http Basic认证是相对简单的认证方式，Kubernetes的各组件与Api Server的通信方式仍然是HTTPS，但不再使用CA数字证书。

# 基于CA证书的双向认证方式

kube-apiserver证书配置

使用kubeadm初始化的Kubernetes集群中，kube-apiserver是以静态Pod的形式运行在Master Node上。 可以在Master Node上找到其定义文件`/etc/kubernetes/manifests/kube-apiserver.json`，其中启动命令参数部分如下：

```
  "containers": [
      {
        "name": "kube-apiserver",
        "image": "gcr.io/google_containers/kube-apiserver-amd64:v1.5.2",
        "command": [
          "kube-apiserver",
          "--insecure-bind-address=127.0.0.1",
          "--admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,ResourceQuota",
          "--service-cluster-ip-range=10.96.0.0/12",
          "--service-account-key-file=/etc/kubernetes/pki/apiserver-key.pem",
          "--client-ca-file=/etc/kubernetes/pki/ca.pem",
          "--tls-cert-file=/etc/kubernetes/pki/apiserver.pem",
          "--tls-private-key-file=/etc/kubernetes/pki/apiserver-key.pem",
          "--token-auth-file=/etc/kubernetes/pki/tokens.csv",
          "--secure-port=6443",
          "--allow-privileged",
          "--advertise-address=192.168.61.100",
          "--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname",
          "--anonymous-auth=false",
          "--etcd-servers=http://127.0.0.1:2379"
        ],
```

我们注意到有如下三个启动参数：

`--client-ca-file`: 指定CA根证书文件为`/etc/kubernetes/pki/ca.pem`，内置CA公钥用于验证某证书是否是CA签发的证书
`--tls-private-key-file`: 指定ApiServer私钥文件为`/etc/kubernetes/pki/apiserver-key.pem`
`--tls-cert-file`：指定ApiServer证书文件为`/etc/kubernetes/pki/apiserver.pem`
说明Api Server已经启动了HTTPS证书认证，此时如果在集群外部使用浏览器访问`https://:6443/api`会提示Unauthorized。

```
curl -k https://192.168.61.100:6443/api
Unauthorized
```

在Master Node上进入/etc/kubernetes/pki/目录：

```
cd /etc/kubernetes/pki/
ls
apiserver-key.pem  apiserver-pub.pem  ca.pem      sa-key.pem  tokens.csv
apiserver.pem      ca-key.pem         ca-pub.pem  sa-pub.pem
```

查看CA根证书ca.pem：

```
openssl x509 -noout -text -in ca.pem
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 0 (0x0)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=kubernetes
        Validity
            Not Before: Jan 12 04:52:08 2017 GMT
            Not After : Jan 12 04:52:08 2027 GMT
        Subject: CN=kubernetes
        Subject Public Key Info:
```
       
查看ApiServer的证书apiserver.pem：

```
openssl x509 -noout -text -in apiserver.pem
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 5846608255968714821 (0x5123533b6de1a045)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=kubernetes
        Validity
            Not Before: Jan 12 04:52:08 2017 GMT
            Not After : Jan 12 04:52:08 2018 GMT
        Subject: CN=kube-apiserver
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
......
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Subject Alternative Name:
                DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster.local, IP Address:192.168.61.100
    Signature Algorithm: sha256WithRSAEncryption
......
```

验证apiserver.pem由ca.pem签发：

```
openssl verify -CAfile ca.pem apiserver.pem
apiserver.pem: OK
```

生成客户端私钥和证书

客户端要通过HTTPS证书双向认证的形式访问Api Server需要生成客户端的私钥和证书，其中客户端证书的在生成时-CA参数要指定为Api Server的CA根证书文件`/etc/kubernetes/pki/ca.pem`，-CAkey参数要指定为Api Server的CA key `/etc/kubernetes/pki/ca-key.pem`。 具体操作可参考Creating Certificates

下面生成客户端私钥和证书：

```
cd /etc/kubernetes/pki/
openssl genrsa -out client.key 2048
openssl req -new -key client.key -subj "/CN=192.168.61.100" -out client.csr
openssl x509 -req -in client.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out client.crt -days 3650
```

其中/CN设置客户端所在Node的IP地址

查看生成的证书：

```
openssl x509 -noout -text -in client.crt

openssl verify -CAfile ca.pem client.crt
client.crt: OK
```

kubectl使用生成的客户端私钥和证书访问ApiServer:

```
cd /etc/kubernetes/pki/


kubectl --server=https://192.168.61.100:6443 \
--certificate-authority=ca.pem  \
--client-certificate=client.crt \
--client-key=client.key \
get nodes

NAME      STATUS         AGE
cent0     Ready,master   7d
cent1     Ready          7d
cent2     Ready          7d
```

master node核心组件与ApiServer的认证方式

接下来我们来看一下master node上其他核心组件与ApiServer通信的认证方式。 `/etc/kubernetes/manifests`下的kube-controller-manager.json和kube-scheduler.json说明Controller Manager和Scheduler都是以静态Pod的形式运行在Master Node上，注意到这两个文件里的启动参数`--master=127.0.0.1:8080`，说明它们直接通过insecure-port 8080和ApiServer通信。 而前面`ApiServer的--insecure-bind-address=127.0.0.1`，因此他们之间无需走secure-port。

# HTTP Token认证

我们继续注意一下Master Node下的`/etc/kubernetes/manifests/kube-apiserver.json`文件：

 ```
 "command": [
          "kube-apiserver",
          "--insecure-bind-address=127.0.0.1",
          "--admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,ResourceQuota",
          "--service-cluster-ip-range=10.96.0.0/12",
          "--service-account-key-file=/etc/kubernetes/pki/apiserver-key.pem",
          "--client-ca-file=/etc/kubernetes/pki/ca.pem",
          "--tls-cert-file=/etc/kubernetes/pki/apiserver.pem",
          "--tls-private-key-file=/etc/kubernetes/pki/apiserver-key.pem",
          "--token-auth-file=/etc/kubernetes/pki/tokens.csv",
          "--secure-port=6443",
          "--allow-privileged",
          "--advertise-address=192.168.61.100",
          "--kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname",
          "--anonymous-auth=false",
          "--etcd-servers=http://127.0.0.1:2379"
        ],
```

`--token-auth-file=/etc/kubernetes/pki/tokens.csv`指定了静态Token文件，说明已经开启了Http Token认证。 这个文件的格式是`token,user,uid,"group1,group2,group3"`

```
cat /etc/kubernetes/pki/tokens.csv
792c62a1b5f2b07b,kubeadm-node-csr,ab47c6cb-f403-11e6-95a3-0800279704c8,system:kubelet-bootstrap
```

请求Api时只要在Authorization头中加入`Bearer Token`即可：

```
curl -k --header "Authorization: Bearer 792c62a1b5f2b07b" https://192.168.61.100:6443/api
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "192.168.61.100:6443"
    }
  ]
}
```

kubectl使用Bearer访问Api Server:

```
kubectl --server=https://192.168.61.100:6443 \
--token=792c62a1b5f2b07b \
--insecure-skip-tls-verify=true \
cluster-info
```

# Http Basic认证
kubeadm初始化的集群没有开启Http Basic认证。实践中不建议使用，这里简单体验一下。

在Master Node上创建/etc/kubernetes/baisc_auth文件，文件中每行的格式为password,user,uid,"group1,group2,group3"。 这里简单写入如下内容：

```
1234,admin,1
```

`/etc/kubernetes/manifests/kube-apiserver.json`文件Container的command中加入：

```
--basic_auth_file=/etc/kubernetes/basic_auth
```

因为静态Pod直接被kubelet管理，所以我们需要重启一下kubelet使上面的配置生效：

```
systemctl restart kubelet
```

请求时使用请求头Authorization `Basic BASE64ENCODED(USER:PASSWORD)：`

```
curl -k --header "Authorization:Basic YWRtaW46MTIzNA==" https://192.168.61.100:6443/api
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "192.168.61.100:6443"
    }
  ]
}
```

```
kubectl --server=https://192.168.61.100:6443 \
--username=admin \
--password=1234 \
--insecure-skip-tls-verify=true \
cluster-info
```


