# Helm Chart

Helm Chart是kubernetes应用快速部署的应用模板， 现在应用广泛。

在本地部署的时候， 可以是`目录`，也可以`tgz文件`

每个 chart 都必须有一个版本号。版本必须遵循 [SemVer 2](https://semver.org/) 标准， 即`主版本号.次版本号.修订号`的格式

# Chart目录结构


```
<chart目录>
├── Chart.yaml # chart版本和配置信息
├── charts     # 依赖目录,包含了这个chart依赖的其他charts，可以是.tgz或者目录，不能以"_","."开头
├── requirements.yaml   #列出chart的依赖
├── README.md   # 写了chart介绍、安装步骤和各个参数的含义等,方便理解
├── templates   # K8s 资源模板信息, 结合 values.yaml 可生成K8s对象的manifest文件
│   ├── NOTES.txt    # helm 提示信息
│   ├── _helpers.tpl # 下划线开头,作为子模板,可被其他模板文件引用,Helm不会交给K8s处理
│   ├── deployment.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml # chart默认值,可以给各个模板使用
```

- `templates/` 目录下的文件都会作为K8s的 manifests 处理
- NOTES.txt 不会被处理
- **“_”** 下划线开头的都不会被处理



## Chart.yaml

`Chart.yaml` 文件是 chart 所必需的。它包含以下字段：

```yaml
apiVersion:  # chart API版本, 通常用"v1" (必填)
name: # chart名称(必填)
version:  # chart版本号，SemVer 2格式(必填)
kubeVersion: # 支持Kubernetes版本号，SemVer 2格式(选填)
description: # 该项目的单句描述 (选填)
keywords:
  - A list of keywords about this project (optional)
home:  #项目主页的URL链接 (选填)
sources:
  - # 源码链接 (选填)
maintainers: # (选填)
  - name: 
    email:
    url:
engine: gotpl  # 模板处理引擎，默认是gotpl (选填)
icon: # 一个SVG或者PNG的链接，作为图标 (选填)
appVersion: # 里面程序的实际版本，不需要是SemVer (选填)
deprecated: # 标明是否废弃 (选填)
tillerVersion: # 需要的tller版本(选填)
```

注意

- `appVersion`表明里面程序的实际版本， 比如一个mysql的chart这个字段值可能是5.8.1，对chart版本没有影响
- 系统包名必须和`version`值一致



## requirements.yaml

在 Helm 中，一个 chart 可能依赖于任何数量的其他 chart。这些依赖关系可以通过 requirements.yaml 文件动态链接或引入 `charts/` 目录并手动管理。

虽然有一些团队需要手动管理依赖关系的优势，但声明依赖关系的首选方法是使用 chart 内部的 `requirements.yaml` 文件。

```yaml
dependencies:
  - name: apache
    version: 1.2.3
    repository: http://example.com/charts
  - name: mysql
    version: 3.2.1
    repository: http://another.example.com/charts
```

- 该 name 字段是 chart 的名称。
- version 字段是 chart 的版本。
- repository 字段是 chart repo 的完整 URL。请注意，**还必须使用 helm repo add 添加该 repo 到本地才能使用**。

有了依赖关系文件，你可以通过运行 `helm dependency update` ，它会使用你的依赖关系文件将所有指定的 chart 下载到你的 `charts/` 目录中。

```
$ helm dep up foochart
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "local" chart repository
...Successfully got an update from the "stable" chart repository
...Successfully got an update from the "example" chart repository
...Successfully got an update from the "another" chart repository
Update Complete. Happy Helming!
Saving 2 charts
Downloading apache from repo http://example.com/charts
Downloading mysql from repo http://another.example.com/charts
```

当 `helm dependency update` 检索 chart 时，它会将它们作为 chart 存档存储在 `charts/` 目录中。因此，对于上面的示例，可以在 chart 目录中看到以下文件：

```bash
charts/
  apache-1.2.3.tgz
  mysql-3.2.1.tgz
```

通过 `requirements.yaml` 管理 chart 是一种轻松更新 chart 的好方法，还可以在整个团队中共享 requirements 信息。

此外，`requirements.yaml`中有可能存在 `tags` 和 `condition`字段， 