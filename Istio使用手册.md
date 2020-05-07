本文基于Istio 1.5.2来安装使用Istio

在 Istio 1.5 中，饱受诟病的 `Mixer` 终于被废弃了，新版本的 HTTP 遥测默认基于 in-proxy Stats filter，同时可使用 WebAssembly开发 `in-proxy` 扩展。更详细的说明请参考 Istio 1.5 发布公告: https://istio.io/news/releases/1.5.x/announcing-1.5/

# 下载 Istio 部署文件

可以去github下载 https://github.com/istio/istio/releases/tag/1.5.2

也可以执行下载命令

```
curl -L https://istio.io/downloadIstio | sh -
```

下载完成后会得到一个 `istio-1.5.2` 目录，里面包含了：

- `install/kubernetes` : 针对 Kubernetes 平台的安装文件
- `samples` : 示例应用
- `bin` : istioctl 二进制文件，可以用来手动注入 sidecar proxy



