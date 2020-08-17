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
├── .helmignore，定义了在helm package时哪些文件不会打包到Chart包tgz中
├── templates   # K8s 资源模板信息, 结合 values.yaml 可生成K8s对象的manifest文件
│   ├── NOTES.txt    # helm 提示信息,提供了安装后的使用说明，在Chart安装和升级等操作
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



## 内置对象

内置对象可以传递到模板中, 类似编程语言中的反射, 下面列了一些常见的内置对象

```
Release：这个对象描述了 release 本身。它里面有几个对象：
Release.Name：release 名称
Release.Time：release 的时间
Release.Namespace：release 的 namespace（如果清单未覆盖）
Release.Service：release 服务的名称（始终是 Tiller）。
Release.Revision：此 release 的修订版本号。它从 1 开始，每 helm upgrade 一次增加一个。
Release.IsUpgrade：如果当前操作是升级或回滚，则将其设置为 true。
Release.IsInstall：如果当前操作是安装，则设置为 true。
Values：从 values.yaml 文件和用户提供的文件传入模板的值。默认情况下，Values 是空的。
Chart：Chart.yaml 文件的内容。任何数据 Chart.yaml 将在这里访问。例如 {{.Chart.Name}}-{{.Chart.Version}} 将打印出来 mychart-0.1.0。chart 指南中 Charts Guide 列出了可用字段
Files：这提供对 chart 中所有非特殊文件的访问。虽然无法使用它来访问模板，但可以使用它来访问 chart 中的其他文件。请参阅 "访问文件" 部分。
Files.Get 是一个按名称获取文件的函数（.Files.Get config.ini）
Files.GetBytes 是将文件内容作为字节数组而不是字符串获取的函数。这对于像图片这样的东西很有用。
Capabilities：这提供了关于 Kubernetes 集群支持的功能的信息。
Capabilities.APIVersions 是一组版本信息。
Capabilities.APIVersions.Has $version 指示是否在群集上启用版本（batch/v1）。
Capabilities.KubeVersion 提供了查找 Kubernetes 版本的方法。它具有以下值：Major，Minor，GitVersion，GitCommit，GitTreeState，BuildDate，GoVersion，Compiler，和 Platform。
Capabilities.TillerVersion 提供了查找 Tiller 版本的方法。它具有以下值：SemVer，GitCommit，和 GitTreeState。
Template：包含有关正在执行的当前模板的信息
Name：到当前模板的 namespace 文件路径（例如 mychart/templates/mytemplate.yaml）
BasePath：当前 chart 模板目录的 namespace 路径（例如 mychart/templates）。
```

内置值始终以大写字母开头。这符合Go的命名约定。



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



# define/template

`define` 允许我们在模板文件中创建一个命名模板, 然后使用`template`来使用该模板

命名模板是限定在一个文件内部的模板，然后给一个名称。在使用命名模板的时候有一个需要特别注意的是：**模板名称是全局的**，如果我们声明了两个相同名称的模板，最后加载的一个模板会覆盖掉另外的模板，由于子 chart 中的模板也是和顶层的模板一起编译的，所以在命名的时候一定要注意，不要重名了。

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

在 labels 区域我们需要4个空格，所以在管道函数`indent`中，传入参数4就可以，而在 data 区域我们只需要2个空格，所以我们传入参数2即可以

## quote

给字符串加上双引号

##  Default

这个函数允许你指定一个默认值：

```
drink: {{ .Values.favorite.drink | default "tea" | quote }}
```

## 操作符

操作符是按照函数的方式实现的，返回一个布尔值。使用 `eq, ne, lt, gt, and, or, not`时，要将它们**放到句子的最前面**，后面跟上对应的参数。**多个操作符一起使用时，可以用小括号包起来**。

```
{{/* include the body of this if statement when the variable .Values.fooString exists and is set to "foo" */}}
{{ if and .Values.fooString (eq .Values.fooString "foo") }}
    {{ ... }}
{{ end }}


{{/* do not include the body of this if statement because unset variables evaluate to false and .Values.setVariable was negated with the not function. */}}
{{ if or .Values.anUnsetVariable (not .Values.aSetVariable) }}
   {{ ... }}
{{ end }}
```



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

## range 循环

Helm 可以通过 `range` 操作符来迭代集合。

在 `values.yaml` 中增加列表：

```
favorite:
  drink: coffee
  food: pizza
pizzaToppings:
  - mushrooms
  - cheese
  - peppers
  - onions
```

现在修改下 ConfigMap 的模板，来打印出上面的列表：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  {{- with .Values.favorite }}
  drink: {{ .drink | default "tea" | quote }}
  food: {{ .food | upper | quote }}
  {{- end }}
  toppings: |-
    {{- range .Values.pizzaToppings }}
    - {{ . | title | quote }}
    {{- end }}
```

和 `with`一样，`range`也可以设置作用域，所以在这里，`.` 表示的是 `pizzaToppings`这个作用域。我们能把 `.` 直接传递给管道使用 `{{ . | title | quote }}`。

运行上面的模板，结果如下：

```
# Source: mychart/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: edgy-dragonfly-configmap
data:
  myvalue: "Hello World"
  drink: "coffee"
  food: "PIZZA"
  toppings: |-
    - "Mushrooms"
    - "Cheese"
    - "Peppers"
    - "Onions"
```

`toppings: |-` 表示这是一个多行的字符串。



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

当你的 YAML 没有解析，但想看看生成了什么时，检索 YAML 的一个简单方法是注释模板中的问题部分，然后重新运行 `helm install --dry-run --debug`：

```yaml
apiVersion: v1
# some: problem section
# {{ .Values.foo | quote }}
```

以上内容将被完整渲染并返回。

```yaml
apiVersion: v1
# some: problem section
#  "bar"
```

这提供了一种快速查看生成的容的方式，而不会由于YAML分析错误而被阻止。