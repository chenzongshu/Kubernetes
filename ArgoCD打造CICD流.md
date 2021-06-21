# 简介

ArgoCD是一个云原生的CICD流（作流中的每个步骤是通过容器实现）。常用来和gitlab来实现gitops工作流。

Argo 可以让用户用一个类似于传统的 YAML 文件定义的 DSL 来运行多个步骤的 Pipeline。该框架提供了复杂的循环、条件判断、依赖管理等功能，这有助于提高部署应用程序的灵活性以及配置和依赖的灵活性。它的实现为 kubernetes 控制器，它持续监控正在运行的应用程序并将当前的实时状态与所需的目标状态（如 Git 存储库中指定的）进行比较

Argo 中的工作流自动化是通过使用 ADSL（Argo 领域特定语言）设计的 YAML 模板（因为 Kubernetes 主要也使用相同的 DSL 方式，所以非常容易使用）进行驱动的。ADSL 中提供的每条指令都被看作一段代码，并与代码仓库中的源码一起托管。Argo 支持6中不同的 YAML 结构：

- 容器模板：根据需要创建单个容器和参数
- 工作流模板：定义一个作业任务（工作流中的一个步骤可以是一个容器）
- 策略模板：触发或者调用作业/通知的规则
- 部署模板：创建一个长期运行的应用程序模板
- Fixture 模板：整合 Argo 外部的第三方资源
- 项目模板：可以在 Argo 目录中访问的工作流定义

Argo 支持几种不同的方式来定义 Kubernetes 资源清单：[Ksonnect](https://ksonnet.io/)、[Helm Chart](http://helm.io/) 以及简单的 YAML/JSON 资源清单目录

# 安装

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

安装完成可以看到

```
[root@node000006 ~]# kubectl -n argocd get po
NAME                                 READY   STATUS    RESTARTS   AGE
argocd-application-controller-0      1/1     Running   0          10m
argocd-dex-server-745d7f54c4-xh8jb   1/1     Running   0          10m
argocd-redis-5f4d559f65-r4mkr        1/1     Running   0          10m
argocd-repo-server-8c957bfc-njgql    1/1     Running   0          10m
argocd-server-65d7bc9fc8-57m25       1/1     Running   0          10m
```

默认的安装是ClusterIP的，可以用LB或者Ingress来暴露

# 使用



