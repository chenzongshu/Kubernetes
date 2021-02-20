# 通过Docker镜像部署实验NFS服务器

https://github.com/ehough/docker-nfs-server



# nfs 服务端安装和配置

nfs 服务端所在主机IP为: 10.110.101.106。可根据自己的实际情况修改。

## 安装 nfs-utils

nfs 服务端安装需要 nfs-utils 程序包。

```
yum install nfs-utils -y
```

## 设置开机自启

```
systemctl enable rpcbind && systemctl enable nfs-server

systemctl start rpcbind && systemctl start nfs-server
```

## 创建共享目录

```
mkdir -p /data/nfs/k8s
chmod 755 /data/nfs/k8s
```

## 配置 nfs 访问规则

 编辑 /etc/exports 文件，设置 nfs 访问规则，允许 10.110.0.0/16 网段的主机读写 /data/nfs/gitlab 目录。

```
/data/nfs/k8s 10.110.0.0/16(rw,sync,no_root_squash)
```

参数说明：

|参数|	作用|
|:--- |:---|
|ro| read-only|
|rw| read-write|
|root_squash| nfs客户端以root管理员身份访问nfs服务端时，映射为nfs服务端所在主机的匿名用户（权限会受限）|
|no_root_squash|nfs客户端以root管理员身份访问nfs服务端时，映射为nfs服务端所在主机的root用户（权限不会受限）
|sync|数据同时写入内存和磁盘。相当于同步双写，因为同时要写内存和磁盘，所以性能会受损，但是数据一致性得以保证，不会丢失|
|async|数据会优先写入内存，然后再写入磁盘。因为写入到内存的数据并不会立刻把数据同步到硬盘中，这时如果断电就会导致部分数据丢失，但是性能却会比sync更有效|

## 重新加载 nfs 服务

```
systemctl reload nfs
```

## 查看 nfs 服务导出列表

```
showmount -e
```



# nfs服务集群安装

## 创建sa和rbac

```
kind: ServiceAccount
apiVersion: v1
metadata:
  name: nfs-client-provisioner
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: kube-system      
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: kube-system     
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

安装 **`kubectl apply -f nfs-rbac.yaml -n kube-system`**

## 部署NFS Provisioner

 **`kubectl apply -f nfs-provisioner-deploy.yaml -n kube-system`**

```
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate      #---设置升级策略为删除再创建(默认为滚动更新)
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          #---由于quay.io仓库国内被墙，所以替换成七牛云的仓库
          image: quay-mirror.qiniu.com/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: nfs-client     #--- nfs-provisioner的名称，以后设置的storageclass要和这个保持一致
            - name: NFS_SERVER
              value: 10.74.20.12   #---NFS服务器地址，和 valumes 保持一致
            - name: NFS_PATH
              value: /data/nfs/k8s      #---NFS服务器目录，和 valumes 保持一致
      volumes:
        - name: nfs-client-root
          nfs:
            server: 10.74.20.12    #---NFS服务器地址
            path: /data/nfs/k8s       #---NFS服务器目录
```

## 部署 NFS SotageClass

**`kubectl apply -f nfs-storage.yaml -n kube-system`**

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true" #---设置为默认的storageclass
provisioner: nfs-client    #---动态卷分配者名称，必须和上面创建的"provisioner"变量中设置的Name一致
parameters:
  archiveOnDelete: "true"  #---设置为"false"时删除PVC不会保留数据,"true"则保留数据
mountOptions:
  - hard        #指定为硬挂载方式
  - nfsvers=4   #指定NFS版本，这个需要根据 NFS Server 版本号设置
```