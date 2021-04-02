我们来根据官方文档实战一下kubebuilder，创建一个CronJob的资源类型

# 初始化

kubebuilder的安装过程省略，可以见我的文档，主要是的前置依赖是 `go` 和 `kustomize`

## 初始化三步

- `go mod init`
- `kubebuilder init`
- `kubebuilder create api`

如果使用的文件夹不在`GOPATH`下面，需要先用go mod初始化

```bash
go mod init tutorial.kubebuilder.io
```

然后项目初始化

```bash
kubebuilder init --domain tutorial.kubebuilder.io
······
go fmt ./...
go vet ./...
go build -o bin/manager main.go
Next: define a resource with:
$ kubebuilder create api
```

> 如果出现`which: no controller-gen`的错误，说明`$GOPATH/bin`不在`$PATH`环境变量中，此时将`$GOPATH/bin/controller-gen`程序放到`/bin/`目录即可，root用户的话是`/root/go/bin`

然后创建api

```bash
$kubebuilder create api --group batch --version v1 --kind CronJob
Create Resource [y/n]
y
Create Controller [y/n]
y
Writing scaffold for you to edit...
api/v1/cronjob_types.go
controllers/cronjob_controller.go
Running make:
$ make
/root/go/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
go fmt ./...
go vet ./...
go build -o bin/manager main.go
```

## init

init之后文件夹里面可以看到生成的文件

```bash
├── bin
│   └── manager
├── config
│   ├── certmanager
│   │   ├── certificate.yaml
│   │   ├── kustomization.yaml
│   │   └── kustomizeconfig.yaml
│   ├── default
│   │   ├── kustomization.yaml
│   │   ├── manager_auth_proxy_patch.yaml
│   │   ├── manager_webhook_patch.yaml
│   │   └── webhookcainjection_patch.yaml
│   ├── manager
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── prometheus
│   │   ├── kustomization.yaml
│   │   └── monitor.yaml
│   ├── rbac
│   │   ├── auth_proxy_client_clusterrole.yaml
│   │   ├── auth_proxy_role_binding.yaml
│   │   ├── auth_proxy_role.yaml
│   │   ├── auth_proxy_service.yaml
│   │   ├── kustomization.yaml
│   │   ├── leader_election_role_binding.yaml
│   │   ├── leader_election_role.yaml
│   │   └── role_binding.yaml
│   └── webhook
│       ├── kustomization.yaml
│       ├── kustomizeconfig.yaml
│       └── service.yaml
├── Dockerfile
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
├── main.go
├── Makefile
└── PROJECT
```

- `config`：该文件夹包含了所有配置相关的文件，其中 `default` 文件夹包含标准配置启动控制器；`manager` 文件含有在集群中以 pod 的形式启动控制器的配置； 
- `main.go` : 项目入口函数

### main.go

main 文件最开始是 import 一些基本库，尤其是：

- 核心的 [控制器运行时](https://pkg.go.dev/sigs.k8s.io/controller-runtime?tab=doc) 库  
- 默认的控制器运行时日志库-- Zap (后续会对它作更多的介绍)

```go
package main

import (
······
    "k8s.io/apimachinery/pkg/runtime"
    _ "k8s.io/client-go/plugin/pkg/client/auth/gcp"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/cache"
    "sigs.k8s.io/controller-runtime/pkg/log/zap"
    // +kubebuilder:scaffold:imports
)
```

每一组控制器都需要一个 [*Scheme*](https://book.kubebuilder.io/cronjob-tutorial/gvks.html#err-but-whats-that-scheme-thing)，它提供了 Kinds 和相应的 Go 类型之间的映射。我们将在编写 API 定义的时候再谈一谈 Kinds，所以现在只需要记住它就好。

```go
var (
    scheme   = runtime.NewScheme()
    setupLog = ctrl.Log.WithName("setup")
)

func init() {
    // +kubebuilder:scaffold:scheme
}
```

然后是main()函数

```go
func main() {
    var metricsAddr string
    flag.StringVar(&metricsAddr, "metrics-addr", ":8080", "The address the metric endpoint binds to.")
    flag.Parse()

    ctrl.SetLogger(zap.New(zap.UseDevMode(true)))

    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{Scheme: scheme, MetricsBindAddress: metricsAddr})
    if err != nil {
        setupLog.Error(err, "unable to start manager")
        os.Exit(1)
    }
```

这段代码的核心逻辑比较简单:

- 我们通过 flag 库解析入参
- 我们实例化了一个[*manager*](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/manager#Manager)，它记录着我们所有控制器的运行情况，以及设置共享缓存和API服务器的客户端（注意，我们把我们的 Scheme 的信息告诉了 manager）。
- 运行 manager，它反过来运行我们所有的控制器和 webhooks。manager 状态被设置为 Running，直到它收到一个优雅停机 (graceful shutdown) 信号。这样一来，当我们在 Kubernetes 上运行时，我们就可以优雅地停止 pod。

虽然我们现在还没有任何业务代码可供执行，但请记住 `+kubebuilder:scaffold:builder` 的注释 

注意：Manager 可以通过以下方式限制控制器可以监听资源的命名空间。

```go
    mgr, err = ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme:             scheme,
        Namespace:          namespace,
        MetricsBindAddress: metricsAddr,
    })
```

上面的例子将把你的项目改成只监听单一的命名空间。在这种情况下，建议通过将默认的 ClusterRole 和 ClusterRoleBinding 分别替换为 Role 和 RoleBinding 来限制所提供给这个命名空间的授权。

另外，也可以使用 [MultiNamespacedCacheBuilder](https://pkg.go.dev/github.com/kubernetes-sigs/controller-runtime/pkg/cache#MultiNamespacedCacheBuilder) 来监听特定的命名空间。

```go
    var namespaces []string // List of Namespaces

    mgr, err = ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
        Scheme:             scheme,
        NewCache:           cache.MultiNamespacedCacheBuilder(namespaces),
        MetricsBindAddress: fmt.Sprintf("%s:%d", metricsHost, metricsPort),
    })
```

更多信息请参见 [MultiNamespacedCacheBuilder](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/cache?tab=doc#MultiNamespacedCacheBuilder)

```go
    // +kubebuilder:scaffold:builder
    setupLog.Info("starting manager")
    if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
        setupLog.Error(err, "problem running manager")
        os.Exit(1)
    }
}
```



## create api

然后看看项目文件夹，主要多生成了下面3个文件夹

```bash
├── api
│   └── v1
│       ├── cronjob_types.go   #包含了新Kind的内容
│       ├── groupversion_info.go 
│       └── zz_generated.deepcopy.go 
├── config
│   ├── crd
│   │   ├── kustomization.yaml
│   │   ├── kustomizeconfig.yaml
│   │   └── patches
│   │       ├── cainjection_in_cronjobs.yaml
│   │       └── webhook_in_cronjobs.yaml
├── controllers
│   ├── cronjob_controller.go  #控制器主要逻辑
│   └── suite_test.go
```

其中 `cronjob_types.go` 和 `cronjob_controller.go` 是主要需要修改的文件

### cronjob_types.go

```go
// 包含所有 Kubernetes 种类共有的元数据
import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// CronJobSpec 定义了 CronJob 期待的状态
type CronJobSpec struct {
  // 注意: json 标签是必需的。为了能够序列化字段，任何你添加的新的字段一定有json标签。
}

// CronJobStatus 定义了 CronJob 观察的的状态
type CronJobStatus struct {
}

// +kubebuilder:object:root=true

// CronJob 是 cronjobs API 的 Schema
type CronJob struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   CronJobSpec   `json:"spec,omitempty"`
	Status CronJobStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true

// CronJobList contains a list of CronJob
type CronJobList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []CronJob `json:"items"`
}

func init() {
	SchemeBuilder.Register(&CronJob{}, &CronJobList{})
}
```

### groupversion_info.go

`groupversion_info.go` 包含了关于 group-version 的一些元数据，注册到schema中

### zz_generated.deepcopy.go

`zz_generated.deepcopy.go` 包含了前述 `runtime.Object` 接口的自动实现，这些实现标记了代表 `Kinds` 的所有根类型。

`runtime.Object` 接口的核心是一个深拷贝方法，即 `DeepCopyObject`。

controller-tools 中的 `object` 生成器也能够为每一个根类型以及其子类型生成另外两个易用的方法：`DeepCopy` 和 `DeepCopyInto`。

### cronjob_controller.go

> 控制器调整实际状态到期望状态的过程称为**reconciling**(调谐)，在 controller-runtime 中，为特定种类实现 reconciling 的逻辑被称为 [*Reconciler*](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/reconcile)。 Reconciler 接受一个对象的名称，并返回我们是否需要再次尝试（例如在错误或周期性控制器的情况下，如 HorizontalPodAutoscaler）。

```go
// 基本的 reconciler 结构。几乎每一个调节器都需要记录日志，并且能够获取对象，所以可以直接使用
type CronJobReconciler struct {
	client.Client
	Log    logr.Logger
	Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs/status,verbs=get;update;patch

func (r *CronJobReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
	_ = context.Background()
	_ = r.Log.WithValues("cronjob", req.NamespacedName)

	// your logic here

	return ctrl.Result{}, nil
}
// 将Reconcile 添加到 manager 中，这样当 manager 启动时它就会被启动
func (r *CronJobReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&batchv1.CronJob{}).
		Complete(r)
}
```

最主要的逻辑就在 `Reconcile()` 函数中

- `Reconcile` 实际上是对单个对象进行调谐。我们的 Request 只是有一个名字，我们可以使用 client 从缓存中获取这个对象。

- 我们返回一个空的结果，没有错误，这就向 controller-runtime 表明我们已经成功地对这个对象进行了调谐，在有一些变化之前不需要再尝试调谐。
- 大多数控制器需要一个日志句柄和一个上下文，所以我们在 Reconcile 中将他们初始化
  - 上下文是用来允许取消请求的，也或者是实现 tracing 等功能。它是所有 client 方法的第一个参数。`Background` 上下文只是一个基本的上下文，没有任何额外的数据或超时时间限制
  - runtime通过一个名为`logr`的库使用结构化的日志记录。日志记录的工作原理是将键值对附加到静态消息中。我们可以预先分配一些对，让这些键值对附加到这个调和器的所有日志行

# 设计编码

## API设计

通常来说，CronJob 由以下几部分组成：

- 一个时间表（ CronJob 中的 cron ）
- 要运行的 Job 模板（ CronJob 中的 Job ）

当然 CronJob 还需要一些额外的东西，使得它更加易用

- 一个已经启动的 Job 的超时时间（如果该 Job 执行超时，那么我们会将在下次调度的时候重新执行该 Job）。
- 如果多个 Job 同时运行，该怎么办（我们要等待吗？还是停止旧的 Job ？）
- 暂停 CronJob 运行的方法，以防出现问题。
- 对旧 Job 历史的限制

使用几个标记（`// +comment`）来指定额外的元数据。在生成 CRD 清单时，[controller-tools](https://github.com/kubernetes-sigs/controller-tools) 将使用这些数据。我们稍后将看到，controller-tools 也将使用 GoDoc 来生成字段的描述

```go
type CronJobSpec struct {
    // +kubebuilder:validation:MinLength=0

    // The schedule in Cron format, see https://en.wikipedia.org/wiki/Cron.
    Schedule string `json:"schedule"`

    // +kubebuilder:validation:Minimum=0

    // Optional deadline in seconds for starting the job if it misses scheduled
    // time for any reason.  Missed jobs executions will be counted as failed ones.
    // +optional
    StartingDeadlineSeconds *int64 `json:"startingDeadlineSeconds,omitempty"`

    // Specifies how to treat concurrent executions of a Job.
    // Valid values are:
    // - "Allow" (default): allows CronJobs to run concurrently;
    // - "Forbid": forbids concurrent runs, skipping next run if previous run hasn't finished yet;
    // - "Replace": cancels currently running job and replaces it with a new one
    // +optional
    ConcurrencyPolicy ConcurrencyPolicy `json:"concurrencyPolicy,omitempty"`

    // This flag tells the controller to suspend subsequent executions, it does
    // not apply to already started executions.  Defaults to false.
    // +optional
    Suspend *bool `json:"suspend,omitempty"`

    // Specifies the job that will be created when executing a CronJob.
    JobTemplate batchv1beta1.JobTemplateSpec `json:"jobTemplate"`

    // +kubebuilder:validation:Minimum=0

    // The number of successful finished jobs to retain.
    // This is a pointer to distinguish between explicit zero and not specified.
    // +optional
    SuccessfulJobsHistoryLimit *int32 `json:"successfulJobsHistoryLimit,omitempty"`

    // +kubebuilder:validation:Minimum=0

    // The number of failed finished jobs to retain.
    // This is a pointer to distinguish between explicit zero and not specified.
    // +optional
    FailedJobsHistoryLimit *int32 `json:"failedJobsHistoryLimit,omitempty"`
}
```

我们定义了一个自定义类型来保存我们的并发策略。实际上，它的底层类型是 string，但该类型给出了额外的文档，并允许我们在类型上附加验证，而不是在字段上验证，使验证逻辑更容易复用。

```go
// ConcurrencyPolicy describes how the job will be handled.
// Only one of the following concurrent policies may be specified.
// If none of the following policies is specified, the default one
// is AllowConcurrent.
// +kubebuilder:validation:Enum=Allow;Forbid;Replace
type ConcurrencyPolicy string

const (
    // AllowConcurrent allows CronJobs to run concurrently.
    AllowConcurrent ConcurrencyPolicy = "Allow"

    // ForbidConcurrent forbids concurrent runs, skipping next run if previous
    // hasn't finished yet.
    ForbidConcurrent ConcurrencyPolicy = "Forbid"

    // ReplaceConcurrent cancels currently running job and replaces it with a new one.
    ReplaceConcurrent ConcurrencyPolicy = "Replace"
)
```

接下来，让我们设计一下我们的 status，它表示实际看到的状态。它包含了我们希望用户或其他控制器能够轻松获得的任何信息。

我们将保存一个正在运行的 Jobs，以及我们最后一次成功运行 Job 的时间。注意，我们使用 `metav1.Time` 而不是 `time.Time` 来保证序列化的兼容性以及稳定性，如上所述。

```go
// CronJobStatus defines the observed state of CronJob
type CronJobStatus struct {
    // INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
    // Important: Run "make" to regenerate code after modifying this file

    // A list of pointers to currently running jobs.
    // +optional
    Active []corev1.ObjectReference `json:"active,omitempty"`

    // Information when was the last time the job was successfully scheduled.
    // +optional
    LastScheduleTime *metav1.Time `json:"lastScheduleTime,omitempty"`
}
```

## Controller设计

CronJob 控制器的基本逻辑如下:

1. 根据名称加载定时任务
2. 列出所有有效的 job，更新其状态
3. 根据保留的历史版本数清理版本过旧的 job
4. 检查当前 CronJob 是否被挂起(如果被挂起，则不执行任何操作)
5. 计算 job 下一个定时执行时间
6. 如果 job 符合执行时机，没有超出截止时间，且不被并发策略阻塞，执行该 job
7. 当任务进入运行状态或到了下一次执行时间， job 重新排队

先引进依赖库

```go
package controllers

import (
    "context"
    "fmt"
    "sort"
    "time"

    "github.com/go-logr/logr"
    "github.com/robfig/cron"
    kbatch "k8s.io/api/batch/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    ref "k8s.io/client-go/tools/reference"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"

    batch "tutorial.kubebuilder.io/project/api/v1"
)
```

接下来，我们需要一个时钟好让我们在测试中模拟计时。

```go
// CronJob 调谐器对 CronJob 对象进行调谐
type CronJobReconciler struct {
    client.Client
    Log    logr.Logger
    Scheme *runtime.Scheme
    Clock
}
```

注意，我们需要获得RBAC权限——我们需要一些额外权限去 创建和管理job，添加如下一些字段

```go
// +kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=cronjobs/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=batch,resources=jobs,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=batch,resources=jobs/status,verbs=get
```

控制器的核心部分——调谐逻辑

```go
var (
    scheduledTimeAnnotation = "batch.tutorial.kubebuilder.io/scheduled-at"
)

func (r *CronJobReconciler) Reconcile(req ctrl.Request) (ctrl.Result, error) {
    ctx := context.Background()
    log := r.Log.WithValues("cronjob", req.NamespacedName)
```

### 1、根据名称加载定时任务

通过 client 获取定时任务。所有 client 方法第一个参数都是 context（用于取消定时任务）作为 第一个参数，把请求对象信息作为最后一个参数。Get 方法例外，它把 [`NamespacedName`](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/client?tab=doc#ObjectKey) 作为中间的第二个参数（大多数方法都没有中间的NamespacedName参数，下文会提到）

许多client方法的最后一个参数接受变长参数。

```go
    var cronJob batch.CronJob
    if err := r.Get(ctx, req.NamespacedName, &cronJob); err != nil {
        log.Error(err, "unable to fetch CronJob")
        //忽略掉 not-found 错误，它们不能通过重新排队修复（要等待新的通知）
        //在删除一个不存在的对象时，可能会报这个错误。
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
```

### 2、列出所有有效 job，更新它们的状态

为确保每个 job 的状态都会被更新到，我们需要列出某个 CronJob 在当前命名空间下的所有 job。 和 Get 方法类似，我们可以使用 List 方法来列出 CronJob 下所有的 job。注意，我们使用变长参数 来映射命名空间和任意多个匹配变量（实际上相当于是建立了一个索引）。

```go
    var childJobs kbatch.JobList
    if err := r.List(ctx, &childJobs, client.InNamespace(req.Namespace), client.MatchingFields{jobOwnerKey: req.Name}); err != nil {
        log.Error(err, "unable to list child Jobs")
        return ctrl.Result{}, err
    }
```

> 索引的作用？
>
> 调谐器会获取 cronjob 下的所有 job 以更新它们状态。随着 cronjob 数量的增加，遍历全部 conjob 查找会变的相当低效。为了提高查询效率，这些任务会根据控制器名称建立索引。缓存后的 job 对象会 被添加上一个 jobOwnerKey 字段。这个字段引用其归属控制器和函数作为索引。在下文中，我们将配置 manager 作为这个字段的索引

查找到所有的 job 后，将其归类为 active，successful，failed 三种类型，同时持续跟踪其 最新的执行情况以更新其状态。牢记，status 值应该是从实际的运行状态中实时获取。从 cronjob 中读取 job 的状态通常不是一个好做法。应该从每次执行状态中获取。我们后续也采用这种方法。

我们可以检查一个 job 是否已处于 “finished” 状态，使用 status 条件还可以知道它是 succeeded 或 failed 状态。

```go
    // 找出所有有效的 job
    var activeJobs []*kbatch.Job
    var successfulJobs []*kbatch.Job
    var failedJobs []*kbatch.Job
    var mostRecentTime *time.Time // 记录其最近一次运行时间以便更新状态
```

当一个 job 被标记为 “succeeded” 或 “failed” 时，我们认为这个任务处于 “finished” 状态。 Status conditions 允许我们给 job 对象添加额外的状态信息，开发人员或控制器可以通过 这些校验信息来检查 job 的完成或健康状态。

```go
    isJobFinished := func(job *kbatch.Job) (bool, kbatch.JobConditionType) {
        for _, c := range job.Status.Conditions {
            if (c.Type == kbatch.JobComplete || c.Type == kbatch.JobFailed) && c.Status == corev1.ConditionTrue {
                return true, c.Type
            }
        }

        return false, ""
    }
```

使用辅助函数来提取创建 job 时注释中排定的执行时间

```go
    for i, job := range childJobs.Items {
        _, finishedType := isJobFinished(&job)
        switch finishedType {
        case "": // ongoing
            activeJobs = append(activeJobs, &childJobs.Items[i])
        case kbatch.JobFailed:
            failedJobs = append(failedJobs, &childJobs.Items[i])
        case kbatch.JobComplete:
            successfulJobs = append(successfulJobs, &childJobs.Items[i])
        }

        //将启动时间存放在注释中，当job生效时可以从中读取
        scheduledTimeForJob, err := getScheduledTimeForJob(&job)
        if err != nil {
            log.Error(err, "unable to parse schedule time for child job", "job", &job)
            continue
        }
        if scheduledTimeForJob != nil {
            if mostRecentTime == nil {
                mostRecentTime = scheduledTimeForJob
            } else if mostRecentTime.Before(*scheduledTimeForJob) {
                mostRecentTime = scheduledTimeForJob
            }
        }
    }

    if mostRecentTime != nil {
        cronJob.Status.LastScheduleTime = &metav1.Time{Time: *mostRecentTime}
    } else {
        cronJob.Status.LastScheduleTime = nil
    }
    cronJob.Status.Active = nil
    for _, activeJob := range activeJobs {
        jobRef, err := ref.GetReference(r.Scheme, activeJob)
        if err != nil {
            log.Error(err, "unable to make reference to active job", "job", activeJob)
            continue
        }
        cronJob.Status.Active = append(cronJob.Status.Active, *jobRef)
    }
```

此处会记录我们观察到的 job 数量。为便于调试，略微提高日志级别。注意，这里没有使用 格式化字符串，使用由键值对构成的固定格式信息来输出日志。这样更易于过滤和查询日志

```go
    log.V(1).Info("job count", "active jobs", len(activeJobs), "successful jobs", len(successfulJobs), "failed jobs", len(failedJobs))
```

使用收集到日期信息来更新 CRD 状态。和之前类似，通过 client 来完成操作。 针对 status 这一子资源，我们可以使用`Status`部分的`Update`方法。

status 子资源会忽略掉对 spec 的变更。这与其它更新操作的发生冲突的风险更小， 而且实现了权限分离。

```go
    if err := r.Status().Update(ctx, &cronJob); err != nil {
        log.Error(err, "unable to update CronJob status")
        return ctrl.Result{}, err
    }
```

更新状态后，后续要确保状态符合我们在 spec 定下的预期。

### 3、根据保留的历史版本数清理过旧的 job

我们先清理掉一些版本太旧的 job，这样可以不用保留太多无用的 job

```go
    // 注意: 删除操作采用的“尽力而为”策略
    // 如果个别job删除失败了，不会将其重新排队，直接结束删除操作
    if cronJob.Spec.FailedJobsHistoryLimit != nil {
        sort.Slice(failedJobs, func(i, j int) bool {
            if failedJobs[i].Status.StartTime == nil {
                return failedJobs[j].Status.StartTime != nil
            }
            return failedJobs[i].Status.StartTime.Before(failedJobs[j].Status.StartTime)
        })
        for i, job := range failedJobs {
            if int32(i) >= int32(len(failedJobs))-*cronJob.Spec.FailedJobsHistoryLimit {
                break
            }
            if err := r.Delete(ctx, job, client.PropagationPolicy(metav1.DeletePropagationBackground)); client.IgnoreNotFound(err) != nil {
                log.Error(err, "unable to delete old failed job", "job", job)
            } else {
                log.V(0).Info("deleted old failed job", "job", job)
            }
        }
    }

    if cronJob.Spec.SuccessfulJobsHistoryLimit != nil {
        sort.Slice(successfulJobs, func(i, j int) bool {
            if successfulJobs[i].Status.StartTime == nil {
                return successfulJobs[j].Status.StartTime != nil
            }
            return successfulJobs[i].Status.StartTime.Before(successfulJobs[j].Status.StartTime)
        })
        for i, job := range successfulJobs {
            if int32(i) >= int32(len(successfulJobs))-*cronJob.Spec.SuccessfulJobsHistoryLimit {
                break
            }
            if err := r.Delete(ctx, job, client.PropagationPolicy(metav1.DeletePropagationBackground)); (err) != nil {
                log.Error(err, "unable to delete old successful job", "job", job)
            } else {
                log.V(0).Info("deleted old successful job", "job", job)
            }
        }
    }
```

### 4、检查是否被挂起

如果当前 cronjob 被挂起，不会再运行其下的任何 job，我们将其停止。这对于某些 job 出现异常 的排查非常有用。我们无需删除 cronjob 来中止其后续其他 job 的运行。

```go
    if cronJob.Spec.Suspend != nil && *cronJob.Spec.Suspend {
        log.V(1).Info("cronjob suspended, skipping")
        return ctrl.Result{}, nil
    }
```

### 5、计算 job 下一次执行时间

如果 cronjob 没被挂起，则我们需要计算它的下一次执行时间， 同时检查是否有遗漏的执行没被处理

借助强大的 cron 库，我们可以轻易的计算出定时任务的下一次执行时间。 我们根据最近一次的执行时间来计算下一次执行时间，如果没有找到最近 一次执行时间，则根据定时任务的创建时间来计算。

如果遗漏了很多次执行并且没有为这些执行设置截止时间。我们将其忽略 以避免异常造成控制器的频繁重启和资源紧张。

如果遗漏的执行次数并不多，我们返回最近一次的执行时间和下一次将要 执行的时间，这样我们可以知道该何时去调谐。

```go
    getNextSchedule := func(cronJob *batch.CronJob, now time.Time) (lastMissed time.Time, next time.Time, err error) {
        sched, err := cron.ParseStandard(cronJob.Spec.Schedule)
        if err != nil {
            return time.Time{}, time.Time{}, fmt.Errorf("Unparseable schedule %q: %v", cronJob.Spec.Schedule, err)
        }

        // 出于优化的目的，我们可以使用点技巧。从上一次观察到的执行时间开始执行，
        // 这个执行时间可以被在这里被读取。但是意义不大，因为我们刚更新了这个值。

        var earliestTime time.Time
        if cronJob.Status.LastScheduleTime != nil {
            earliestTime = cronJob.Status.LastScheduleTime.Time
        } else {
            earliestTime = cronJob.ObjectMeta.CreationTimestamp.Time
        }
        if cronJob.Spec.StartingDeadlineSeconds != nil {
            // 如果开始执行时间超过了截止时间，不再执行
            schedulingDeadline := now.Add(-time.Second * time.Duration(*cronJob.Spec.StartingDeadlineSeconds))

            if schedulingDeadline.After(earliestTime) {
                earliestTime = schedulingDeadline
            }
        }
        if earliestTime.After(now) {
            return time.Time{}, sched.Next(now), nil
        }

        starts := 0
        for t := sched.Next(earliestTime); !t.After(now); t = sched.Next(t) {
            lastMissed = t
            // 一个 CronJob 可能会遗漏多次执行。举个例子，周五5:00pm技术人员下班后，
            // 控制器在5:01pm发生了异常。然后直到周二早上才有技术人员发现问题并
            // 重启控制器。那么所有的以1小时为周期执行的定时任务，在没有技术人员
            // 进一步的干预下，都会有80多个 job 在恢复正常后一并启动（如果 job 允许
            // 多并发和延迟启动）

            // 如果 CronJob 的某些地方出现异常，控制器或 apiservers (用于设置任务创建时间)
            // 的时钟不正确, 那么就有可能出现错过很多次执行时间的情形（跨度可达数十年）
            // 这将会占满控制器的CPU和内存资源。这种情况下，我们不需要列出错过的全部
            // 执行时间。

            starts++
            if starts > 100 {
                // 获取不到最近一次执行时间，直接返回空切片
                return time.Time{}, time.Time{}, fmt.Errorf("Too many missed start times (> 100). Set or decrease .spec.startingDeadlineSeconds or check clock skew.")
            }
        }
        return lastMissed, sched.Next(now), nil
    }
```

```go
    // 计算出定时任务下一次执行时间（或是遗漏的执行时间）
    missedRun, nextRun, err := getNextSchedule(&cronJob, r.Now())
    if err != nil {
        log.Error(err, "unable to figure out CronJob schedule")
        // 重新排队直到有更新修复这次定时任务调度，不必返回错误
        return ctrl.Result{}, nil
    }
```

上述步骤执行完后，将准备好的请求加入队列直到下次执行， 然后确定这些 job 是否要实际执行

```go
    scheduledResult := ctrl.Result{RequeueAfter: nextRun.Sub(r.Now())} // 保存以便别处复用
    log = log.WithValues("now", r.Now(), "next run", nextRun)
```

### 6、如果 job 符合执行时机，并且没有超出截止时间，且不被并发策略阻塞，执行该 job

如果 job 遗漏了一次执行，且还没超出截止时间，把遗漏的这次执行也不上

```go
    if missedRun.IsZero() {
        log.V(1).Info("no upcoming scheduled times, sleeping until next")
        return scheduledResult, nil
    }

    // 确保错过的执行没有超过截止时间
    log = log.WithValues("current run", missedRun)
    tooLate := false
    if cronJob.Spec.StartingDeadlineSeconds != nil {
        tooLate = missedRun.Add(time.Duration(*cronJob.Spec.StartingDeadlineSeconds) * time.Second).Before(r.Now())
    }
    if tooLate {
        log.V(1).Info("missed starting deadline for last run, sleeping till next")
        // TODO(directxman12): events
        return scheduledResult, nil
    }
```

如果确认 job 需要实际执行。我们有三种策略执行该 job。要么先等待现有的 job 执行完后，在启动本次 job； 或是直接覆盖取代现有的job；或是不考虑现有的 job，直接作为新的 job 执行。因为缓存导致的信息有所延迟， 当更新信息后需要重新排队。

```go
    // 确定要 job 的执行策略 —— 并发策略可能禁止多个job同时运行
    if cronJob.Spec.ConcurrencyPolicy == batch.ForbidConcurrent && len(activeJobs) > 0 {
        log.V(1).Info("concurrency policy blocks concurrent runs, skipping", "num active", len(activeJobs))
        return scheduledResult, nil
    }

    // 直接覆盖现有 job
    if cronJob.Spec.ConcurrencyPolicy == batch.ReplaceConcurrent {
        for _, activeJob := range activeJobs {
            // we don't care if the job was already deleted
            if err := r.Delete(ctx, activeJob, client.PropagationPolicy(metav1.DeletePropagationBackground)); client.IgnoreNotFound(err) != nil {
                log.Error(err, "unable to delete active job", "job", activeJob)
                return ctrl.Result{}, err
            }
        }
    }
```

确定如何处理现有 job 后，创建符合我们预期的 job

基于 CronJob 模版构建 job，从模板复制 spec 及对象的元信息。

然后在注解中设置执行时间，这样我们可以在每次的调谐中获取起作为“上一次执行时间”

最后，还需要设置 owner reference字段。当我们删除 CronJob 时，Kubernetes 垃圾收集 器会根据这个字段对应的 job。同时，当某个job状态发生变更（创建，删除，完成）时， controller-runtime 可以根据这个字段识别出要对那个 CronJob 进行调谐。

```go
    constructJobForCronJob := func(cronJob *batch.CronJob, scheduledTime time.Time) (*kbatch.Job, error) {
        // job 名称带上执行时间以确保唯一性，避免排定执行时间的 job 创建两次
        name := fmt.Sprintf("%s-%d", cronJob.Name, scheduledTime.Unix())

        job := &kbatch.Job{
            ObjectMeta: metav1.ObjectMeta{
                Labels:      make(map[string]string),
                Annotations: make(map[string]string),
                Name:        name,
                Namespace:   cronJob.Namespace,
            },
            Spec: *cronJob.Spec.JobTemplate.Spec.DeepCopy(),
        }
        for k, v := range cronJob.Spec.JobTemplate.Annotations {
            job.Annotations[k] = v
        }
        job.Annotations[scheduledTimeAnnotation] = scheduledTime.Format(time.RFC3339)
        for k, v := range cronJob.Spec.JobTemplate.Labels {
            job.Labels[k] = v
        }
        if err := ctrl.SetControllerReference(cronJob, job, r.Scheme); err != nil {
            return nil, err
        }

        return job, nil
    }
```

```go
    // 构建 job
    job, err := constructJobForCronJob(&cronJob, missedRun)
    if err != nil {
        log.Error(err, "unable to construct job from template")
        // job 的 spec 没有变更，无需重新排队
        return scheduledResult, nil
    }

    // ...在集群中创建 job
    if err := r.Create(ctx, job); err != nil {
        log.Error(err, "unable to create Job for CronJob", "job", job)
        return ctrl.Result{}, err
    }

    log.V(1).Info("created Job for CronJob run", "job", job)
```

### 7、当 job 开始运行或到了 job 下一次的执行时间，重新排队

最终我们返回上述预备的结果。我们还需重新排队当任务还有下一次执行时。 这被视作最长截止时间——如果期间发生了变更，例如 job 被提前启动或是提前 结束，或被修改，我们可能会更早进行调谐。

```go
    // 当有 job 进入运行状态后，重新排队，同时更新状态
    return scheduledResult, nil
}
```

### 8、启动 CronJob 控制器

最后，我们还要完善下我们的启动过程。为了让调谐器可以通过 job 的 owner 值快速找到 job。 我们需要一个索引。声明一个索引键，后续我们可以将其用于 client 的虚拟变量名中，从 job 对象中提取索引值。此处的索引会帮我们处理好 namespaces 的映射关系。所以如果 job 有 owner 值，我们快速地获取 owner 值。

另外，我们需要告知 manager，这个控制器拥有哪些 job。当对应的 job 发生变更或被删除时， 自动调用调谐器对 CronJob 进行调谐。

```go
var (
    jobOwnerKey = ".metadata.controller"
    apiGVStr    = batch.GroupVersion.String()
)

func (r *CronJobReconciler) SetupWithManager(mgr ctrl.Manager) error {
    // 此处不是测试，我们需要创建一个真实的时钟
    if r.Clock == nil {
        r.Clock = realClock{}
    }

    if err := mgr.GetFieldIndexer().IndexField(context.Background(), &kbatch.Job{}, jobOwnerKey, func(rawObj runtime.Object) []string {
        //获取 job 对象，提取 owner...
        job := rawObj.(*kbatch.Job)
        owner := metav1.GetControllerOf(job)
        if owner == nil {
            return nil
        }
        // ...确保 owner 是个 CronJob...
        if owner.APIVersion != apiGVStr || owner.Kind != "CronJob" {
            return nil
        }

        // ...是 CronJob，返回
        return []string{owner.Name}
    }); err != nil {
        return err
    }

    return ctrl.NewControllerManagedBy(mgr).
        For(&batch.CronJob{}).
        Owns(&kbatch.Job{}).
        Complete(r)
}
```

## main.go

一般情况下不需要修改，kubebuilder 已经自动添加了一个阻塞调用我们的 CornJob 控制器的 `SetupWithManager` 方法。





# 安装&卸载



