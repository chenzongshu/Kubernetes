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
mkdir -p /data/nfs/gitlab
```

## 配置 nfs 访问规则

 编辑 /etc/exports 文件，设置 nfs 访问规则，允许 10.110.0.0/16 网段的主机读写 /data/nfs/gitlab 目录。

```
/data/nfs/gitlab 10.110.0.0/16(rw,sync,no_root_squash)
```

参数说明：

|:-|:-|
|参数|	作用|
|ro|read-only|
|rw|read-write|
|root_squash|nfs客户端以root管理员身份访问nfs服务端时，映射为nfs服务端所在主机的匿名用户（权限会受限）|
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

# nfs 客户端安装和配置

## 安装 nfs-utils

因为 nfs-utils 包中同时提供了客户端和服务端，所以在客户端安装时，也需要安装 nfs-utils 程序包。

```
yum install nfs-utils -y
```

## 创建挂载目录

```
mkdir /data/gitlab -p
```

## 设置开机自挂载

编辑 /etc/fstab，设置开机挂载 nfs

## 挂载

将nfs服务端的 /data/nfs/gitlab 目录挂载到本机 /data/gitlab 目录

```
echo "10.110.101.106:/data/nfs/gitlab /data/gitlab nfs defaults 0 0" >> /etc/fstab
```

## 启动挂载

```
mount -a
```

## 验证挂载

```
df -h | grep -i nfs
```


