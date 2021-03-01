我们在私有云中，往往需要构建自己的私有镜像，以提高镜像下载速度，而且结合CICD流程，方便构建流水线, 这里我们使用Harbor, 比起Docker原生的registry, 增加了权限管理, 镜像同步等功能, 比较适合生产应用



Harbor官网有两种搭建方式 , 一种是使用Helm Chart, 一种是用docker compose



对应的, 使用Chart来搭建的, 就是搭建在集群内部, docker compose搭建在集群外部



# Kubernetes搭建



## 添加仓库

```
helm repo add harbor https://helm.goharbor.io
```



## 下载Chart

下载并解压

```
helm pull harbor/harbor

tar -zxvf harbor-1.3.1.tgz
```



## 修改配置文件

```
vim values.yaml

expose:
  type: ingress
  tls:
    commonName: "czs.harbor.org"

  ingress:
    hosts:
      core: czs.harbor.org
      
externalURL: https://czs.harbor.org

harborAdminPassword: "czs123"
```



## 创建PV



我们使用NFS来创建PV， 观察value.yaml文件，可以知道使用了5个PVC，3个1G的，2个5G的



在NFS服务器上创建5个文件夹（安装步骤省略，可以见我另外的文章）



```
[root@k8s-system ~]# mkdir -p /data/nfs/harbor/1g-1
[root@k8s-system ~]# mkdir -p /data/nfs/harbor/1g-2
[root@k8s-system ~]# mkdir -p /data/nfs/harbor/1g-3
[root@k8s-system ~]# mkdir -p /data/nfs/harbor/5g-1
[root@k8s-system ~]# mkdir -p /data/nfs/harbor/5g-2

# 修改目录权限
chmod -R 777 /data/nfs/harbor
```



新建PV资源文件 `pv-harbor-1g.yaml`



```
apiVersion: v1
kind: PersistentVolume
metadata:
    name: pv-harbor-1g-1
spec:
    capacity:
        storage: 1Gi
    volumeMode: Filesystem
    accessModes:
    -  ReadWriteOnce
    persistentVolumeReclaimPolicy: Retain
    nfs:
        server: 192.168.152.103
        path: /data/nfs/harbor/1g-1
---
apiVersion: v1
kind: PersistentVolume
metadata:
    name: pv-harbor-1g-2
spec:
    capacity:
        storage: 1Gi
    volumeMode: Filesystem
    accessModes:
    -  ReadWriteOnce
    persistentVolumeReclaimPolicy: Retain
    nfs:
        server: 192.168.152.103
        path: /data/nfs/harbor/1g-2
---
apiVersion: v1
kind: PersistentVolume
metadata:
    name: pv-harbor-1g-3
spec:
    capacity:
        storage: 1Gi
    volumeMode: Filesystem
    accessModes:
    -  ReadWriteOnce
    persistentVolumeReclaimPolicy: Retain
    nfs:
        server: 192.168.152.103
        path: /data/nfs/harbor/1g-3
```



然后创建小规格PV

```
kubectl apply -f pv-harbor-1g.yaml
```



小规格 pv 创建结束后，注意查看一下创建状态，等到状态都为 Bound 后，再创建大规格 pv



```
apiVersion: v1
kind: PersistentVolume
metadata:
    name: pv-harbor-5g-1
spec:
    capacity:
        storage: 5Gi
    volumeMode: Filesystem
    accessModes:
    -  ReadWriteOnce
    persistentVolumeReclaimPolicy: Retain
    nfs:
        server: 192.168.152.103
        path: /data/nfs/harbor/5g-1
---
apiVersion: v1
kind: PersistentVolume
metadata:
    name: pv-harbor-5g-2
spec:
    capacity:
        storage: 5Gi
    volumeMode: Filesystem
    accessModes:
    -  ReadWriteOnce
    persistentVolumeReclaimPolicy: Retain
    nfs:
        server: 192.168.152.103
        path: /data/nfs/harbor/5g-2
```



## 创建Harbor Release



创建Harbor命令空间

```
kubectl create ns harbor
```

安装Harbor

```
## 第一个 harbor 是 release；第二个是命名空间；第三个是解压后根目录

helm install harbor -n harbor harbor
```

然后检查PVC有没有bound

```
kubectl get pvc -n harbor
```



## 访问

私有云内部别忘了配置域名解析, 然后使用浏览器访问 `https://czs.harbor.org` 即可



## 客户端登录



### 免密登录

docker可以配置免数字证书登录,修改 `/etc/docker/daemon.json` 文件

```
{
    "insecure-registries" : ["czs.harbor.org"]
}
```

### 加密登录

获取数字证书

```
kubectl get secret harbor-harbor-ingress -n harbor  -o yaml
```

解码数字证书值（因为 secret 中证书信息值是采用 base64 编码过的，所以你在使用前还需要进行解码）

```
echo '<ca.crt>' | base64 -d > ca.crt  # ca.crt就是上面yaml中 ca.crt的值
```

在 docker 证书根目录下创建你 ingress 指定域名目录（比如我这里就是 czs.harbor.org），并将上面生成的证书拷贝到此处

```
mkdir -p /etc/docker/cert.d/jiuxi.harbor.org && mv ca.crt /etc/docker/cert.d/czs.harbor.org
```



# Docker-compose安装



最小2C4G40G, 推荐4C8G160G,  提前安装好docker和docker-compose

docker-compose可以去 `https://github.com/docker/compose/releases/`下载



使用端口如下

- 443: Harbor门户和核心API的HTTPS请求

- 4443: 只有在启用Notary时才需要连接到Docker Content Trust服务

- 80: Harbor门户和核心API的HTTP请求



## 下载

下载离线安装包 `https://github.com/goharbor/harbor/releases` , 可以得到一个600M的离线安装包



解压

```
tar xvf harbor-offline-installer-v1.10.2.tgz
```



默认harbor是没有配置证书的, 这里我们简化下流程, 不配置证书, 如果需要配置证书, 参考Harbor官网地址 `https://goharbor.io/docs/1.10/install-config/configure-https/`



## 配置



编辑harbor.yml文件



```
# hostname设置访问地址，可以使用ip、域名，不可以设置为127.0.0.1或localhost
hostname: harbor.phpdev.com

port: 9000
harbor_admin_password: czs123

# The location to store harbor's data
data_volume: /var/harbor/data
# The directory to store store log
location: /var/log/harbor

#如果是不启动HTTPS， 必须把这些注释掉
#https:
  # https port for harbor, default is 443
#  port: 443
  # The path of cert and key files for nginx
#  certificate: /your/certificate/path
#  private_key: /your/private/key/path


```



## 安装



执行`install.sh` ，经过一系列安装, Harbor安装完成， 最后的提示如下



```
[Step 5]: starting Harbor ...
Creating network "harbor_harbor" with the default driver
Creating harbor-log ... done
Creating redis         ... done
Creating harbor-portal ... done
Creating registry      ... done
Creating registryctl   ... done
Creating harbor-db     ... done
Creating harbor-core   ... done
Creating harbor-jobservice ... done
Creating nginx             ... done
✔ ----Harbor has been installed and started successfully.----
```



## 卸载



进入改文件夹，执行 `docker-compose down`





