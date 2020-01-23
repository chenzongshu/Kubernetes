# Kubernetes日志



```
--v=0	通常，这对操作者来说总是可见的。
--v=1	当您不想要很详细的输出时，这个是一个合理的默认日志级别。
--v=2	有关服务和重要日志消息的有用稳定状态信息，这些信息可能与系统中的重大更改相关。这是大多数系统推荐的默认日志级别。
--v=3	关于更改的扩展信息。
--v=4	调试级别信息。
--v=6	显示请求资源。
--v=7	显示 HTTP 请求头。
--v=8	显示 HTTP 请求内容。
--v=9	显示 HTTP 请求内容，并且不截断内容。
```

# 查看Pod信息

如果一些Pod信息闪创闪退, 每次敲命令看不到, 可以加上 "-w"参数, 一直输出

```
kubectl get po -o wide -w
```


# Kubernetes service

- service的域名, **只能在同一个namespace中解析**, 如果跨域名必须使用 `域名.Namespace名`的形式来访问

## Pod访问apiserver

pod中进程访问apiserver, 是因为apiserver本身也是一个service, 他的名称就是`kubernetes`

## 负载分发策略

有两种: 

- `RoundRobin`：轮询模式，即轮询将请求转发到后端的各个pod上（默认模式）；
- `SessionAffinity`：基于客户端IP地址进行会话保持的模式，第一次客户端访问后端某个pod，之后的请求都转发到这个pod上。

# RBAC

## role

- role只能对命名空间内的资源进行授权







