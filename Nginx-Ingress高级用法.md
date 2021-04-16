我们使用Nginx-Ingress-Controller对外暴露服务的时候，在实际生产中可能有复杂的需求，下面我就来聊聊Nginx-Ingress的高级用法

# https

具体方法的思路就是先创建一个证书，然后把证书放到secrets里面，然后Ingress引用该证书。

但是需要注意的是secrets、Ingress都是区分namespaces的，所以要和应用放到一个ns下面。

## 创建证书

需要单独管理每个证书，以Kubernetes-Dashboard为例

```bash
openssl req -x509 -nodes -days 10000 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=stg-kubernetes-dashboard.czs.cn/O=stg-kubernetes-dashboard.czs.cn"

```

生成 `tls.crt` 和 `tls.key`两个文件

## 创建Secrets

```bash
$ kubectl -n kubernetes-dashboard create secret tls dashboard-tls --key tls.key --cert tls.crt
secret/dashboard-tls created
```

## 创建Ingress

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-stg-k8s-dashboard
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  tls:
  - hosts:
    - stg-kubernetes-dashboard.czs.cn
    secretName: dashboard-tls
  rules:
  - host: stg-kubernetes-dashboard.czs.cn
    http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443
```

这种方法，如果通过网页访问，还是需要信任自建的证书