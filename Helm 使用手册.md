https://blog.51cto.com/14625168/2454599

# 安装

Helm 3.0只有一个客户端, 只需要在本地能连通kubectl的机器上, 即可使用helm

# 公共仓库

## helm 仓库查看

```
# helm repo list
```

首次安装 helm 3 是没有指定默认仓库的。需要手动疯狂添加仓库才可以获取到程序包。

## helm 仓库添加

使用如下命令添加 helm 仓库。

```
# helm repo add stable   https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts

# helm repo add google  https://kubernetes-charts.storage.googleapis.com

# helm repo add jetstack https://charts.jetstack.io
```


## helm 仓库删除

```
# helm repo remove stable

# helm repo remove google

# helm repo remove jetstack
```

## helm 查询 chart

添加完上面的 helm 仓库后，就可以愉快的查找你深爱的程序包（chart）了。

```
# helm search repo chart_name，比如想查找 nginx 的 chart，使用如下命令：

# helm search repo nginx
```

# 私有仓库



# Chart

## chart下载

```
# helm pull chart_name（名称须具体，例 google/nginx-ingress。不能只是关键字，否则下载不到 ）
# helm pull google/nginx-ingress
```

## 自建chart

```
helm create mychart
```

## chart 打包

如果你想上传自建的 chart 到私有仓库中去，需要先将自建的 chart 打包。

```
# helm package mychart
```

## chart 上传

上传 chart 需要 4 个步骤：

- 1 自建私有仓库
- 2 生成或更新 chart 索引文件
- 3 上传 chart 和索引文件
- 4 更新本地 chart 仓库

生成或更新 chart 索引文件, 可以使用下面命令生成一个 index.yaml文件

```
# helm repo index /root/helm/repo
```

更新本地 chart 仓库

```
# 更新本地 chart 仓库，跟远程仓库的 chart 保持同步
# helm repo update
```

# release

## release 介绍

helm 的两大术语：chart 和 release。如果可以把 chart 比作程序源码的话，那么 release 则可以看做是程序运行时的进程。

## release 查看

```
# helm ls
```

## release 安装

在线安装指定的 chart，比如 nginx-ingress。

```
# helm search repo nginx-ingress

# helm install mynginx-ingress google/nginx-ingress
```


## release 更新

如果想修改运行时 release 的配置，可以使用 --set 或者 -f 选项进行修改。

### 基于命令行更新 release

```
## mynginx-ingress 是上面创建的 release 名；google/nginx-ingress 是在线 chart 名

# helm upgrade --set controller.hostNetwork=true mynginx-ingress google/nginx-ingress
```

### 基于文件更新 release

如果想基于文件来更新 release，则首先需要将 chart 下载到本地，然后手动修改 chart 的 values.yaml 文件。

```
## 下载 chart

# helm pull google/nginx-ingress

## 解压缩 chart

# tar -zxvf nginx-ingress-1.26.1.tgz

## 修改 values.yaml 内容。比如修改 hostNetwork 的值为 true

# sed -i 's/hostNetwork: false/hostNetwork: true/g' nginx-ingress/values.yaml

## 针对文件使用 -f 选项更新 release

# helm upgrade mynginx-ingress nginx-ingress -f nginx-ingress/values.yaml
```

## 查看 release 更新后的新值

```
# helm get values mynginx-ingress
```

## release 版本

```
# helm history mynginx-ingress
```

## release 回滚

```
# helm rollback mynginx-ingress 4
```

## release 卸载

```
# helm uninstall mynginx-ingress
```

# helm安装nginx-ingress

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

注意是nodeport的端口, 浏览器中输入 http://zcy.jiuxi.com:30742。




