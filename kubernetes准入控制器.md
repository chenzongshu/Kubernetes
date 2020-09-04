# 什么是准入控制器

准入控制（Admission Controller）是 Kubernetes API Server 用于拦截请求的一种手段。`Admission` 可以做到对请求的资源对象进行校验，修改。**`service mesh` 最近很火的项目 `Istio` 天生支持 Kubernetes，利用的就是 mutating webhooks 来自动将`Envoy`这个 sidecar 容器注入到 Pod 中去的。**



Kubernetes 1.10 之前的版本可以使用 `--admission-control` 打开准入控制。同时 `--admission-control` 的顺序决定 Admission 运行的先后。其实这种方式对于用户来讲其实是挺复杂的，因为这要求用户对所有的准入控制器需要完全了解。

如果使用 Kubernetes 1.10 之后的版本，`--admission-control` 已经废弃，建议使用
`--enable-admission-plugins` 和 `--disable-admission-plugins` 指定需要打开或者关闭的准入控制器。 同时**用户指定的顺序并不影响实际准入控制器的执行顺序**，对用户来讲非常友好。

值得一提的是，有些准入控制器可能会使用 `Alpha` 版本的 API，这时必须首先使能其使用的 API 版本。否则准入控制器不能工作，可能会影响系统功能。