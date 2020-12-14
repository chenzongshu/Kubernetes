# 简介

kustomize是sig-cli的一个子项目，它的设计目的是给kubernetes的用户提供一种可以重复使用同一套配置的声明式应用管理，没有新的语法引入。设计思想类似于Docker的镜像分层， 上层需要修改的值覆盖下面的基础配置。

kustomize允许用户将不同环境所共享的配置放在一个文件目录下，而将其他不同的配置放在另外的目录下。这样用户就可以很容易的区分那些值是当前环境所特有的，从而在修改的时候会额外关注。

官方github地址是 ： [kustomize](https://github.com/kubernetes-sigs/kustomize)

# 安装

从 1.14 版本开始，`kubectl` 集成了kustomize

要查看包含 kustomization 文件的目录中的资源，执行下面的命令

```shell
kubectl kustomize <kustomization_directory>
```

要应用这些资源，使用参数 `--kustomize` 或 `-k` 标志来执行 `kubectl apply`：

```shell
kubectl apply -k <kustomization_directory>
```

# 说明

## 文件结构

```bash
~/someApp
├── base
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   └── service.yaml
└── overlays
    ├── development  # dev环境
    │   ├── cpu_count.yaml
    │   ├── kustomization.yaml
    │   └── replica_count.yaml
    └── production   # prd环境
        ├── cpu_count.yaml
        ├── kustomization.yaml
        └── replica_count.yaml
```

如果要构建prd的yaml，执行命令

```
kubectl kustomize ~/someApp/overlays/production
```

## 关键文件

`kustomization.yaml`，只有包含了该文件的文件夹才能执行`kustomize build`，base文件夹或者overlay里面的略有不同，base文件中的会指明使用哪些资源yaml文件，overlay的指明需要覆盖的文件，和base目录

```
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - service.yaml
  - deployment.yaml
```

overlay：

```
# overlay/prd/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

bases:
- ../../base

patchesStrategicMerge:
- custom-env.yaml
```

- **执行kustomize命令的目录必须包含`kustomization.yaml`文件**
- base可以被多个overlay引用
- overlay可以拥有多个base

## 补丁

 Kustomize 通过 `patchesStrategicMerge` 和 `patchesJson6902` 支持不同的打补丁机制。

### patchesStrategicMerge

`patchesStrategicMerge` 的内容是一个文件路径的列表，其中每个文件都应可解析为 `策略性合并补丁`。 **补丁文件中的名称必须与已经加载的资源的名称匹配**。 建议构造规模较小的、仅做一件事情的补丁。 例如，构造一个补丁来增加 Deployment 的副本个数；构造另外一个补丁来设置内存限制。

```bash
# 创建 deployment.yaml 文件
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

# 生成一个补丁 increase_replicas.yaml
cat <<EOF > increase_replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3
EOF

# 生成另一个补丁 set_memory.yaml
cat <<EOF > set_memory.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  template:
    spec:
      containers:
      - name: my-nginx
        resources:
        limits:
          memory: 512Mi
EOF

cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml
patchesStrategicMerge:
- increase_replicas.yaml
- set_memory.yaml
EOF
```



### patchesJson6902

 `patchesJson6902` 来应用 `JSON 补丁` 的能力。为了给 JSON 补丁找到正确的资源，需要在 `kustomization.yaml` 文件中指定资源的 组（group）、版本（version）、类别（kind）和名称（name）。 

```bash
# 创建一个 deployment.yaml 文件
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

# 创建一个 JSON 补丁文件
cat <<EOF > patch.yaml
- op: replace
  path: /spec/replicas
  value: 3
EOF

# 创建一个 kustomization.yaml
cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml

patchesJson6902:
- target:
    group: apps
    version: v1
    kind: Deployment
    name: my-nginx
  path: patch.yaml
EOF
```

### 对象字段注入

除了补丁之外，Kustomize 还提供定制容器镜像或者将其他对象的字段值注入到容器 中的能力，并且不需要创建补丁。 例如，你可以通过在 `kustomization.yaml` 文件的 `images` 字段设置新的镜像来 更改容器中使用的镜像。

```
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml
images:
- name: nginx
  newName: my.image.registry/nginx
  newTag: 1.4.0
EOF
```

# 功能列表

内容不详细介绍，见官方文档：

https://kubernetes.io/zh/docs/tasks/manage-kubernetes-objects/kustomization/#composing-and-customizing-resources 

| 字段                  | 类型                                                         | 解释                                                         |
| --------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| namespace             | string                                                       | 为所有资源添加名字空间                                       |
| namePrefix            | string                                                       | 此字段的值将被添加到所有资源名称前面                         |
| nameSuffix            | string                                                       | 此字段的值将被添加到所有资源名称后面                         |
| commonLabels          | map[string]string                                            | 要添加到所有资源和选择算符的标签                             |
| commonAnnotations     | map[string]string                                            | 要添加到所有资源的注解                                       |
| resources             | []string                                                     | 列表中的每个条目都必须能够解析为现有的资源配置文件           |
| configmapGenerator    | [][ConfigMapArgs](https://github.com/kubernetes-sigs/kustomize/blob/release-kustomize-v4.0/api/types/kustomization.go#L99) | 列表中的每个条目都会生成一个 ConfigMap                       |
| secretGenerator       | [][SecretArgs](https://github.com/kubernetes-sigs/kustomize/blob/release-kustomize-v4.0/api/types/kustomization.go#L106) | 列表中的每个条目都会生成一个 Secret                          |
| generatorOptions      | [GeneratorOptions](https://github.com/kubernetes-sigs/kustomize/blob/release-kustomize-v4.0/api/types/kustomization.go#L109) | 更改所有 ConfigMap 和 Secret 生成器的行为                    |
| bases                 | []string                                                     | 列表中每个条目都应能解析为一个包含 kustomization.yaml 文件的目录 |
| patchesStrategicMerge | []string                                                     | 列表中每个条目都能解析为某 Kubernetes 对象的策略性合并补丁   |
| patchesJson6902       | [][Json6902](https://github.com/kubernetes-sigs/kustomize/blob/release-kustomize-v4.0/api/types/patchjson6902.go#L8) | 列表中每个条目都能解析为一个 Kubernetes 对象和一个 JSON 补丁 |
| vars                  | [][Var](https://github.com/kubernetes-sigs/kustomize/blob/master/api/types/var.go#L31) | 每个条目用来从某资源的字段来析取文字                         |
| images                | [][Image](https://github.com/kubernetes-sigs/kustomize/tree/master/api/types/image.go#L23) | 每个条目都用来更改镜像的名称、标记与/或摘要，不必生成补丁    |
| configurations        | []string                                                     | 列表中每个条目都应能解析为一个包含 [Kustomize 转换器配置](https://github.com/kubernetes-sigs/kustomize/tree/master/examples/transformerconfigs) 的文件 |
| crds                  | []string                                                     | 列表中每个条目都赢能够解析为 Kubernetes 类别的 OpenAPI 定义文件 |



