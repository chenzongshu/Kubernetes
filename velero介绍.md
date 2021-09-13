# Velero

`Heptio Velero`是一款用于 `Kubernetes` 集群资源和持久存储卷（PV）的备份、迁移以及灾难恢复等的开源工具。

## 优点

与直接访问Kubernetes etcd数据库以执行备份和还原的其他工具不同，Velero使用Kubernetes API捕获群集资源的状态并在必要时对其进行还原。这种由API驱动的方法具有许多关键优势：

- 备份可以捕获群集资源的子集，并按名称空间，资源类型和/或标签选择器进行过滤，从而为备份和还原的内容提供了高度的灵活性。
- 托管Kubernetes产品的用户通常无法访问底层的etcd数据库，因此无法对其进行直接备份/还原。
- 通过聚合的API服务器公开的资源可以轻松备份和还原，即使它们存储在单独的etcd数据库中也是如此。

> 注意: 备份过程中创建的对象是不会被备份的

## 相关组件

### Restic

Restic 是一款 GO 语言开发的数据加密备份工具，顾名思义，可以将本地数据加密后传输到指定的仓库。支持的仓库有 Local、SFTP、Aws S3、Minio、OpenStack Swift、Backblaze B2、Azure BS、Google Cloud storage、Rest Server。 项目地址：https://github.com/restic/restic

现阶段通过 `Restic` 备份会有一些限制。

- 不支持备份 hostPath
- 备份数据标志只能通过 Pod 来识别
- 单线程操作大量文件比较慢

### Minio

Minio是一个基于Apache License v2.0开源协议的对象存储服务。它兼容亚马逊S3云存储服务接口，非常适合于存储大容量非结构化的数据，例如图片、视频、日志文件、备份数据和容器/虚拟机镜像等，而一个对象文件可以是任意大小，从几kb到最大5T不等。

## 备份原理

### 备份过程

1. 本地 `Velero` 客户端发送备份指令。
2. `Kubernetes` 集群内就会创建一个 `Backup` 对象。
3. `BackupController` 监测 `Backup` 对象并开始备份过程。
4. `BackupController` 会向 `API Server` 查询相关数据。
5. `BackupController` 将查询到的数据备份到远端的对象存储。

### 后端存储

`Velero` 支持两种关于后端存储的 `CRD`，分别是 `BackupStorageLocation` 和 `VolumeSnapshotLocation`。

- `BackupStorageLocation` 主要用来定义 `Kubernetes` 集群资源的数据存放位置，也就是集群对象数据，不是 `PVC` 的数据。主要支持的后端存储是 `S3` 兼容的存储，比如：`Mino` 和阿里云 `OSS` 等。
- `VolumeSnapshotLocation` 主要用来给 PV 做快照，需要云提供商提供插件。阿里云已经提供了插件，这个需要使用 CSI 等存储机制。你也可以使用专门的备份工具 `Restic`，把 PV 数据备份到阿里云 OSS 中去(安装时需要自定义选项)。

# 安装





# 删除

```bash
kubectl delete namespace/velero clusterrolebinding/velero
kubectl delete crds -l component=velero
```

