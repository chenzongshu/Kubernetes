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

此外，`requirements.yaml`中有可能存在 `tags` 和 `condition`字段， 这两个字段可以评估并控制应用是否加载，具体解释省略。



# 子模板

又叫`命名模板`, 是限定在一个文件内部的模板，然后给一个名称。在使用命名模板的时候有一个需要特别注意的是：**模板名称是全局的**，如果我们声明了两个相同名称的模板，最后加载的一个模板会覆盖掉另外的模板，由于子 chart 中的模板也是和顶层的模板一起编译的，所以在命名的时候一定要注意，不要重名了。

```go
{{ define "ChartName.TplName" }}
# 模板内容区域
{{ end }}
```

## 作用域

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" . }}
data:
  {{- range $key, $value := .Values.course }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
```

我们在 template 末尾传递了`.`，表示当前的最顶层的作用范围



# 函数

## include

`include`函数允许你引入另一个模板，然后将结果传递给其他模板函数。

例如，下面的模板片段中引用了一个名为`mytpl`的模板，然后将结果转成小写，并用双引号包装起来：

```go
value: {{ include "mytpl" . | lower | quote }}
```

## required

`required`函数允许你根据模板的需要声明特定的值，如果值为空，则默认渲染的时候会报错。下面的这个示例被声明为 .Values.who 是必须的，为空的时候会打印出一段错误提示信息：

```go
value: {{required "A valid .Values.who entry required!" .Values.who }}
```

## indent

由于 YAML 对于缩进级别和空格的重要性, 该函数就是控制缩进的空格的, 可以配合`include`函数以及管道一起使用

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
{{- include "mychart.labels" . | indent 4 }}
data:
  {{- range $key, $value := .Values.course }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
{{- include "mychart.labels" . | indent 2 }}
```

在 labels 区域我们需要4个空格，所以在管道函数`indent`中，传入参数4就可以，而在 data 区域我们只需要2个空格，所以我们传入参数2即可以，

# Hook

`Kubernetes`里面的容器一样，`Helm`也提供了 [Hook](https://docs.helm.sh/developing_charts/#hooks) 的机制，允许 chart 开发人员在 release 的生命周期中的某些节点来进行干预

值得注意的是 Hooks 和普通模板一样工作，但是它们具有特殊的注释，可以使 Helm 以不同的方式使用它们。

Hook 在资源清单中的 metadata 部分用 annotations 的方式进行声明：

```yaml
apiVersion: ...
kind: ....
metadata:
  annotations:
    "helm.sh/hook": "pre-install"
# ...
```

在 Helm 中定义了如下一些可供我们使用的 Hooks：

- 预安装`pre-install`：在模板渲染后，kubernetes 创建任何资源之前执行
- 安装后`post-install`：在所有 kubernetes 资源安装到集群后执行
- 预删除`pre-delete`：在从 kubernetes 删除任何资源之前执行删除请求
- 删除后`post-delete`：删除所有 release 的资源后执行
- 升级前`pre-upgrade`：在模板渲染后，但在任何资源升级之前执行
- 升级后`post-upgrade`：在所有资源升级后执行
- 预回滚`pre-rollback`：在模板渲染后，在任何资源回滚之前执行
- 回滚后`post-rollback`：在修改所有资源后执行回滚请求
- `crd-install`：在运行其他检查之前添加 CRD 资源，只能用于 chart 中其他的资源清单定义的 CRD 资源。



# 流程控制

## if/else

语法：

```
{{ if PIPELINE }}  # Do something
{{ else if OTHER PIPELINE }}  # Do something else
{{ else }}  # Default case
{{ end }}
```

demo:

```
{{- if .Values.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
{{- end -}}
```



## with

with 用于修改作用域，正常 **“.”** 代表全局作用域，with 可以修改 **“.”** 的含义。

语法：

```
{{ with 被引用的对象 }}
. 用来引用with指定的对象
{{ end }}
```

`values.yaml` ：

```
ingress:  
  enabled: false  
  annotations:     
    kubernetes.io/ingress.class: nginx    
    kubernetes.io/tls-acme: "true"
```

使用 with：

```
kind: Ingress
metadata:  
  # 指定引用对象为 ingress.annotations  
  {{- with .Values.ingress.annotations }}  
  annotations:    
    # "." 点代表该对象    
    {{- toYaml . | nindent 4 }}  
  {{- end }}
```



# 调试模板

Helm提供几种调试命令：

```
# 验证chart是否符合最佳实践
helm lint

# 适合渲染本地的chart
helm install --dry-run --debug
helm template --debug chartDir

# 适合用于查看服务器上安装好的
charthelm get manifest
```