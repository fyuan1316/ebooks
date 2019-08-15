# 第2章 Kubernetes API基础知识

在本章中，我们将向您介绍Kubernetes API的基础知识。这包括深入了解API服务器的内部工作，API本身以及如何从命令行与API进行交互。我们将向您介绍Kubernetes API概念，例如资源和种类，以及分组和版本控制。

# API服务器

Kubernetes由一组具有不同角色的节点（集群中的机器）组成，[如图2-1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#k8s-arch-overview)所示：主节点上的控制平面由API服务器，控制器管理器和调度程序组成。API服务器是中央管理实体，也是与分布式存储组件直接对话的唯一组件`etcd`。

API服务器具有以下核心职责：

- 至服务于Kubernetes API。此API由主组件，工作节点和Kubernetes本机应用程序在内部集群使用，也可由客户端外部使用`kubectl`。
- 代理群集组件（例如Kubernetes仪表板），或流式传输日志，服务端口或服务`kubectl exec`会话。

提供API意味着：

- 读取状态：获取单个对象，列出它们以及流式更改
- 操作状态：创建，更新和删除对象

国家坚持通过`etcd`。

![Kubernetes建筑概述](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0201.png)

###### 图2-1。Kubernetes建筑概述

Kubernetes的核心是它的API服务器。但是API服务器如何工作？我们首先将API服务器视为黑盒子并仔细查看其HTTP接口，然后我们将继续讨论API服务器的内部工作原理。

## API服务器的HTTP接口

从从客户的角度来看，API服务器公开了一个RESTful HTTP API，其中包含JSON或[*协议缓冲区*](http://bit.ly/1HhFC5L)（简称*protobuf*）有效负载，主要用于集群内部通信，出于性能原因。

API服务器HTTP接口使用以下[HTTP谓词](https://mzl.la/2WX21hL)（或HTTP方法）处理HTTP请求以查询和操作Kubernetes资源：

- HTTP `GET`谓词用于检索具有特定资源（例如某个窗格）或资源集合或列表（例如，命名空间中的所有窗格）的数据。
- HTTP `POST`谓词用于创建资源，例如服务或部署。
- HTTP `PUT`谓词用于更新现有资源 - 例如，更改窗格的容器图像。
- HTTP `PATCH`谓词用于现有资源的部分更新。阅读Kubernetes文档中的[“使用JSON合并补丁更新部署”](http://bit.ly/2Xpbi6I)，以了解有关可用策略和含义的更多信息。
- HTTP `DELETE`动词用于以不可恢复的方式销毁资源。

如果你看一下Kubernetes [1.14 API参考](http://bit.ly/2IVevBG)，你可以看到不同的HTTP动词。例如，要使用等效的CLI命令列出当前命名空间中的pod `kubectl -n` `*THENAMESPACE*` `get pods`，您将发出（参见[图2-2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#api-server-list-pods)）。`GET /api/v1/namespaces/*THENAMESPACE*/pods`

![运行中的API服务器HTTP接口：列出给定命名空间中的pod](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0202.png)

###### 图2-2。运行中的API服务器HTTP接口：列出给定命名空间中的pod

有关如何从Go程序调用API服务器HTTP接口的介绍，请参阅[“客户端库”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#client-go)。

## API术语

之前 我们进入API业务，让我们首先定义Kubernetes API服务器上下文中使用的术语：

- 类

  该实体的类型。每个对象都有一个字段`Kind`（`kind`JSON中的小写，`Kind`在Golang中大写），它告诉客户端，例如`kubectl`它代表一个pod。那里 有三类：对象表示系统中*的持久实体* - 例如，`Pod`或`Endpoints`。对象具有名称，其中许多都存在于名称空间中。列表是一种或多种实体的集合。列表具有有限的公共元数据集。例子包括`PodList`s或`NodeList`s。当你做一个**kubectl get pods**，这正是你得到的。专用类用于对象和非持久实体（如`/binding`或）的特定操作`/scale`。对于发现，Kubernetes使用`APIGroup`和`APIResource`; 对于错误结果，它使用`Status`。

在Kubernetes程序，直接与Golang类型对应。因此，作为Golang类型，种类是单数的，并以大写字母开头。

- API组

  一个`Kind`与逻辑相关的s的集合。例如，所有批次的对象像`Job`或`ScheduledJob`是批处理API组中使用。

- 版

  每API组可以存在多个版本，其中大多数都可以。例如，一个小组首先出现，`v1alpha1`然后晋升为`v1beta1`最终毕业`v1`。`v1beta1`可以在每个支持的版本中检索在一个版本（例如）中创建的对象。API服务器无损转换为返回请求版本中的对象。从集群用户的角度来看，版本只是相同对象的不同表示。

###### 小费

没有“ `v1`集群中有一个对象，集群中有另一个对象`v1beta1`。”而是，每个对象都可以作为`v1`表示或`v1beta1`表示返回，就像集群用户所希望的那样。

- 资源

  一个通常是小写的，复数的单词（例如，`pods`）标识一组HTTP端点（路径），其暴露系统中某个对象类型的CRUD（创建，读取，更新，删除）语义。常见路径是：根目录，例如*... / pods*，列出该类型的所有实例各个命名资源的路径，例如*... / pods / nginx*通常，这些端点中的每一个都返回并接收一种类型（`PodList`在第一种情况下为a `Pod`，在第二种情况下为a ）。但在其他情况下（例如，在出现错误的情况下），`Status`会返回一个类型的对象。除了具有完整CRUD语义的主资源之外，资源还可以有其他端点来执行特定操作（例如，*... / pod / nginx / port-forward*，*... / pod / nginx / exec*，或*... / pod / nginx / logs*）。我们调用这些*子资源*（参见[“子资源”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#crd-subresources)）。这些通常实现自定义协议而不是REST - 例如，通过WebSockets或命令式API进行某种流式连接。

###### 小费

资源和种类经常混在一起。请注意明确的区别：

- 资源对应于HTTP路径。
- 种类是这些端点返回和接收的对象类型，以及持久化的对象类型`etcd`。

资源始终是API组和版本的一部分，统称为*GroupVersionResource*（或GVR）。GVR唯一地定义HTTP路径。例如，`default`命名空间中的具体路径是*/ apis / batch / v1 / namespaces / default / jobs*。[图2-3](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#gvr)显示了命名空间资源的示例GVR，a `Job`。

![Kubernetes API  - 集团，版本，资源（GVR）](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0203.png)

###### 图2-3。Kubernetes API - GroupVersionResource（GVR）

与`jobs`GVR示例相反，群集范围的资源（如节点或命名空间）本身在路径中没有*$ NAMESPACE*部分。例如，`nodes`GVR示例可能如下所示：*/ api / v1 / nodes*。请注意，名称空间显示在其他资源的HTTP路径中，但也是资源本身，可在*/ api / v1 / namespaces访问*。

同样对于GVR，每种类型都存在于API组中，具有版本，并通过*GroupVersionKind*（GVK）进行识别。

##### 同居 - 生活在多个API组中的种类

种同名的，不仅可以在不同的*版本中*共存，也可以在不同的API组中共存。例如，`Deployment`在扩展组中以alpha类型开始，最终在其自己的组中升级为稳定版本`apps.k8s.io`。我们称之为*同居*。虽然在Kubernetes中不常见，但有一些：

- `Ingress`，`NetworkPolicy`在`extensions`和`networking.k8s.io`
- `Deployment`，`DaemonSet`，`ReplicaSet`在`extensions`和`apps`
- `Event` 在核心小组和 `events.k8s.io`

GVK和GVR是相关的。GVK在GVR标识的HTTP路径下提供。调用将GVK映射到GVR的过程REST映射。我们将`RESTMappers`在[“REST Mapping”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#RESTMapping)看到在Golang中实现REST映射。

从从全局的角度来看，API资源空间在逻辑上形成了一个具有顶级节点的树，包括*/ api*，*/ apis*和一些非等级端点，例如*/ healthz*或*/ metrics*。此API空间的示例呈现[如图2-4](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#api-space-tree)所示。请注意，确切的形状和路径取决于Kubernetes版本，多年来趋于稳定。

![一个示例Kubernetes API空间](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0204.png)

###### 图2-4。一个示例Kubernetes API空间

## Kubernetes API版本控制

对于可扩展性原因，Kubernetes支持不同API路径下的多个API版本，例如*/ api / v1*或*/ apis / extensions / v1beta1*。不同的API版本意味着不同级别的稳定性和支持：

- `v1alpha1`通常在默认情况下禁用*Alpha*级别（例如）; 可以随时删除对功能的支持，恕不另行通知，并且只能在短期测试群集中使用。
- `v2beta3`默认情况下启用*Beta*级别（例如），这意味着代码已经过充分测试; 但是，对象的语义可能会在随后的beta或稳定版本中以不兼容的方式发生变化。
- `v1`对于许多后续版本，*稳定*（通常可用或GA）级别（例如）将出现在已发布的软件中。

让我们来看看如何构造HTTP API空间：在顶层我们区分核心组 - 即*/ api / v1*下面的所有内容- 以及*/ apis / $ NAME/ $*形式的路径中的命名组`*VERSION*`。

###### 注意

由于历史原因，核心小组位于*/ apis / core / v1*下，`/api/v1`而不是人们所期望的。在引入API组的概念之前，核心组已存在。

API服务器公开了第三种类型的HTTP路径 - 非资源对齐的路径：群集范围的实体，例如*/ metrics*，*/ logs*或*/ healthz*。此外，API服务器支持手表; 也就是说，您可以添加`?watch=true`特定请求，并将API服务器更改为[监视模式](http://bit.ly/2x5PnTl)，而不是按设定的时间间隔轮询资源。

## 声明性国家管理

最API对象区分了*所需*资源*状态*的规范和当前时间对象的*状态*。甲*规范*，或规范的简称，是一种资源的期望状态的完整描述，并且通常坚持稳定存储，通常`etcd`。

###### 注意

为什么我们说“通常`etcd`”？好吧，有Kubernetes发行版和产品，如[k3s](https://k3s.io/)或微软的AKS，已经取代或正在努力替换`etcd`其他东西。由于Kubernetes控制平面的模块化架构，这很好用。

让我们在API服务器的上下文中更多地讨论规范（期望状态）与状态（观察状态）。

规范描述了您所需的资源状态，您需要通过命令行工具提供，例如`kubectl`通过Go代码以编程方式提供。状态描述资源的观察或实际状态，并由控制平面管理，可由核心组件（如控制器管理器）或您自己的自定义控制器管理（请参阅[“控制器和操作员”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#ch_controllers-operators)）。例如，在部署中，您可以指定您希望始终运行20个应用程序副本。部署控制器是控制平面中控制器管理器的一部分，它读取您提供的部署规范并创建一个副本集，然后负责管理副本：它创建相应数量的pod，最终（通过`kubelet`）导致容器在工作节点上启动。如果任何副本失败，部署控制器将使您知道状态。这就是我们所谓的*声明式状态管理* - 也就是说，声明所需的状态并让Kubernetes处理剩下的事情。

当我们开始从命令行开始探索API时，我们将在下一节中看到声明式状态管理。

# 使用命令行中的API

在本节我们将使用`kubectl`并`curl`演示Kubernetes API的使用。如果您不熟悉这些CLI工具，现在是安装它们并试用它们的好时机。

首先，让我们看一下资源的期望和观察状态。我们将`kube-dns`在`kube-system`命名空间中使用可能在每个集群中可用的控制平面组件，CoreDNS插件（旧的Kubernetes版本正在使用）（此输出经过大量编辑以突出显示重要部分）：

```
$ kubectl -n kube-system get deploy / coredns -o =yaml
apiVersion：apps / v1
kind：部署
元数据：
  名称：coredns
  命名空间：kube-system
  ...
规格：
  模板：
    规格：
      容器：
      - 名称：coredns
        图片：602401143452.dkr.ecr.us-east-2.amazonaws.com/eks/coredns:v1.2.2
  ...
状态：
  复制品：2
  条件：
  - type：可用
    status："True"
    lastUpdateTime："2019-04-01T16:42:10Z"
  ...
```

正如您可以从此`kubectl`命令中看到的那样，在`spec`部署的部分中，您将定义一些特征，例如要使用的容器映像以及要并行运行的副本数，并在本`status`节中学习了多少个副本。当前时间点实际上正在运行。

为了执行与CLI相关的操作，在本章的其余部分中，我们将使用批处理操作作为运行示例。让我们首先在终端中执行以下命令：

```
$ kubectl proxy --port =8080
开始服务于127.0.0.1:8080
```

此命令将Kubernetes API代理到本地计算机，并且还负责身份验证和授权位。它允许我们通过HTTP直接发出请求并接收JSON有效负载。让我们通过启动我们查询的第二个终端会话来做到这一点`v1`：

```
$ 卷曲http://127.0.0.1:8080/apis/batch/v1
 {
  "kind"："APIResourceList"，
   "apiVersion"："v1"，
   "groupVersion"："batch/v1"，
   "resources"：[
    {
      "name"："jobs"，
       "singularName"：""，
       "namespaced"：true，
       "kind"："Job"，
       "verbs"：[
        "create"，
         "delete"，
         "deletecollection"，
         "get"，
         "list"，
         "patch"，
         "update"，
         "watch"
      ]，
       "categories"：[
        "all"
      ]
    }，
     {
      "name"："jobs/status"，
       "singularName"：""，
       "namespaced"：true，
       "kind"："Job"，
       "verbs"：[
        "get"，
         "patch"，
        "update"
      ]
    }
  ]
}
```

###### 小费

You不必`curl`与`kubectl proxy`命令一起使用即可获得对Kubernetes API的直接HTTP API访问。您可以改为使用`kubectl get --raw`命令：例如，替换`curl http://127.0.0.1:8080/apis/batch/v1`为`kubectl get --raw /apis/batch/v1`。

将其与`v1beta1`版本进行比较，注意到在查看*http://127.0.0.1:8080/apis/batch* 时，您可以获得批处理API组的受支持版本列表`v1beta1`：

```
$ 卷曲http://127.0.0.1:8080/apis/batch/v1beta1
 {
  "kind"："APIResourceList"，
   "apiVersion"："v1"，
   "groupVersion"："batch/v1beta1"，
   "resources"：[
    {
      "name"："cronjobs"，
       "singularName"：""，
       "namespaced"：true，
       "kind"："CronJob"，
       "verbs"：[
        "create"，
         "delete"，
         "deletecollection"，
         "get"，
         "list"，
         "patch"，
         "update"，
         "watch"
      ]，
       "shortNames"：[
        "cj"
      ]，
       "categories"：[
        "all"
      ]
    }，
     {
      "name"："cronjobs/status"，
       "singularName"：""，
       "namespaced"：true，
       "kind"："CronJob"，
       "verbs"：[
        "get"，
         "patch"，
        "update"
      ]
    }
  ]
}
```

如您所见，该`v1beta1`版本还包含`cronjobs`具有该类型的资源`CronJob`。在撰写本文时，cron工作尚未晋升`v1`。

如果您想了解群集中支持哪些API资源，包括它们的类型，是否为命名空间，以及它们的短名称（主要用于`kubectl`命令行），您可以使用以下命令：

```
$ kubectl api-resources
NAME SHORTNAMES APIGROUP NAMESPACED KIND
绑定                                    true         绑定
componentstatuses cs                   false        ComponentStatus
configmaps cm                   true         ConfigMap
端点ep                   true         端点
事件ev                   true         事件
limitranges限制               true         LimitRange
名称空间ns                   false        命名空间
节点没有                   false        节点
persistentvolumeclaims pvc                  true         PersistentVolumeClaim
persistentvolumes pv                   false        PersistentVolume
pods po                   true         pod
子                                true         模板PodTemplate
replicationcontrollers rc                   true         ReplicationController
resourcequotas配额                true         ResourceQuota
秘密                                     true         的秘密
serviceaccounts sa                   true         ServiceAccount
服务svc                  true         服务
controllerrevisions应用程序      true         ControllerRevision
daemonsets ds apps      true         DaemonSet
部署部署应用程序      true         部署
...
```

该 以下是一个相关命令，对于确定群集中支持的不同资源版本非常有用：

```
$ kubectl api-versions
admissionregistration.k8s.io/v1beta1
apiextensions.k8s.io/v1beta1
apiregistration.k8s.io/v1
apiregistration.k8s.io/v1beta1
appmesh.k8s.aws/v1alpha1
appmesh.k8s.aws/v1beta1
应用程序/ V1
应用程序/ v1beta1
应用程序/ v1beta2
authentication.k8s.io/v1
authentication.k8s.io/v1beta1
authorization.k8s.io/v1
authorization.k8s.io/v1beta1
自动缩放/ V1
自动缩放/ v2beta1
自动缩放/ v2beta2
批/ V1
批/ v1beta1
certificates.k8s.io/v1beta1
coordination.k8s.io/v1beta1
crd.k8s.amazonaws.com/v1alpha1
events.k8s.io/v1beta1
扩展/ v1beta1
networking.k8s.io/v1
政策/ v1beta1
rbac.authorization.k8s.io/v1
rbac.authorization.k8s.io/v1beta1
scheduling.k8s.io/v1beta1
storage.k8s.io/v1
storage.k8s.io/v1beta1
V1
```

# API服务器如何处理请求

现在如果您了解面向外部的HTTP接口，那么我们将重点关注API服务器的内部工作原理。[图2-5](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#api-server-high-level-flow)显示了API服务器中请求处理的高级概述。

![Kubernetes API服务器请求处理概述](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0205.png)

###### 图2-5。Kubernetes API服务器请求处理概述

所以呢实际上当HTTP请求到达Kubernetes API时会发生什么？在较高的层面上，发生以下互动：

1. HTTP请求由注册的过滤器链处理`DefaultBuildHandlerChain()`。该链在[*k8s.io/apiserver/pkg/server/config.go中*](http://bit.ly/2x9t27e)定义，并在[*稍后*](http://bit.ly/2x9t27e)详细讨论。它对所述请求应用一系列过滤操作。任一个滤波器通过并附着各信息添加到上下文准确地来说`ctx.RequestInfo`，与`ctx`作为[上下文](https://golang.org/pkg/context)在Go（例如，通过认证的用户） -或，如果请求不通过过滤器，它返回一个适当的HTTP响应代码说明原因（例如，用户身份验证失败时的[`401`响应](https://httpstatuses.com/401)）。
2. 接下来，根据HTTP路径，[*k8s.io/apiserver/pkg/server/handler.go中*](http://bit.ly/2WUd0c6)的多路复用器将HTTP请求路由到相应的处理程序。
3. 为每个API组注册了一个处理程序 - 有关详细信息，请参阅[*k8s.io/apiserver/pkg/endpoints/groupversion.go*](http://bit.ly/2IvvSKA)和[*k8s.io/apiserver/pkg/endpoints/installer.go*](http://bit.ly/2Y1eySV)。它接受HTTP请求以及上下文（例如，用户和访问权限）并检索并从`etcd`存储中传递请求的对象。

现在让我们仔细看看[*server / config.go中*](http://bit.ly/2LWUUnQ)设置的过滤器链，以及每个过滤器`DefaultBuildHandlerChain()`中发生的事情：

```
func DefaultBuildHandlerChain(apiHandler http.Handler, c *Config) http.Handler {
    h := WithAuthorization(apiHandler, c.Authorization.Authorizer, c.Serializer)
    h = WithMaxInFlightLimit(h, c.MaxRequestsInFlight,
          c.MaxMutatingRequestsInFlight, c.LongRunningFunc)
    h = WithImpersonation(h, c.Authorization.Authorizer, c.Serializer)
    h = WithAudit(h, c.AuditBackend, c.AuditPolicyChecker, LongRunningFunc)
    ...
    h = WithAuthentication(h, c.Authentication.Authenticator, failed, ...)
    h = WithCORS(h, c.CorsAllowedOriginList, nil, nil, nil, "true")
    h = WithTimeoutForNonLongRunningRequests(h, LongRunningFunc, RequestTimeout)
    h = WithWaitGroup(h, c.LongRunningFunc, c.HandlerChainWaitGroup)
    h = WithRequestInfo(h, c.RequestInfoResolver)
    h = WithPanicRecovery(h)
    return h
}
```

所有包都在[*k8s.io/apiserver/pkg中*](http://bit.ly/2LUzTdx)。更具体地审查：

- `WithPanicRecovery()`

  处理恢复和记录恐慌。在[*server / filters / wrap.go中*](http://bit.ly/2N0zfNB)定义。

- `WithRequestInfo()`

  附上一个`RequestInfo`上下文。在[*端点/过滤器/ requestinfo.go中*](http://bit.ly/2KvKjQH)定义。

- `WithWaitGroup()`

  将所有非长时间运行的请求添加到等待组; 用于优雅关机。在[*server / filters / waitgroup.go中*](http://bit.ly/2ItnsD6)定义。

- `WithTimeoutForNonLongRunningRequests()`

  超时非长期运行的请求（最喜欢的`GET`，`PUT`，`POST`，和`DELETE`请求），而相比之下，长期运行的请求，如手表和代理请求。在[*server / filters / timeout.go中*](http://bit.ly/2KrKk8r)定义。

- `WithCORS()`

  提供[CORS](https://enable-cors.org/)实现。CORS是跨源资源共享的缩写，是一种允许嵌入HTML页面的JavaScript将XMLHttpRequests发送到与JavaScript发起的域不同的域的机制。在[*server / filters / cors.go中*](http://bit.ly/2L2A6uJ)定义。

- `WithAuthentication()`

  尝试将给定请求作为人员或机器用户进行身份验证，并将用户信息存储在提供的上下文中。成功时，`Authorization`将从请求中删除HTTP标头。如果身份验证失败，则返回HTTP `401`状态代码。在[*端点/过滤器/ authentication.go中*](http://bit.ly/2Fjzr4b)定义。

- `WithAudit()`

  使用所有传入请求的审核日志记录信息来装饰处理程序。审计日志条目包含诸如请求的源IP，用户调用操作和请求的命名空间之类的信息。在[*录取/审计中*](http://bit.ly/2XpQN9U)定义。

- `WithImpersonation()`

  通过检查尝试更改用户的请求（类似于`sudo`）来处理用户模拟。在[*端点/过滤器/ impersonation.go中*](http://bit.ly/2L2UETP)定义。

- `WithMaxInFlightLimit()`

  限制飞行中的请求数量。在[*server / filters / maxinflight.go中*](http://bit.ly/2IY4unl)定义。

- `WithAuthorization()`

  通过调用授权模块检查权限，并将所有授权请求传递给多路复用器，多路复用器将请求分派给正确的处理程序。如果用户没有足够的权限，则返回HTTP `403`状态代码。Kubernetes现在使用基于角色的访问控制（RBAC）。在[*端点/过滤器/ authorization.go中*](http://bit.ly/31M2NSA)定义。

传递此通用处理程序链后（[图2-5中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#api-server-high-level-flow)的第一个框），实际的请求处理开始（即执行请求处理程序的语义）：

- 直接处理对*/*，*/ version*，*/ apis*，*/ healthz*和其他nonRESTful API的请求。

- RESTful资源的请求进入请求管道，包括：

  - *入场*

    传入的对象通过准入链。这个链条有20种不同录取插件。[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#idm46336866991944)每个插件都可以是变异阶段的一部分（参见[图2-5中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#api-server-high-level-flow)的第三个框），验证阶段的一部分（参见图中的第四个框），或两者兼而有之。在变异阶段，可以改变传入的请求有效载荷; 例如，图像拉策略被设置为`Always`，`IfNotPresent`，或`Never`取决于入场配置。第二个入场阶段纯粹是为了验证; 例如，验证pod中的安全设置，或者在创建该命名空间中的对象之前验证是否存在命名空间。

  - *验证*

    根据大型验证逻辑检查传入对象，该逻辑对系统中的每个对象类型都存在。例如，检查字符串格式以验证服务名称中仅使用有效的DNS兼容字符，或者pod中的所有容器名称是唯一的。

  - `etcd`- *支持CRUD逻辑*

    这里实现了我们在[“API服务器的HTTP接口”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#api-server-http-interface)看到的不同动词; 例如，更新逻辑从中读取对象`etcd`，检查没有其他用户在[“乐观并发”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#optimistic-concurrency)意义上修改了对象，如果没有，则将请求对象写入`etcd`。

我们将在以下章节中更详细地研究所有这些步骤; 对于例如：

- 自定义资源

  验证在[“验证自定义资源”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#crd-validation)，在录取[“招生网络挂接”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#admission-webhooks)中，和一般的CRUD语义[第4章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#ch_crds)

- Golang原生资源

  [“验证”中的](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-validation)验证，[“](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-validation)录取[”中的录取](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-admission)以及[“注册表和策略”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-registry)中CRUD语义的实现

# 摘要

在本章中，我们首先将Kubernetes API服务器作为黑盒子进行了讨论，并查看了其HTTP接口。然后你学习了如何在命令行上与那个黑盒子进行交互，最后我们打开了黑盒子并探索了它的内部工作原理。到目前为止，您应该知道API服务器如何在内部工作，以及如何使用CLI工具`kubectl`与资源进行交互以进行资源探索和操作。

现在是时候将手动交互留在我们身后的命令行，并开始使用Go：meet `client-go`（Kubernetes“标准库”的核心）进行编程API服务器访问。

[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#idm46336866991944-marker)在一个Kubernetes 1.14簇，这些是（以该顺序）： ，`AlwaysAdmit`，`NamespaceAutoProvision`，，，，，，，，，，，，，，，，，，，，，，，，，，，，和。`NamespaceLifecycle``NamespaceExists``SecurityContextDeny``LimitPodHardAntiAffinityTopology``PodPreset``LimitRanger``ServiceAccount``NodeRestriction``TaintNodesByCondition``AlwaysPullImages``ImagePolicyWebhook``PodSecurityPolicy``PodNodeSelector``Priority``DefaultTolerationSeconds``PodTolerationRestriction``DenyEscalatingExec``DenyExecOnPrivileged``EventRateLimit``ExtendedResourceToleration``PersistentVolumeLabel``DefaultStorageClass``StorageObjectInUseProtection``OwnerReferencesPermissionEnforcement``PersistentVolumeClaimResize``MutatingAdmissionWebhook``ValidatingAdmissionWebhook``ResourceQuota``AlwaysDeny`