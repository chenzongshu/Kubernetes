> 基于1.19版本

# 组件

kube-apiserver 共由 3 个组件构成（Aggregator、KubeAPIServer、APIExtensionServer），这些组件依次通过 Delegation 处理请求：

- **Aggregator**：暴露的功能类似于一个七层负载均衡，将来自用户的请求拦截转发给其他服务器，并且负责整个 APIServer 的 Discovery 功能；
- **KubeAPIServer** ：负责对请求的一些通用处理，认证、鉴权等，以及处理各个内建资源的 REST 服务；
- **APIExtensionServer**：主要处理 CustomResourceDefinition（CRD）和 CustomResource（CR）的 REST 请求，也是 Delegation 的最后一环，如果对应 CR 不能被处理的话则会返回 404。

## Aggregator

Aggregator 通过 APIServices 对象关联到某个 Service 来进行请求的转发，其关联的 Service 类型进一步决定了请求转发形式。Aggregator 包括一个 `GenericAPIServer` 和维护自身状态的 Controller。其中 `GenericAPIServer` 主要处理 `apiregistration.k8s.io` 组下的 APIService 资源请求。

**Aggregator 除了处理资源请求外还包含几个 controller：**

- 1、`apiserviceRegistrationController`：负责 APIServices 中资源的注册与删除；
- 2、`availableConditionController`：维护 APIServices 的可用状态，包括其引用 Service 是否可用等；
- 3、`autoRegistrationController`：用于保持 API 中存在的一组特定的 APIServices；
- 4、`crdRegistrationController`：负责将 CRD GroupVersions 自动注册到 APIServices 中；
- 5、`openAPIAggregationController`：将 APIServices 资源的变化同步至提供的 OpenAPI 文档；

kubernetes 中的一些附加组件，比如 metrics-server 就是通过 Aggregator 的方式进行扩展的，实际环境中可以通过使用 [apiserver-builder](https://github.com/kubernetes-sigs/apiserver-builder-alpha) 工具轻松以 Aggregator 的扩展方式创建自定义资源。

### 启用 API Aggregation

在 kube-apiserver 中需要增加以下配置来开启 API Aggregation：

```
--proxy-client-cert-file=/etc/kubernetes/certs/proxy.crt
--proxy-client-key-file=/etc/kubernetes/certs/proxy.key
--requestheader-client-ca-file=/etc/kubernetes/certs/proxy-ca.crt
--requestheader-allowed-names=aggregator
--requestheader-extra-headers-prefix=X-Remote-Extra-
--requestheader-group-headers=X-Remote-Group
--requestheader-username-headers=X-Remote-User
```

## KubeAPIServer

KubeAPIServer 主要是提供对 API Resource 的操作请求，为 kubernetes 中众多 API 注册路由信息，暴露 RESTful API 并且对外提供 kubernetes service，使集群中以及集群外的服务都可以通过 RESTful API 操作 kubernetes 中的资源。

## APIExtensionServer

APIExtensionServer 作为 Delegation 链的最后一层，是处理所有用户通过 Custom Resource Definition 定义的资源服务器。

其中包含的 controller 以及功能如下所示：

- 1、`openapiController`：将 crd 资源的变化同步至提供的 OpenAPI 文档，可通过访问 `/openapi/v2` 进行查看；
- 2、`crdController`：负责将 crd 信息注册到 apiVersions 和 apiResources 中，两者的信息可通过 `$ kubectl api-versions` 和 `$ kubectl api-resources` 查看；
- 3、`namingController`：检查 crd obj 中是否有命名冲突，可在 crd `.status.conditions` 中查看；
- 4、`establishingController`：检查 crd 是否处于正常状态，可在 crd `.status.conditions` 中查看；
- 5、`nonStructuralSchemaController`：检查 crd obj 结构是否正常，可在 crd `.status.conditions` 中查看；
- 6、`apiApprovalController`：检查 crd 是否遵循 kubernetes API 声明策略，可在 crd `.status.conditions` 中查看；
- 7、`finalizingController`：类似于 finalizes 的功能，与 CRs 的删除有关；

# 启动

和别的一样，CORBA程序，位置是`cmd/kube-apiserver/apiserver.go`

```
┌─────────────────────┐                                         
│NewAPIServerCommand()│                                         
└──────────┬──────────┘                                         
           │                                                    
           ▼                                                    
       ┌──────┐                                                 
       │Run() │                                                 
       └───┬──┘                                                 
           │    ┌───────────────────┐                           
           ├───▶│CreateServerChain()│                           
           │    └───────────────────┘                           
           │                         ┌─────────────────────────┐
           │                         │createAPIExtensionsServer│
           │                         └─────────────────────────┘
           │                         ┌───────────────────┐      
           │                         │CreateKubeAPIServer│      
           │                         └───────────────────┘      
           │                         ┌──────────────────────┐   
           │                         │createAggregatorServer│   
           │                         └──────────────────────┘   
           │                                                    
           │    ┌───────────────────┐                           
           ├───▶│server.PrepareRun()│                           
           │    └───────────────────┘                           
           │    ┌───────────────┐                               
           └───▶│prepared.Run() │                               
                └───────────────┘                                 
```



