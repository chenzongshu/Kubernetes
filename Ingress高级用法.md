我们使用Nginx-Ingress-Controller对外暴露服务的时候，在实际生产中可能有复杂的需求，下面我就来聊聊Ingress的高级用法

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



# 泛域名配置

Ingress资源默认就支持对域名配置泛域名，例如可配置`*. ingress-regex.com`泛域名。



# 正则表达式

Ingress资源不支持对域名配置正则表达式，但是可以通过`nginx.ingress.kubernetes.io/server-alias`注解来实现。

创建Ingress，以正则表达式`~^www\.\d+\.example\.com`为例。

```
cat <<-EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ingress-regex
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/server-alias: '~^www\.\d+\.example\.com$, abc.example.com'
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: http-svc1
          servicePort: 80
 EOF
```

# 灰度发布

灰度发布功能可以通过设置注解来实现，为了启用灰度发布功能，需要设置注解`nginx.ingress.kubernetes.io/canary: "true"` , 通过不同注解可以实现不同的灰度发布功能：

- `nginx.ingress.kubernetes.io/canary-weight`：设置请求到指定服务的百分比（值为0~100的整数）。
- `nginx.ingress.kubernetes.io/canary-by-header`：基于request header的流量切分，当配置的`hearder`值为always时，请求流量会被分配到灰度服务入口；当`hearder`值为never时，请求流量不会分配到灰度服务；将忽略其他hearder值，并通过灰度优先级将请求流量分配到其他规则设置的灰度服务。
- `nginx.ingress.kubernetes.io/canary-by-header-value`和`nginx.ingress.kubernetes.io/canary-by-header`：当请求中的`hearder`和`header-value`与设置的值匹配时，请求流量会被分配到灰度服务入口；将忽略其他hearder值，并通过灰度优先级将请求流量分配到其他规则设置的灰度服务。
- `nginx.ingress.kubernetes.io/canary-by-cookie`：基于cookie的流量切分，当配置的`cookie`值为always时，请求流量将被分配到灰度服务入口；当配置的`cookie`值为never时，请求流量将不会分配到灰度服务入口。



