# 第4章使用自定义资源

在 本章我们向您介绍自定义资源（CR），这是整个Kubernetes生态系统中使用的中心扩展机制之一。

自定义资源用于小型的内部配置对象，没有任何相应的控制器逻辑 - 纯粹以声明方式定义。但是，对于希望提供Kubernetes原生API体验的Kubernetes之上的许多重要开发项目，自定义资源也发挥着核心作用。示例是服务网格，例如Istio，Linkerd 2.0和AWS App Mesh，它们都有自定义资源。

还记得[第1章中的](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#intro) “动机例子” 吗？它的核心是它有一个如下所示的CR：

```
apiVersion: cnat.programming-kubernetes.info/v1alpha1
kind: At
metadata:
  name: example-at
spec:
  schedule: "2019-07-03T02:00:00Z"
status:
  phase: "pending"
```

习惯自1.7版以来，每个Kubernetes集群都提供了资源。它们存储在与`etcd`主Kubernetes API资源相同的实例中，并由相同的Kubernetes API服务器提供服务。[如图4-1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#apiextensions-apiserver)所示，请求回退到为`apiextensions-apiserver`资源提供服务的请求 通过CRD定义，如果它们不是以下两者：

- 由聚合的API服务器处理（参见[第8章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#ch_custom-api-servers)）。
- 本地Kubernetes资源。

![Kubernetes API服务器内部的API Extensions服务器API](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0401.png)

###### 图4-1。Kubernetes API服务器内的API Extensions API服务器

CustomResourceDefinition（CRD）本身就是Kubernetes资源。它描述了群集中的可用CR。对于前面的示例CR，相应的CRD如下所示：

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ats.cnat.programming-kubernetes.info
spec:
  group: cnat.programming-kubernetes.info
  names:
    kind: At
    listKind: AtList
    plural: ats
    singular: at
  scope: Namespaced
  subresources:
    status: {}
  version: v1alpha1
  versions:
  - name: v1alpha1
    served: true
    storage: true
```

在这种情况下，CRD的名称 - `ats.cnat.programming-kubernetes.info`必须匹配复数名称，后跟组名称。它将`At`API组中的种类CR 定义`cnat.programming-kubernetes.info`为名为的命名空间资源`ats`。

如果在群集中创建此CRD，`kubectl`将自动检测资源，用户可以通过以下方式访问它：

```
$ kubectl得到了
姓名创建于
ats.cnat.programming-kubernetes.info 2019-04-01T14：03：33Z
```

# 发现信息

背后场景，`kubectl`使用来自API服务器的发现信息来了解新资源。让我们看一下这个发现机制。

在增加详细级别后`kubectl`，我们实际上可以看到它如何了解新的资源类型：

```
$ kubectl得到ats -v =7
... GET https://XXX.eks.amazonaws.com/apis/cnat.programming-kubernetes.info/
                                      v1alpha1 / namespaces / cnat / ats？limit =500
...请求标题：
...接受：application / json ;as=表;v=v1beta1 ;g=meta.k8s.io，application / json
      User-Agent：kubectl / v1.14.0 (darwin / amd64 )kubernetes / 641856d
...响应状态：200以607毫秒为单位确定
姓名年龄
例子 - 在43s
```

详细的发现步骤是：

1. 最初，`kubectl`不知道`ats`。
2. 因此，`kubectl`通过*/ apis*发现端点向API服务器询问所有现有API组。
3. 接下来，`kubectl`通过*/ apis /group version* group发现端点向API服务器询问所有现有API组中的资源。
4. 然后，`kubectl`将给定类型转换`ats`为以下三倍：
   - 集团（这里`cnat.programming-kubernetes.info`）
   - 版本（这里`v1alpha1`）
   - 资源（这里`ats`）。

发现端点提供了在最后一步执行转换所需的所有信息：

```
$ http localhost：8080 / apis /
{
  “群组”：[{
    “name”：“at.cnat.programming-kubernetes.info”，
    “preferredVersion”：{
      “groupVersion”：“cnat.programming-kubernetes.info/v1”，
      “版本”：“v1alpha1”
    },
    “版本”：[{
      “groupVersion”：“cnat.programming-kubernetes.info/v1alpha1”，
      “版本”：“v1alpha1”
    }]
  }, ...]
}

$ http localhost：8080 / apis / cnat.programming-kubernetes.info / v1alpha1
{
  “apiVersion”：“v1”，
  “groupVersion”：“cnat.programming-kubernetes.info/v1alpha1”，
  “kind”：“APIResourceList”，
  “资源”：[{
    “善良”：“在”，
    “名字”：“ats”，
    “namespaced”：是的，
    “动词”：[“创建”，“删除”，“删除集合”，
      “get”，“list”，“patch”，“update”，“watch”
    ]
  }, ...]
}
```

这一切都是由发现实现的`RESTMapper`。我们也看到了这个非常常见的类型`RESTMapper`在[“REST映射”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#RESTMapping)。

###### 警告

该`kubectl`CLI还保持资源类型的高速缓存中的*〜/ .kubectl*，使其不必再检索每次访问发现信息。此缓存每10分钟无效。因此，CRD的更改可能会在相应用户的CLI中显示，最多10分钟后。

# 类型定义

现在让我们更详细地看一下CRD和提供的功能：如`cnat`示例中所示，CRD是Kubernetes API服务器进程内部`apiextensions.k8s.io/v1beta1`提供的API组中的Kubernetes资源`apiextensions-apiserver`。

CRD的架构如下所示：

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: name
spec:
  group: group name
  version: version name
  names:
    kind: uppercase name
    plural: lowercase plural name
    singular: lowercase singular name # defaulted to be lowercase kind
    shortNames: list of strings as short names # optional
    listKind: uppercase list kind # defaulted to be kindList
    categories: list of category membership like "all" # optional
  validation: # optional
    openAPIV3Schema: OpenAPI schema # optional
  subresources: # optional
    status: {} # to enable the status subresource (optional)
    scale: # optional
      specReplicasPath: JSON path for the replica number in the spec of the
                        custom resource
      statusReplicasPath: JSON path for the replica number in the status of
                          the custom resource
      labelSelectorPath: JSON path of the Scale.Status.Selector field in the
                         scale resource
  versions: # defaulted to the Spec.Version field
  - name: version name
    served: boolean whether the version is served by the API server # defaults to false
    storage: boolean whether this version is the version used to store object
  - ...
```

许多字段是可选的或默认的。我们将在以下部分中更详细地解释这些字段。

后创建一个CRD对象，`apiextensions-apiserver`内部`kube-apiserver`将检查名称并确定它们是否与其他资源冲突或者它们本身是否一致。片刻之后，它会将结果报告给CRD的状态，例如：

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ats.cnat.programming-kubernetes.info
spec:
  group: cnat.programming-kubernetes.info
  names:
    kind: At
    listKind: AtList
    plural: ats
    singular: at
  scope: Namespaced
  subresources:
    status: {}
  validation:
    openAPIV3Schema:
      type: object
      properties:
        apiVersion:
          type: string
        kind:
          type: string
        metadata:
          type: object
        spec:
          properties:
            schedule:
              type: string
          type: object
        status:
          type: object
  version: v1alpha1
  versions:
  - name: v1alpha1
    served: true
    storage: true
status:
    acceptedNames:
      kind: At
      listKind: AtList
      plural: ats
      singular: at
    conditions:
    - lastTransitionTime: "2019-03-17T09:44:21Z"
      message: no conflicts found
      reason: NoConflicts
      status: "True"
      type: NamesAccepted
    - lastTransitionTime: null
      message: the initial names have been accepted
      reason: InitialNamesAccepted
      status: "True"
      type: Established
    storedVersions:
    - v1alpha1
```

您可以看到规范中缺少的名称字段是默认的，并在状态中反映为可接受的名称。此外，还设置了以下条件：

- `NamesAccepted` 描述规范中给定的名称是否一致且没有冲突。
- `Established`描述API服务器在名称下提供给定资源`status.acceptedNames`。

请注意，在创建CRD后很长时间内可以更改某些字段。例如，您可以添加短名称或列。在这种情况下，可以建立CRD，即使用旧名称提供 - 尽管规范名称存在冲突。因此，`NamesAccepted`条件将是错误的，规范名称和接受的名称将不同。

# 自定义资源的高级功能

在本节中，我们将讨论自定义资源的高级功能，例如验证或子资源。

## 验证自定义资源

CR的可以在创建和更新期间由API服务器进行验证。这是基于CRD规范中的字段中指定的[OpenAPI v3架构](http://bit.ly/2RqtN5i)完成的`validation`。

当请求创建或改变CR时，规范中的JSON对象将根据此规范进行验证，如果出现错误，则会在HTTP代码`400`响应中将冲突字段返回给用户。[图4-2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#apiextensions-apiserver-validation)显示了在...内的请求处理程序中进行验证的位置`apiextensions-apiserver`。

可以在验证准入webhooks中实现更复杂的验证 - 即，使用图灵完整的编程语言。[图4-2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#apiextensions-apiserver-validation)显示了在本节中描述的基于OpenAPI的验证之后直接调用这些webhook。在[“Admission Webhooks”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#admission-webhooks)，我们将看到如何实施和部署入场webhook。在那里，我们将研究将其他资源考虑在内的验证，因此远远超出OpenAPI v3验证。幸运的是，对于许多用例，OpenAPI v3模式就足够了。

![验证步骤在`apiextensions-apiserver`的处理程序堆栈中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0402.png)

###### 图4-2。apiextensions-apiserver的处理程序堆栈中的验证步骤

该OpenAPI模式语言基于[JSON Schema标准](http://bit.ly/2J7aIT7)，该[标准](http://bit.ly/2J7aIT7)使用JSON / YAML本身来表示模式。这是一个例子：

```
type: object
properties:
  apiVersion:
    type: string
  kind:
    type: string
  metadata:
    type: object
  spec:
    type: object
    properties:
      schedule:
        type: string
        pattern: "^\d{4}-([0]\d|1[0-2])-([0-2]\d|3[01])..."
      command:
        type: string
    required:
    - schedule
    - command
  status:
    type: object
    properties:
      phase:
        type: string
required:
- metadata
- apiVersion
- kind
- spec
```

此模式指定该值实际上是JSON对象; [1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#idm46336863073336)即，它是一个字符串映射，而不是一个切片或一个像数字的值。此外，它具有（除了`metadata`，`kind`，和`apiVersion`，其中隐式地定制资源定义的）两个附加属性：`spec`和`status`。

每个都是JSON对象。`spec`有需要的领域`schedule`和`command`，这两者都是字符串。`schedule`必须匹配ISO日期的模式（在这里用一些正则表达式勾画）。optional `status`属性有一个名为的字符串字段`phase`。

##### OPENAPI V3架构，完整性及其未来

OpenAPI v3模式曾经是CRD中的可选模式。在Kubernetes 1.14之前，它们仅用于服务器端验证。为此目的，它们也可能是不完整的 - 换句话说，它们可能没有指定所有字段。

从Kubernetes 1.15开始，CRD模式将作为Kubernetes API服务器OpenAPI规范的一部分发布。这个特别`kubectl`用于客户端验证。客户端验证会抱怨未知字段。例如，当用户键入`foo:bar`对象并且OpenAPI架构未指定时`foo`，`kubectl`将拒绝该对象。因此，优良作法是传递完整的OpenAPI模式。

最后，[将来将修剪自定义资源实例](http://bit.ly/2WY8lKY)。这意味着 - 类似于本地Kubernetes资源类型的pod-未知（未指定）字段将不会被持久化。这不仅对数据一致性很重要，而且对安全性也很重要。这是CRD的OpenAPI模式应该完整的另一个原因。

有关完整参考，请参阅[OpenAPI v3架构文档](http://bit.ly/2RqtN5i)。

手动创建OpenAPI模式可能很乏味。幸运的是，正在进行的工作是通过代码生成使这更容易：Kubebuilder项目 - 参见[“Kubebuilder”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch06.html#kubebuilder) - 已[`crd-gen`在*sig.k8s.io/controller-tools中*](http://bit.ly/2J00kvi)开发，并且这是逐步扩展的，以便它可以在其他地方使用上下文。发电机[`crd-schema-gen`](http://bit.ly/31N0eQf)是`crd-gen`这个方向的叉子。

## 短名称和类别

喜欢本机资源，自定义资源可能具有长资源名称。它们在API级别上很棒，但在CLI中输入很繁琐。CR也可以有短名称，就像`daemonsets`可以查询的本机资源一样`kubectl get ds`。这些短名称也称为别名，每个资源可以包含任意数量的别名。

至查看所有可用的短名称，使用如下`kubectl api-resources`命令：

```
$ kubectl api-resources
NAME SHORTNAMES APIGROUP NAMESPACED KIND
绑定                                     true        绑定
componentstatuses cs                    false       ComponentStatus
configmaps cm                    true        ConfigMap
端点ep                    true        端点
事件ev                    true        事件
limitranges限制                true        LimitRange
名称空间ns                    false       命名空间
节点没有                    false       节点
persistentvolumeclaims pvc                   true       PersistentVolumeClaim
persistentvolumes pv                    false       PersistentVolume
pods po                    true        pod
statefulsets sts apps      true        StatefulSet
...
```

再次，`kubectl`了解短名称通过发现信息（参见[“发现信息”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#discovery)）。这是一个例子：

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ats.cnat.programming-kubernetes.info
spec:
  ...
  shortNames:
  - at
```

之后，a `kubectl get at`将列出`cnat`命名空间中的所有CR。

此外，CR-与任何其他资源一样 - 可以是其中的一部分类别。最常见的用途是`all`类别，如`kubectl get all`。它列出了群集中所有面向用户的资源，如pod和服务。

群集中定义的CR可以通过以下`categories`字段加入类别或创建自己的类别：

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ats.cnat.programming-kubernetes.info
spec:
  ...
  categories:
  - all
```

有了这个，`kubectl get all`还会`cnat`在命名空间中列出CR。

## 打印机列

该`kubectl`CLI工具使用服务器端打印来呈现输出`kubectl get`。这意味着它向API服务器查询要显示的列以及每行中的值。

自定义资源也支持服务器端打印机列`additionalPrinterColumns`。它们被称为“附加”，因为第一列始终是对象的名称。这些列的定义如下：

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: ats.cnat.programming-kubernetes.info
spec:
  additionalPrinterColumns: (optional)
  - name: kubectl column name
    type: OpenAPI type for the column
    format: OpenAPI format for the column (optional)
    description: human-readable description of the column (optional)
    priority: integer, always zero supported by kubectl
    JSONPath: JSON path inside the CR for the displayed value
```

该`name`字段是列名，`type`是规范的[数据类型](http://bit.ly/2N0DSY4)部分中定义的OpenAPI类型，并且`format`（在同一文档中定义）是可选的，可能由`kubectl`其他客户端解释。

此外，`description`是一个可选的人类可读字符串，用于文档目的。显示列的`priority`详细模式的控件`kubectl`。在撰写本文时（使用Kubernetes 1.14），仅支持零，并且隐藏所有具有更高优先级的列。

最后，`JSONPath`定义要显示的值。它是CR内部的简单JSON路径。这里，“简单”意味着它支持对象字段语法`.spec.foo.bar`，但不支持循环遍历数组或类似的更复杂的JSON路径。

有了这个，引言中的示例CRD可以`additionalPrinterColumns`像这样扩展：

```
additionalPrinterColumns: #(optional)
- name: schedule
  type: string
  JSONPath: .spec.schedule
- name: command
  type: string
  JSONPath: .spec.command
- name: phase
  type: string
  JSONPath: .status.phase
```

然后`kubectl`将呈现`cnat`如下资源：

```
$ kubectl得到了
名称SCHEDULER命令阶段
foo 2019-07-03T02：00：00Z   echo "hello world"  待定
```

接下来，我们来看看子资源。

## 子资源

我们简要提到了[“Status Subresources：UpdateStatus”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#client-go-subresource)中的子资源。子资源是特殊的HTTP端点，使用附加到普通资源的HTTP路径的后缀。例如，pod标准HTTP路径是*/ api / v1 / namespace / namespace/ pods /name*。Pod有许多子资源，例如*/ logs*，*/ portforward*，*/ exec*和*/ status*。相应的子资源HTTP路径是：

- */ api / v1 / namespace /* `namespace`*/ pods /* `name`*/ logs*
- */ api / v1 / namespace /* `namespace`*/ pods /* `name`*/ portforward*
- */ api / v1 / namespace /* `namespace`*/ pods /* `name`*/ exec*
- */ api / v1 / namespace /* `namespace`*/ pods /* `name`*/ status*

子资源端点使用与主资源端点不同的协议。

在撰写本文时，自定义资源支持两个子资源：*/ scale*和*/ status*。两者都是选择性的，即必须在CRD中明确启用它们。

### 状态子资源

该*/状态*子资源用于从控制器提供的状态中拆分用户提供的CR实例规范。这样做的主要动机是特权分离：

- 用户通常不应该写状态字段。
- 控制器不应写入规范字段。

该用于访问控制的RBAC机制不允许在该详细级别上的规则。这些规则始终是每个资源。该*/状态*子资源通过提供两个端点是自己的资源解决这个问题。每个都可以独立地使用RBAC规则进行控制。这通常称为*规范状态拆分*。以下是`ats`资源的此类规则的示例，该规则仅适用于*/ status*子资源（同时`"ats"`与主资源匹配）：

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: ...
rules:
- apiGroups: [""]
  resources: ["ats/status"]
  verbs: ["update", "patch"]
```

具有*/ status*子资源的资源（包括自定义资源）已更改语义，也适用于主资源端点：

- 它们忽略在创建期间对主HTTP端点上的状态的更改（在创建期间刚刚删除状态）和更新。
- 同样，*/ status*子资源端点忽略有效负载状态之外的更改。无法在*/ status*端点上创建操作。
- 每当外部`metadata`和外部的`status`变化（这尤其意味着规范的变化）时，主资源端点将增加该`metadata.generation`值。这可以用作控制器的触发器，指示用户期望已经改变。

注意通常都`spec`和`status`在更新请求被发送，但在技术上你可以留出一个请求负载相应的另一个部分。

另请注意，*/ status*端点将忽略*状态*之外的所有内容，包括标签或注释等元数据更改。

自定义资源的规范状态拆分启用如下：

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
spec:
  subresources:
    status: {}
  ...
```

请注意`status`，YAML片段中的字段被分配了空对象。这是设置没有其他属性的字段的方法。只是写作

```
subresources:
  status:
```

将导致验证错误，因为在YAML中，结果是`null`值`status`。

###### 警告

启用规范状态拆分是API的重大变化。旧控制器将写入主端点。他们不会注意到从激活分割的位置始终忽略状态。同样，在激活拆分之前，新控制器无法写入新的*/状态*端点。

在Kubernetes 1.13及更高版本中，可以为每个版本配置子资源。这允许我们引入*/ status*子资源而不会发生重大变化：

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
spec:
  ...
  versions:
  - name: v1alpha1
    served: true
    storage: true
  - name: v1beta1
    served: true
    subresources:
      status: {}
```

这将启用*/ status*子资源`v1beta1`，但不能用于`v1alpha1`。

###### 注意

乐观并发语义（参见[“乐观并发”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#optimistic-concurrency)）与主要资源端点相同; 也就是说，`status`并且`spec`共享相同的资源版本计数器和*/状态*更新可能由于写入主资源而发生冲突，反之亦然。换句话说，存储层上没有分割`spec`和分割`status`。

### 扩展子资源

该可用于自定义资源的第二个子资源是*/ scale*。的*/尺度*子资源为（投影）[2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#idm46336862332104)上的资源视图，使我们能够查看和修改仅复制值。这个 subresource以Kubernetes中的部署和副本集等资源而闻名，显然可以扩展和缩小。

该`kubectl scale`命令使用*/ scale*子资源; 例如，以下内容将修改给定实例中的指定副本值：

```
$ kubectl scale --replicas=3 your-custom-resource -v=7
I0429 21:17:53.138353   66743 round_trippers.go:383] PUT
https://host/apis/group/v1/your-custom-resource/scale
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
spec:
  subresources:
    scale:
      specReplicasPath: .spec.replicas
      statusReplicasPath: .status.replicas
      labelSelectorPath: .status.labelSelector
  ...
```

这样，`spec.replicas`在a期间，将副本值的更新写入并从那里返回`GET`。

标签选择器不能通过*/ status*子资源更改，只能读取。其目的是为控制器提供计算相应对象的信息。例如，`ReplicaSet`控制器计算满足此选择器的相应pod。

标签选择器是可选的。如果您的自定义资源语义不适合标签选择器，则不要为其指定JSON路径。

在前面`kubectl scale --replicas=3 ...`的值示例`3`写入`spec.replicas`。当然，可以使用任何其他简单的JSON路径; 例如，`spec.instances`或者`spec.size`是一个合理的字段名称，具体取决于上下文。

##### 副本整数值与创建和删除副本的控制器

我们只讨论在自定义资源中读取和设置副本整数值。其背后的实际语义 - 例如，创建和删除实际副本的实例 - 必须由自定义控制器实现（请参阅[“控制器和操作员”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#ch_controllers-operators)）。

从端点读取或写入的对象类型`Scale`来自`autoscaling/v1`API组。这是它的样子：

```
type Scale struct {
    metav1.TypeMeta `json:",inline"`
    // Standard object metadata; More info: https://git.k8s.io/
    // community/contributors/devel/api-conventions.md#metadata.
    // +optional
    metav1.ObjectMeta `json:"metadata,omitempty"`

    // defines the behavior of the scale. More info: https://git.k8s.io/community/
    // contributors/devel/api-conventions.md#spec-and-status.
    // +optional
    Spec ScaleSpec `json:"spec,omitempty"`

    // current status of the scale. More info: https://git.k8s.io/community/
    // contributors/devel/api-conventions.md#spec-and-status. Read-only.
    // +optional
    Status ScaleStatus `json:"status,omitempty"`
}

// ScaleSpec describes the attributes of a scale subresource.
type ScaleSpec struct {
    // desired number of instances for the scaled object.
    // +optional
    Replicas int32 `json:"replicas,omitempty"`
}

// ScaleStatus represents the current status of a scale subresource.
type ScaleStatus struct {
    // actual number of observed instances of the scaled object.
    Replicas int32 `json:"replicas"`

    // label query over pods that should match the replicas count. This is the
    // same as the label selector but in the string format to avoid
    // introspection by clients. The string will be in the same
    // format as the query-param syntax. More info about label selectors:
    // http://kubernetes.io/docs/user-guide/labels#label-selectors.
    // +optional
    Selector string `json:"selector,omitempty"`
}
```

实例将如下所示：

```
metadata:
  name: cr-name
  namespace: cr-namespace
  uid: cr-uid
  resourceVersion: cr-resource-version
  creationTimestamp: cr-creation-timestamp
spec:
  replicas: 3
  status:
    replicas: 2
    selector: "environment = production"
```

请注意，主要资源和*/ scale*子资源的乐观并发语义相同。也就是说，主要资源写入可能与*/ scale*写入冲突，反之亦然。

# 开发人员对自定义资源的看法

自定义资源可以使用许多客户端从Golang访问。我们将专注于：

- 使用`client-go`动态客户端（请参阅[“动态客户端”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#dynamic-client)）
- 使用键入的客户端：
  - 由[kubernetes-sigs / controller-runtime提供](http://bit.ly/2ZFtDKd)，并由Operator SDK和Kubebuilder使用（请参阅[“](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#controller-runtime)Operator SDK和Kubebuilder的[控制器 - 运行时客户端”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#controller-runtime)）
  - 正如[*k8s.io/client-go/kubernetes中*](http://bit.ly/2FnmGWA)所生成的`client-gen`那样（参见[“通过client-gen创建的类型客户端”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#clientgen-client)）

选择使用哪个客户端主要取决于要编写的代码的上下文，尤其是实现的逻辑和要求的复杂性（例如，动态和支持在编译时未知的GVK）。

前面的客户列表：

- 降低处理未知GVK的灵活性。
- 增加类型安全性。
- 增加了他们提供的Kubernetes API功能的完整性。

## 动态客户端

该[*k8s.io/client-go/dynamic中的*](http://bit.ly/2Y6eeSK)动态客户端与已知的GVK完全无关。除了[*unstructured.Unstructured之外*](http://bit.ly/2WYZ6oS)，它甚至不使用任何Go类型，它包含just `json.Unmarshal`和它的输出。

动态客户端既不使用方案也不使用RESTMapper。这意味着开发人员必须通过以GVR的形式提供资源（请参阅[“资源”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#resources)）来手动提供有关类型的所有知识：

```
schema.GroupVersionResource{
  Group: "apps",
  Version: "v1",
  Resource: "deployments",
}
```

如果可以使用REST客户端配置（请参阅[“创建和使用客户端”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#rest-client-config)），可以在一行中创建动态客户端：

```
client, err := NewForConfig(cfg)
```

对给定GVR的REST访问非常简单：

```
client.Resource(gvr).
   Namespace(namespace).Get("foo", metav1.GetOptions{})
```

这使您可以`foo`在给定的命名空间中进行部署。

###### 注意

您必须知道资源的范围（即，它是命名空间还是集群作用域）。集群范围的资源只是忽略了`Namespace(namespace)`调用。

动态客户端的输入和输出`*unstructured.Unstructured`是一个对象，它包含`json.Unmarshal`在解组时输出的相同数据结构：

- 对象用表示`map[string]interface{}`。
- 数组表示为`[]interface{}`。
- 原始类型`string`，`bool`，`float64`，或`int64`。

该方法`UnstructuredContent()`提供对非结构化对象内部的数据结构的访问（我们也可以访问`Unstructured.Object`）。在同一个包中有助手可以轻松检索字段并可以对对象进行操作 - 例如：

```
name, found, err := unstructured.NestedString(u.Object, "metadata", "name")
```

`"foo"`在这种情况下返回部署的名称。`found`如果实际找到该字段（不仅是空的，而是实际存在的），则为真。`err`报告现有字段的类型是否是意外的（即，在这种情况下不是字符串）。其他助手是通用助手，一次是结果的深层副本，一次是没有：

```
func NestedFieldCopy(obj map[string]interface{}, fields ...string)
  (interface{}, bool, error)
func NestedFieldNoCopy(obj map[string]interface{}, fields ...string)
  (interface{}, bool, error)
```

还有其他类型的变体进行类型转换，如果失败则返回错误：

```
func NestedBool(obj map[string]interface{}, fields ...string) (bool, bool, error)
func NestedFloat64(obj map[string]interface{}, fields ...string)
  (float64, bool, error)
func NestedInt64(obj map[string]interface{}, fields ...string) (int64, bool, error)
func NestedStringSlice(obj map[string]interface{}, fields ...string)
  ([]string, bool, error)
func NestedSlice(obj map[string]interface{}, fields ...string)
  ([]interface{}, bool, error)
func NestedStringMap(obj map[string]interface{}, fields ...string)
  (map[string]string, bool, error)
```

最后一个通用的setter：

```
func SetNestedField(obj, value, path...)
```

动态客户端在Kubernetes中用于通用控制器，如垃圾收集控制器，它删除父项已消失的对象。垃圾收集控制器可以与系统中的任何资源一起使用，因此可以广泛使用动态客户端。

## 键入的客户端

键入的客户端不要使用`map[string]interface{}`类似的通用数据结构，而是使用真实的Golang类型，每个GVK都不同且具体。它们更易于使用，大大提高了类型安全性，并使代码更简洁，更易读。缺点是，它们的灵活性较低，因为必须在编译时知道已处理的类型，并生成这些客户端，这会增加复杂性。

在进入类型化客户端的两个实现之前，让我们看看Golang类型系统中各种类型的表示（有关Kubernetes类型系统背后的理论，请参阅[“深度API机械”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#api-machinery-core)）。

### 一种类型的解剖

种表示为Golang结构。通常结构被命名为类型（虽然从技术上讲它不必是），并且被放置在与手头的GVK的组和版本相对应的包中。常见的惯例是放置GVK *group*/ *version*。*Kind*进入Go包：

```
pkg / apis / group / version
```

并*Kind*在文件*types.go中*定义Golang结构。

对应于GVK的每个Golang类型都嵌入`TypeMeta`了包[*k8s.io/apimachinery/pkg/apis/meta/v1中*](http://bit.ly/2Y5HdWT)的结构。`TypeMeta`只包括`Kind`和`ApiVersion`字段：

```
type TypeMeta struct {
    // +optional
    APIVersion string `json:"apiVersion,omitempty" yaml:"apiVersion,omitempty"`
    // +optional
    Kind string `json:"kind,omitempty" yaml:"kind,omitempty"`
}
```

此外，每个顶级类型 - 即具有自己的端点并因此具有一个（或多个）对应的GVR（参见[“REST映射”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#RESTMapping)） - 具有存储名称，命名空间资源的命名空间和漂亮的类型。大量进一步的元级域。所有这些都存储在[*k8s.io/apimachinery/pkg/apis/meta/v1*](http://bit.ly/2XSt8eo)`ObjectMeta`包中调用的结构中：

```
type ObjectMeta struct {
    Name string `json:"name,omitempty"`
    Namespace string `json:"namespace,omitempty"`
    UID types.UID `json:"uid,omitempty"`
    ResourceVersion string `json:"resourceVersion,omitempty"`
    CreationTimestamp Time `json:"creationTimestamp,omitempty"`
    DeletionTimestamp *Time `json:"deletionTimestamp,omitempty"`
    Labels map[string]string `json:"labels,omitempty"`
    Annotations map[string]string `json:"annotations,omitempty"`
    ...
}
```

还有许多其他字段。我们强烈建议您阅读[大量的内联文档](http://bit.ly/2IutNyh)，因为它可以很好地描述Kubernetes对象的核心功能。

Kubernetes顶级类型（即那些具有嵌入式`TypeMeta`和嵌入式`ObjectMeta`，并且在这种情况下持久化的类型`etcd`）看起来非常相似，因为它们通常具有a `spec`和a `status`。从[*k8s.io/kubernetes/apps/v1/types.go*](http://bit.ly/2RroTFb)查看此部署[*示例*](http://bit.ly/2RroTFb)：

```
type Deployment struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec DeploymentSpec `json:"spec,omitempty"`
    Status DeploymentStatus `json:"status,omitempty"`
}
```

虽然不同类型的类型的实际内容`spec`和`status`不同类型之间存在显着差异，但这在Kubernetes中分为`spec`并且`status`是一个共同主题甚至是惯例，尽管它在技术上并不需要。因此，优良作法也是遵循这种CRD结构。一些CRD功能甚至需要这种结构; 例如，自定义资源的*/ status*子资源（请参阅[“状态子资源”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#status-subresource)） - 启用时 - 始终仅应用于`status`自定义资源实例的子结构。它无法重命名。

### GOLANG封装结构

如我们已经看到，Golang类型传统上放在包*pkg / apis /* */中*名为*types.go*的文件中。除了该文件之外，还有一些我们想要浏览的文件。其中一些是由开发人员手动编写的，而另一些是使用代码生成器生成的。详细信息请参见[第5章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#ch_autocodegen)。*groupversion*

该*doc.go*文件描述了API的目的，包括许多包全局代码生成标签：

```
// Package v1alpha1 contains the cnat v1alpha1 API group
//
// +k8s:deepcopy-gen=package
// +groupName=cnat.programming-kubernetes.info
package v1alpha1
```

接下来，*register.go*包含帮助程序以将自定义资源Golang类型注册到方案中（请参阅[“方案”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#scheme)）：

```
package version

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/runtime/schema"

    group "repo/pkg/apis/group"
)

// SchemeGroupVersion is group version used to register these objects
var SchemeGroupVersion = schema.GroupVersion{
    Group: group.GroupName,
    Version: "version",
}

// Kind takes an unqualified kind and returns back a Group qualified GroupKind
func Kind(kind string) schema.GroupKind {
    return SchemeGroupVersion.WithKind(kind).GroupKind()
}

// Resource takes an unqualified resource and returns a Group
// qualified GroupResource
func Resource(resource string) schema.GroupResource {
    return SchemeGroupVersion.WithResource(resource).GroupResource()
}

var (
    SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
    AddToScheme   = SchemeBuilder.AddToScheme
)

// Adds the list of known types to Scheme.
func addKnownTypes(scheme *runtime.Scheme) error {
    scheme.AddKnownTypes(SchemeGroupVersion,
        &SomeKind{},
        &SomeKindList{},
    )
    metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
    return nil
}
```

然后，*zz_generated.deepcopy.go*定义在自定义资源Golang顶级类型深复制方法（即，`SomeKind`与`SomeKindList`在前面的示例代码）。此外，所有substructs（像那些为`spec`和`status`）成为深可复制为好。

因为本例使用的标签`+k8s:deepcopy-gen=package`在*doc.go*，深拷贝世代是选择退出的基础上; 也就是说，`DeepCopy`为包中没有选择退出的每种类型生成方法`+k8s:deepcopy-gen=false`。有关详细信息，请参阅[第5章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#ch_autocodegen)，尤其是[“deepcopy-gen标签”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#deepcopy-tags)。

### 通过CLIENT-GEN创建的类型客户端

与API包*PKG的/ apis / group/version*到位，客户端发生器`client-gen`创建一个输入客户端（参见[第5章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#ch_autocodegen)的详细信息，特别是[“客户端根标签”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#clientgen-tags)），在*PKG /生成/ clientset /版本*默认（PKG /客户端/客户端/版本化的旧版本的生成器）。更确切地说，生成的顶级对象是客户端集。它包含许多API组，版本和资源。

该[顶层文件](http://bit.ly/2GdcikH)如下所示：

```
// Code generated by client-gen. DO NOT EDIT.

package versioned

import (
    discovery "k8s.io/client-go/discovery"
    rest "k8s.io/client-go/rest"
    flowcontrol "k8s.io/client-go/util/flowcontrol"

    cnatv1alpha1 ".../cnat/cnat-client-go/pkg/generated/clientset/versioned/
)

type Interface interface {
    Discovery() discovery.DiscoveryInterface
    CnatV1alpha1() cnatv1alpha1.CnatV1alpha1Interface
}

// Clientset contains the clients for groups. Each group has exactly one
// version included in a Clientset.
type Clientset struct {
    *discovery.DiscoveryClient
    cnatV1alpha1 *cnatv1alpha1.CnatV1alpha1Client
}

// CnatV1alpha1 retrieves the CnatV1alpha1Client
func (c *Clientset) CnatV1alpha1() cnatv1alpha1.CnatV1alpha1Interface {
    return c.cnatV1alpha1
}

// Discovery retrieves the DiscoveryClient
func (c *Clientset) Discovery() discovery.DiscoveryInterface {
   ...
}

// NewForConfig creates a new Clientset for the given config.
func NewForConfig(c *rest.Config) (*Clientset, error) {
    ...
}
```

客户端集由接口表示，`Interface`并为每个版本提供对API组客户端接口的访问权限 - 例如，`CnatV1alpha1Interface`在此示例代码中：

```
type CnatV1alpha1Interface interface {
    RESTClient() rest.Interface
    AtsGetter
}

// AtsGetter has a method to return a AtInterface.
// A group's client should implement this interface.
type AtsGetter interface {
    Ats(namespace string) AtInterface
}

// AtInterface has methods to work with At resources.
type AtInterface interface {
    Create(*v1alpha1.At) (*v1alpha1.At, error)
    Update(*v1alpha1.At) (*v1alpha1.At, error)
    UpdateStatus(*v1alpha1.At) (*v1alpha1.At, error)
    Delete(name string, options *v1.DeleteOptions) error
    DeleteCollection(options *v1.DeleteOptions, listOptions v1.ListOptions) error
    Get(name string, options v1.GetOptions) (*v1alpha1.At, error)
    List(opts v1.ListOptions) (*v1alpha1.AtList, error)
    Watch(opts v1.ListOptions) (watch.Interface, error)
    Patch(name string, pt types.PatchType, data []byte, subresources ...string)
        (result *v1alpha1.At, err error)
    AtExpansion
}
```

可以使用`NewForConfig`辅助函数创建客户端集的实例。这类似于[“创建和使用客户端”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#rest-client-config)讨论的核心Kubernetes资源[的客户端](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#rest-client-config)：

```
import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/tools/clientcmd"

    client "github.com/.../cnat/cnat-client-go/pkg/generated/clientset/versioned"
)

kubeconfig = flag.String("kubeconfig", "~/.kube/config", "kubeconfig file")
flag.Parse()
config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
clientset, err := client.NewForConfig(config)

ats := clientset.CnatV1alpha1Interface().Ats("default")
book, err := ats.Get("kubernetes-programming", metav1.GetOptions{})
```

如您所见，代码生成机制允许我们以与核心Kubernetes资源相同的方式为自定义资源编写逻辑。也可以使用像线人这样的高级工具; 看`informer-gen`在[第5章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#ch_autocodegen)。

## operator SDK和Kubebuilder的controller-runtime客户端

对于为了完整起见，我们想快速浏览一下第三个客户端，它被列为[“开发人员对自定义资源的视图”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#crd-dev)的第二个选项。该`controller-runtime`项目为[第6章中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch06.html#ch_operator-solutions)介绍的运营商解决方案Operator SDK和Kubebuilder提供了基础。它包括一个使用“类型[剖析”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#anatomy-of-CRD-types)提供的Go类型的客户端。

与`client-gen`先前[“通过客户端创建的类型化客户端”的生成客户端相比](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#clientgen-client)，并且类似于[“动态客户端”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#dynamic-client)，该客户端是一个实例，能够处理在给定方案中注册的任何类型。

它使用来自API服务器的发现信息将种类映射到HTTP路径。请注意，[第6章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch06.html#ch_operator-solutions)将详细介绍如何将此客户端用作这两个运算符解决方案的一部分。

以下是如何使用的快速示例`controller-runtime`：

```
import (
    "flag"

    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes/scheme"
    "k8s.io/client-go/tools/clientcmd"

    runtimeclient "sigs.k8s.io/controller-runtime/pkg/client"
)

kubeconfig = flag.String("kubeconfig", "~/.kube/config", "kubeconfig file path")
flag.Parse()
config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)

cl, _ := runtimeclient.New(config, client.Options{
    Scheme: scheme.Scheme,
})
podList := &corev1.PodList{}
err := cl.List(context.TODO(), client.InNamespace("default"), podList)
```

客户端对象的`List()`方法接受`runtime.Object`在给定方案中注册的任何方法，在这种情况下是从`client-go`所有标准Kubernetes种类中注册的方案。在内部，客户端使用给定的方案将Golang类型映射`*corev1.PodList`到GVK。在第二步中，该`List()`方法使用发现信息来获取pod的GVR `schema.GroupVersionResource{"", "v1", "pods"}`，因此访问*/ api / v1 / namespace / default / pods*以获取传递的命名空间中的pod列表。

相同的逻辑可以与自定义资源一起使用。主要区别是使用包含传递的Go类型的自定义方案：

```
import (
    "flag"

    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes/scheme"
    "k8s.io/client-go/tools/clientcmd"

    runtimeclient "sigs.k8s.io/controller-runtime/pkg/client"
    cnatv1alpha1 "github.com/.../cnat/cnat-kubebuilder/pkg/apis/cnat/v1alpha1"
)

kubeconfig = flag.String("kubeconfig", "~/.kube/config", "kubeconfig file")
flag.Parse()
config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)

crScheme := runtime.NewScheme()
cnatv1alpha1.AddToScheme(crScheme)

cl, _ := runtimeclient.New(config, client.Options{
    Scheme: crScheme,
})
list := &cnatv1alpha1.AtList{}
err := cl.List(context.TODO(), client.InNamespace("default"), list)
```

请注意`List()`命令的调用根本不会发生变化。

想象一下，您编写了一个使用此客户端访问许多不同类型的运算符。使用[“通过client-gen创建](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#clientgen-client)的类型化客户端[”的类型客户端](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#clientgen-client)，您必须将许多不同的客户端传递给操作员，使得管道代码非常复杂。相比之下，`controller-runtime`这里介绍的客户只是所有类型的一个对象，假设它们都在一个方案中。

所有三种类型的客户端都有其用途，根据使用它们的上下文有利有弊。在处理未知对象的通用控制器中，只能使用动态客户端。在类型安全有助于强制执行代码正确性的控制器中，生成的客户端非常适合。Kubernetes项目本身有很多贡献者，即使代码的稳定性非常重要，即使它被很多人扩展和重写。如果方便和高速度和最小的管道是重要的，`controller-runtime`客户是一个很好的选择。

# 摘要

我们向您介绍了自定义资源，本章中Kubernetes生态系统中使用的中心扩展机制。到目前为止，您应该很好地了解它们的功能和限制以及可用的客户端。

现在让我们继续使用代码生成来管理所述资源。

[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#idm46336863073336-marker)不要在这里混淆Kubernetes和JSON对象。后者只是字符串映射的另一个术语，用于JSON和OpenAPI的上下文中。

[2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#idm46336862332104-marker) “投影”在这里意味着`scale`对象是主要资源的投影，因为它只显示某些字段并隐藏其他所有字段。