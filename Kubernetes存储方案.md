主要比较GlusterFS和Ceph Rook



# GlusterFS/Heketi

GlusterFS 是知名的开源存储方案，是由 Redhat 提供的开源存储方案。[Heketi](https://github.com/gluster/gluster-kubernetes#glusterfs-native-storage-service-for-kubernetes) 是 GlusterFS 的 RESTful 卷管理界面。它提供了易用的方式为 GlusterFS 卷提供了动态供给的功能。如果没有 Heketi 的辅助，就只能手工创建 GlusterFS 卷并映射到 K8s PV 了

## 安装

// TODO 待补充

## 总结

#### 优点

- 久经考验的存储方案。
- 比 Ceph 轻量。

#### 缺点

- Heketi 在公有云上表现不佳。在私有云上表现良好，安装会方便一些。
- 并非为结构化数据设计，例如 SQL 数据库。然而可以使用 GlusterFS 为数据库提供备份和恢复支持。

# Ceph Rook

安装指南见 https://github.com/rook/rook/blob/master/Documentation/ceph-quickstart.md#ceph-storage-quickstart 

## 安装

// TODO 待补充

## 总结

#### 优点

- 在大型生产环境上的健壮存储系统。
- Rook 很好的简化了生命周期管理。

#### 缺点

- 复杂：更加重量级，也不太适合在公有云上运行。在私有云上的运行可能更加合适。



# 性能对比测试

|           | 随机读 BW | 随机写 BW  | 随机读 IOPS | 随机写 IOPS |  顺序读BW | 顺序写BW | 混合IOPS | 混合写IOPS |
| --------- | --------- | ---------- | ----------- | ----------- | ------ | ------ | -------- | -------- |
| GlusterFS | 235 MiB/S | 28.7 MiB/S | 2124        | 827         | 35.9 MiB/S | 17.7 MiB/S | 1283 | 422 |
| Ceph Rook | 118 MiB/S | 13.1 MiB/S | 10900       | 1324        | 81.4 MiB/S | 20.8 MiB/S | 2612 | 882 |

