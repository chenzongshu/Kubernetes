# 前言



在第二篇里面，我们安装了kubernetes集群和calico网络组件，一个基本的kubernetes集群已经能够正常工作了，但是在生产实际中，这是远远不够的，这篇里面，我们来安装Helm、ingress组件、Harbor、EFK、prometheus



为了方便，我们使用Helm的chart来安装 ingress、harbor、EFK和Prometheus

# Helm



## 安装



可以到GitHub官网下载： https://github.com/helm/helm/releases ， 选择对应的版本即可，我们为了方便安装Helm3.X的版本



注意：

- Helm 3.X只有一个客户端,  必须放到可以使用kubectl的环境
- Helm 2.X有服务端tiller，提供Restful接口，可以方便自有的平台对接

## helm仓库

首次安装 helm 3 是没有指定默认仓库的。需要手动疯狂添加仓库才可以获取到程序包。可以用下面命令查看

```
# helm repo list
```

## helm 仓库添加

使用如下命令添加 helm 仓库。私有仓库的建立不在本次教程范围内，

```
# helm repo add stable   https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts

# helm repo add google  https://kubernetes-charts.storage.googleapis.com

# helm repo add jetstack https://charts.jetstack.io
```

## helm 安装 chart

###  在线安装

```
# helm search repo nginx-ingress

# helm install mynginx-ingress google/nginx-ingress
```

### 离线安装

如果你的环境不通外网，可以在有外网的环境下载对应的安装包和镜像文件之后离线安装

```
# helm pull chart_name（名称须具体，例 google/nginx-ingress。不能只是关键字，否则下载不到 ）
# helm pull google/nginx-ingress
```



```
helm install <release名> <本地安装包>
```




# nginx-ingress

使用helm chart来安装

## 下载 nginx-ingress 

```
helm pull stable/nginx-ingress
```

可以根据自己需求去下载不同版本, 国内可以选择阿里云

## 配置 nginx-ingress 

修改 values.yaml 文件：

```
1) hostNetwork: false 改为 true

2）type: LoadBalancer 改为 NodePort

3）rbac：

            create: false 改为 true
```

> 注意: 流量是先导入到nginx-ingress, 再转发到各个service, 所以必须把nginx-ingress本身通过各种方式暴露出去, 比如可以通过下面的方式暴露
>
> - loadbalancer
> - NodePort
> - HostPort

## 安装 nginx-ingress 

```
## 第一个 nginx-ingress 是 release 名。第二个 nginx-ingress 是 chart 解压目录。

helm install nginx-ingress nginx-ingress
```

### 修改 deployment version


> Error: unable to build kubernetes objects from release manifest: unable to recognize "": no matches for kind "Deployment" in version "extensions/v1beta1"

如果有如上报错，需要修改 nginx-ingress deployment 文件的 apiVersion。

```
grep -irl "extensions/v1beta1" nginx-ingress | grep deploy | xargs sed -i 's#extensions/v1beta1#apps/v1#g'
```

### 添加 deployment selector

> Error: unable to build kubernetes objects from release manifest: error validating "": error validating data: ValidationError(Deployment.spec): missing required field "selector" in io.k8s.api.apps.v1.DeploymentSpec

如果有如上报错，需要在 deployment 文件添加 selector：

```bash
vi nginx-ingress/templates/controller-deployment.yaml

vi nginx-ingress/templates/default-backend-deployment.yaml
```



```
  selector:
    matchLabels:
      app: {{ template "nginx-ingress.name" . }}
```



修改后再次执行安装


## nginx-ingress 组成

```
kubectl get pods -n nginx-ingress
```

可以看到nginx-ingress 包括 2 个组件：

- 1 nginx-ingress-controller：nginx-ingress 控制器，负责 nginx-ingress pod 的生命周期管理。nginx-ingress pod 本质就是 nginx。用来处理请求路由等功能。这也是为什么称 nginx-ingress pod 是集群流量入口的缘故。
- 2 nginx-ingress-default-backend：默认后端。如果你没有配置路由或路由配错了，将会由此 pod 兜底，一般会显示 404 给你。

## 创建 tomcat 微服务验证

nginx-ingress已经创建完毕，相当于nginx已经就绪。下面我们在k8s集群中创建一个名为 myweb-svc 的 tomcat 服务。

```
## 启动并暴露 tomcat 服务

kubectl run myweb --image=tomcat

kubectl expose deployment myweb --port=8080 --name=myweb-svc
```

## 创建 ingress 

 刚创建完 nginx-ingress，现在又创建 ingress。是不是会有点迷，到底两者的区别是什么？你可以这么来理解，nginx-ingress 是 nginx，而 ingress 则相当于 nginx 里面的一段配置信息。例如下面的 ingress 文件，就是创建从 nginx 路由到 tomcat 的配置。

ingress 文件如下：

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
 name: server-jiuxi-ingress
 annotations:
   kubernetes.io/ingress.class: "nginx"
spec:
 rules:
 - host: zcy.jiuxi.org
   http:
     paths: /
     - path:
       backend:
         serviceName: myweb-svc
         servicePort: 8080
```

翻译成 nginx 的配置就是：

```
server {
    server_name zcy.jiuxi.org
    location / {
        proxy_pass http://myweb-svc:8080
    }
}
```

执行下列语句创建 ingress，生成 nginx 到 tomcat 的路由规则：

```
kubectl apply -f server-jiuxi-ingress.yaml
```


## 访问 tomcat 

上面创建了 nginx-ingress（nginx）、又创建了 tomcat 微服务、又生成了从 nginx 路由到 tomcat 的规则（ingress）。那么下一步我们就可以通过设置好的 ingress 来访问 tomcat 微服务了。但是访问前，还需要再做2点配置。

### 配置 hosts

因为 ingress 中设置了域名 zcy.jiuxi.org，所以需要在浏览器所在的机器上设置 dns。

```
## windows 用户。编辑 C:\Windows\System32\drivers\etc\hosts 文件
## linux   用户。编辑 /etc/hosts 文件
```

### 确定 nginx-ingress 的服务端口

```
kubectl get svc
```

注意是nodeport的端口, 浏览器中输入 http://zcy.jiuxi.com:30742



