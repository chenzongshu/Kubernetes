


# 多租户隔离性

- 控制面的隔离性，主要是隔离不同租户间能够访问哪些 API，或者说看到租户都在干些什么；
- 数据面的隔离性，比如说你的业务在实际运行起来之后，不能去占用别人的计算资源、存储资源等。

# Kubernetes多租户

- 租户定义及 API 访问控制
- 节点隔离和 Runtime 安全, 本质是计算资源的隔离
- 网络隔离

## 租户定义及 API 访问控制

Namespace 是 Kubernetes 中实现租户安全的一个基础边界, 可以选择**直接套用namespace**

Kubernetes API 其实可以粗略地分成两类，我们称之为不同的 scope。

一类是 Root scope，比如 Node、PV，他们在全局是同一个命名空间。比如一个 PV 你命名了 PV-a，那另外一个人就不能再创建 PV-a，因为没有一层隔离，名字已被占用。
另外一类就是有 Namespace 的。

通过**RBAC**来控制权限 ,通过**ResourceQuota** 和**LimitRange** 来限制资源使用

Root scope API 可以对普通用户直接屏蔽

## 节点隔离

如果是容器化部署, 可以给master 打上 Taint。Taint 就是节点上的一种特殊标记，可以用来排斥所有的 Pod， 只有拥有特定 toleration 的 Pod 才能被调度上来.

## 网络隔离

Kubernetes 里面现在用的方案就是配置 NetworkPolicy。NetworkPolicy 这个概念其实可以完全去类比虚机里面的安全组，它是完全扁平的。
