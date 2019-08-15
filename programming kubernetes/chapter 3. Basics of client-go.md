# 第3章客户端的基础知识

我们现在将重点关注Kubernetes编程接口在Go。您将学习如何访问众所周知的本机类型（如pod，服务和部署）的Kubernetes API。在后面的章节中，这些技术将扩展到用户定义的类型。但是，我们首先关注每个Kubernetes集群附带的所有API对象。

# 存储库

该Kubernetes项目在GitHub上的*kubernetes*组织下提供了许多第三方可消费Git存储库。您需要使用域别名*k8s.io / ...*（不是*github.com/kubernetes/...*）将所有这些导入到您的项目中。我们将在以下部分介绍这些存储库中最重要的存储库。

## 客户端库

该Go中的Kubernetes编程接口主要由*k8s.io/client-go*库组成（为简洁起见，我们将其称之为`client-go`前进）。*client-go*是一个典型的Web服务客户端库，支持所有正式属于Kubernetes的API类型。它可以用来执行通常的 REST动词：

- *创建*
- *得到*
- *名单*
- *更新*
- *删除*
- *补丁*

这些REST动词中的每一个都使用[“API服务器的HTTP接口”实现](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#api-server-http-interface)。此外，`Watch`支持动词，这对于类似Kubernetes的API是特殊的，并且是与其他API相比的主要区别之一。

[`client-go`](http://bit.ly/2RryyLM) 是可在GitHub上获得（参见[图3-1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#github-client-go)），并在Go代码中使用*k8s.io/client-go*软件包名称。它与Kubernetes本身并行运送; 也就是说，对于每个Kubernetes `1.x.y`版本，都有一个`client-go`带有匹配标记的版本`kubernetes-1.x.y`。

![Github上的`client-go`存储库](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0301.png)

###### 图3-1。GitHub上的client-go存储库

另外，还有语义版本计划。例如，`client-go`9.0.0匹配Kubernetes 1.12版本，`client-go`10.0.0匹配Kubernetes 1.13，依此类推。未来可能会有更细粒度的版本。除了Kubernetes API对象的客户端代码外，`client-go`还包含许多通用库代码。这也用于[第4章中的](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#ch_crds)用户定义的API对象。有关软件包列表，请参[见图3-1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#github-client-go)。

虽然所有软件包都可以使用，但是大多数与Kubernetes API相关的代码都将使用*tools / clientcmd /*从`kubeconfig`文件中设置客户端，而*kubernetes /则*用于实际的Kubernetes API客户端。我们很快就会看到代码这样做。在此之前，让我们快速浏览一下其他相关的存储库和软件包。

## Kubernetes API类型

如我们已经看到，`client-go`拥有客户端接口。pod，服务和部署等对象的Kubernetes API Go类型位于[其自己的存储库中](http://bit.ly/2ZA6dWH)。它`k8s.io/api`在Go代码中被访问。

Pod是遗留API组（通常也称为“核心”组）版本的一部分`v1`。因此，`Pod`Go类型位于*k8s.io/api/core/v1中*，类似于*Kubernetes*中的所有其他API类型。看到 [图3-2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#github-api)显示了包列表，其中大多数都与Kubernetes API组及其版本相对应。

实际的Go类型包含在*types.go*文件中（例如，*k8s.io / api / core / v1 / type.go*）。此外，还有其他文件，其中大部分是由代码生成器自动生成的。

![Github上的API存储库](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0302.png)

###### 图3-2。GitHub上的API存储库

## API机械

持续但并非最不重要的是，还有一个名为[API Machinery](http://bit.ly/2xAZiR2)的第三个存储库，用于`k8s.io/apimachinery`Go。它包括实现Kubernetes类API的所有通用构建块。API Machinery不仅限于容器管理，因此，例如，它可用于为在线商店或任何其他特定于业务的域构建API。

不过，您将在Kubernetes-native Go代码中遇到很多API Machinery软件包。一个重要的是*k8s.io/apimachinery/pkg/apis/meta/v1。*它包含了许多通用的API类型如`ObjectMeta`，`TypeMeta`，`GetOptions`，和`ListOptions`（参见[图3-3](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#github-apimachinery)）。

![Github上的API Machinery存储库](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0303.png)

###### 图3-3。GitHub上的API Machinery存储库

## 创建和使用客户端

现在我们知道创建Kubernetes客户端对象的所有构建块，这意味着我们可以访问Kubernetes集群中的资源。假设您可以访问本地环境中的集群（即，`kubectl`已正确设置并配置了凭据），以下代码说明了如何`client-go`在Go项目中使用：

```
import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/tools/clientcmd"
    "k8s.io/client-go/kubernetes"
)

kubeconfig = flag.String("kubeconfig", "~/.kube/config", "kubeconfig file")
flag.Parse()
config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
clientset, err := kubernetes.NewForConfig(config)

pod, err := clientset.CoreV1().Pods("book").Get("example", metav1.GetOptions{})
```

代码导入`meta/v1`包以获取访问权限`metav1.GetOptions`。此外，它`clientcmd`从中导入`client-go`以便读取和解析 kubeconfig（即具有服务器名称，凭据等的客户端配置）。然后，它`client-go``kubernetes`使用Kubernetes资源的客户端集导入包。

kubeconfig文件的默认位置位于用户主目录的*.kube / config*中。这也是`kubectl`获取Kubernetes集群凭据的地方。

然后使用读取和解析kubeconfig `clientcmd.BuildConfigFromFlags`。我们在整个代码中省略了强制性错误处理，但是`err`如果kubeconfig格式不正确，变量通常会包含语法错误。由于语法错误在Go代码中很常见，因此应该正确检查这样的错误，如下所示：

```
config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
if err != nil {
    fmt.Printf("The kubeconfig cannot be loaded: %v\n", err
    os.Exit(1)
}
```

从`clientcmd.BuildConfigFromFlags`我们得到一个`rest.Config`，您可以在*k8s.io/client-go/rest*包中找到。这个传递`kubernetes.NewForConfig`给以创建实际的Kubernetes *客户端集*。它被称为*客户端集，*因为它包含所有本地Kubernetes资源的多个客户端。

在群集中的pod中运行二进制文件时，`kubelet`将自动将服务帐户装入容器*/var/run/secrets/kubernetes.io/serviceaccount*。它替换刚刚提到的kubeconfig文件，可以很容易地`rest.Config`通过该`rest.InClusterConfig()`方法转换为。你经常会发现下列组合`rest.InClusterConfig()`并`clientcmd.BuildConfigFromFlags()`，包括支持`KUBECONFIG`环境变量：

```
config, err := rest.InClusterConfig()
if err != nil {
    // fallback to kubeconfig
    kubeconfig := filepath.Join("~", ".kube", "config")
    if envvar := os.Getenv("KUBECONFIG"); len(envvar) >0 {
        kubeconfig = envvar
    }
    config, err = clientcmd.BuildConfigFromFlags("", kubeconfig)
    if err != nil {
        fmt.Printf("The kubeconfig cannot be loaded: %v\n", err
        os.Exit(1)
    }
}
```

在下面的示例代码，我们在选择核心小组`v1`与`clientset.CoreV1()`再访问该吊舱`"example"`的`"book"`命名空间：

```
 pod, err := clientset.CoreV1().Pods("book").Get("example", metav1.GetOptions{})
```

请注意，只有最后一个函数调用`Get`实际访问服务器。无论`CoreV1`和`Pods`选择客户，并只可用于以下设置的命名空间`Get`调用（这通常被称为所述*生成器模式*，在这种情况下建立的请求）。

该`Get`调用将HTTP `GET`请求发送到服务器上的*/ api / v1 / namespaces / book / pods / example*，该请求在kubeconfig中设置。如果Kubernetes API服务器使用HTTP代码`200`进行应答，则响应的主体将携带编码的pod对象，或者作为`client-go`JSON-作为协议缓冲区的默认有线格式。

###### 注意

You 通过在从其创建客户端之前修改REST配置，可以为本机Kubernetes资源客户端启用protobuf：

```
cfg, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
cfg.AcceptContentTypes = "application/vnd.kubernetes.protobuf,
                          application/json"
cfg.ContentType = "application/vnd.kubernetes.protobuf"
clientset, err := kubernetes.NewForConfig(cfg)
```

请注意，[第4章中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#ch_crds)介绍的自定义资源不支持协议缓冲区。

## 版本控制和兼容性

Kubernetes API是版本。我们在上一节中看到过pods属于`v1`核心组。核心组实际上现在只存在于一个版本中。还有其他组，例如，该`apps`组，存在于`v1`，`v1beta2,`和`v1beta1`（撰写本文时）。如果您查看[*k8s.io/api/apps*](http://bit.ly/2L1Nyio)包，您将找到这些版本的所有API对象。在[*k8s.io/client-go/kubernetes/typed/apps*](http://bit.ly/2x45Uab)包中，您将看到所有这些版本的客户端实现。

所有这些只是客户端。它没有说Kubernetes集群及其API服务器。使用具有API服务器不支持的API组版本的客户端将失败。客户端硬编码为某个版本，应用程序开发人员必须选择正确的API组版本才能与手头的群集通信。有关API组兼容性保证的更多信息，请参阅[“API版本和兼容性保证”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#api-versions)。

兼容性的第二个方面`client-go`是与之对话的API服务器的元API功能。例如，有选项结构CRUD动词，如`CreateOptions`，`GetOptions`，`UpdateOptions`，和`DeleteOptions`。另一个重要的是`ObjectMeta`（在[“ObjectMeta”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#ObjectMeta)中详细讨论），它是[各种类型的](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#ObjectMeta)一部分。所有这些都经常扩展到新功能; 我们通常称它们为*API机械功能*。在其字段的Go文档中，注释指定功能何时被视为alpha或beta。相同的API兼容性保证适用于任何其他API字段。

在下面的示例中，`DeleteOptions`结构在包[*k8s.io/apimachinery/pkg/apis/meta/v1/types.go中*](http://bit.ly/2MZ9flL)定义：

```
// DeleteOptions may be provided when deleting an API object.
type DeleteOptions struct {
    TypeMeta `json:",inline"`

    GracePeriodSeconds *int64 `json:"gracePeriodSeconds,omitempty"`
    Preconditions *Preconditions `json:"preconditions,omitempty"`
    OrphanDependents *bool `json:"orphanDependents,omitempty"`
    PropagationPolicy *DeletionPropagation `json:"propagationPolicy,omitempty"`

    // When present, indicates that modifications should not be
    // persisted. An invalid or unrecognized dryRun directive will
    // result in an error response and no further processing of the
    // request. Valid values are:
    // - All: all dry run stages will be processed
    // +optional
    DryRun []string `json:"dryRun,omitempty" protobuf:"bytes,5,rep,name=dryRun"`
}
```

最后一个字段，`DryRun`在Kubernetes 1.12中添加为alpha，在1.13中添加为beta（默认启用）。早期版本中的API服务器无法理解它。根据功能的不同，传递这样的选项可能会被忽略甚至被拒绝。因此，拥有一个`client-go`与群集版本相距太远的版本非常重要。

###### 小费

可以使用哪些字段的参考，其中质量级别是*k8s.io/api中*的源，例如，可以访问[`release-1.13`分支中的](http://bit.ly/2Yrhjgq) Kubernetes 1.13 。Alpha字段在其描述中标记为这样。

那里正在[生成API文档](http://bit.ly/2YrfiB2)，方便消费。但是，与*k8s.io/api中的*信息相同。

最后但并非最不重要的是，许多alpha和beta功能都有相应的[功能门](http://bit.ly/2RP5nmi)（请在此处查看[主要来源](http://bit.ly/2FPZPTT)）。功能在[问题中](http://bit.ly/2YuHYcd)被跟踪。

该集群和`client-go`版本之间正式保证的支持矩阵在`client-go` [README中](http://bit.ly/2RryyLM)发布（[参见表3-1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#client-go-compatibility)）。

|                | 在州长1.9 | 在州长1.10 | 在州长1.11 | 在州长1.12 | 在州长1.13 | 在州长1.14 | 在州长1.15 |
| :------------- | :-------- | :--------- | :--------- | :--------- | :--------- | :--------- | :--------- |
| client-go 6.0  | ✓         | +–         | +–         | +–         | +–         | +–         | +–         |
| client-go 7.0  | +–        | ✓          | +–         | +–         | +–         | +–         | +–         |
| 客户端去8.0    | +–        | +–         | ✓          | +–         | +–         | +–         | +–         |
| client-go 9.0  | +–        | +–         | +–         | ✓          | +–         | +–         | +–         |
| client-go 10.0 | +–        | +–         | +–         | +–         | ✓          | +–         | +–         |
| 客户去11.0     | +–        | +–         | +–         | +–         | +–         | ✓          | +–         |
| client-go 12.0 | +–        | +–         | +–         | +–         | +–         | +–         | ✓          |
| 客户 - 去HEAD  | +–        | +–         | +–         | +–         | +–         | +–         | +–         |

- ✓：两者`client-go`和Kubernetes版本具有相同的功能和相同的API组版本。
- `+`：`client-go`具有Kubernetes群集中可能不存在的功能或API组版本。这可能是因为`client-go`Kubernetes删除了旧的，已弃用的功能，因此增加了功能。但是，它们共有的一切（即大多数API）都可以使用。
- `–`：`client-go`明显与Kubernetes集群不兼容。

从外卖[表3-1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#client-go-compatibility)是该`client-go`库与它对应的集群版本的支持。在版本偏差的情况下，开发人员必须仔细考虑他们使用哪些功能和哪些API组，以及应用程序所说的群集版本是否支持这些功能。

在[表3-1中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#client-go-compatibility)，`client-go`列出了版本。我们简单地提到[“客户端库”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#client-go)即`client-go`使用语义版本化（semver）正式，但通过增加`client-go`每次的主要版本Kubernetes的次要版本（1.13.2中的13）增加。随着`client-go`Kubernetes 1.4发布1.0，我们现在`client-go`为Kubernetes 1.15的12.0（在撰写本文时）。

此semver仅适用于`client-go`自身，而不适用于API Machinery或API存储库。相反，后者使用Kubernetes版本进行标记，[如图3-4所示](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#client-go-versioning)。请参见[“Vendoring”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#vendoring)看看这是什么意思为vendoring *k8s.io/client-go*，*k8s.io/apimachinery*和*k8s.io/api*在您的项目。

![客户端版本控制](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0304.png)

###### 图3-4。客户端版本控制

## API版本和兼容性保证

如如果您使用代码定位不同的群集版本，那么在上一节中看到，选择正确的API组版本可能至关重要。Kubernetes版本所有API组。使用了一种常见的Kubernetes风格的版本控制方案，它由alpha，beta和GA（一般可用性）版本组成。

模式是：

- `v1alpha1`，`v1alpha2`，`v2alpha1`，等有称为*alpha版本*并被认为是不稳定的 这意味着：
  - 他们可能会以任何不相容的方式随时离开或改变。
  - 从Kubernetes版本到版本，数据可能会被丢弃，丢失或无法访问。
  - 如果管理员未手动选择，则默认情况下通常会禁用它们。
- `v1beta1`，`v1beta2`，`v2beta1`，等等，都称为*beta版本*。他们正在走向稳定，这意味着：
  - 对于至少一个Kubernetes版本，它们仍将与相应的稳定API版本并行存在。
  - 它们通常不会以不相容的方式改变，但没有严格的保证。
  - 存储在测试版中的对象不会被删除或无法访问。
  - 默认情况下，Beta版本通常在群集中启用。但这可能取决于所使用的Kubernetes分发或云提供商。
- `v1`，`v2`等等是稳定的，通常可用的API; 那是：
  - 他们会留下来。
  - 它们兼容。

###### 小费

Kubernetes有这些经验法则背后的[正式弃用政策](http://bit.ly/2FOrKU8)。您可以在[Kubernetes社区GitHub上](http://bit.ly/2XKPWAX)找到有关哪些API构造被认为兼容的更多详细信息。

与API组版本相关，要记住两个要点：

- API组版本作为一个整体应用于API资源，例如pod或服务的格式。除API组版本外，API资源可能具有正交版本的单个字段; 例如，稳定API中的字段可能在其内联代码文档中标记为alpha质量。与刚刚为API组列出的规则相同的规则将适用于这些字段。例如：

  - 稳定API中的字段字段可能会变得不兼容，丢失数据或随时消失。例如，`ObjectMeta.Initializers`从未提升过alpha 的字段将在不久的将来消失（在1.14中已弃用）：

    ```
    // DEPRECATED - initializers are an alpha field and will be removed
    // in v1.15.
    Initializers *Initializers `json:"initializers,omitempty"
    ```

  - 它通常在默认情况下处于禁用状态，必须使用API服务器功能门启用，如下所示：

    ```
    type JobSpec struct {
        ...
        // This field is alpha-level and is only honored by servers that
        // enable the TTLAfterFinished feature.
        TTLSecondsAfterFinished *int32 `json:"ttlSecondsAfterFinished,omitempty"
    }
    ```

  - API服务器的行为因字段而异。如果未启用相应的要素门，则会拒绝某些Alpha字段，并忽略某些字段。这在字段描述中有记录（参见`TTLSecondsAfterFinished`前面的示例）。

- 此外，API组版本在访问API时发挥作用。在同一资源的不同版本之间，有一个由API服务器完成的即时转换。也就是说，您可以`v1beta1`在任何其他受支持的版本（例如）中访问在一个版本（例如）中创建的对象，`v1`而无需在您的应用程序中进行任何进一步的工作。这对于构建向后兼容和向前兼容的应用程序非常方便。

  - 存储的每个对象`etcd`都存储在特定版本中。通过默认情况下，这称为该资源的*存储版本*。虽然存储版本可以从Kubernetes版本更改为版本，但存储的对象在`etcd`撰写本文时不会自动更新。因此，在删除旧版本支持之前，集群管理员必须确保在更新Kubernetes集群时及时进行迁移。没有通用的迁移机制，迁移不同于Kubernetes分发到分发。
  - 但是，对于应用程序开发人员来说，这项操作工作根本不重要。即时转换将确保应用程序具有群集中对象的统一图片。应用程序甚至不会注意到正在使用哪个存储版本。存储版本控制对于编写的Go代码是透明的。

# Go中的Kubernetes对象

在[“创建和使用客户端”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#rest-client-config)，我们了解了如何为核心组创建客户端以访问Kubernetes集群中的pod。在下文中，我们想要更详细地了解一下pod或任何其他Kubernetes资源，就此而言，就是Go的世界。

Kubernetes资源 - 或者更准确地说是对象 - 作为类型[1的](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#idm46336866123400)实例并且由API服务器作为资源提供，表示为结构。根据所涉及的种类，他们的领域当然不同。但另一方面，他们有一个共同的结构。

从类型系统的角度来看，Kubernetes对象实现了一个名为的Go接口 `runtime.Object`从包*k8s.io/apimachinery/pkg/runtime*，这实际上非常简单：

```
// Object interface must be supported by all API types registered with Scheme.
// Since objects in a scheme are expected to be serialized to the wire, the
// interface an Object must provide to the Scheme allows serializers to set
// the kind, version, and group the object is represented as. An Object may
// choose to return a no-op ObjectKindAccessor in cases where it is not
// expected to be serialized.
type Object interface {
    GetObjectKind() schema.ObjectKind
    DeepCopyObject() Object
}
```

在这里，`schema.ObjectKind`（来自*k8s.io/apimachinery/pkg/runtime/schema*包）是另一个简单的界面：

```
// All objects that are serialized from a Scheme encode their type information.
// This interface is used by serialization to set type information from the
// Scheme onto the serialized version of an object. For objects that cannot
// be serialized or have unique requirements, this interface may be a no-op.
type ObjectKind interface {
    // SetGroupVersionKind sets or clears the intended serialized kind of an
    // object. Passing kind nil should clear the current setting.
    SetGroupVersionKind(kind GroupVersionKind)
    // GroupVersionKind returns the stored group, version, and kind of an
    // object, or nil if the object does not expose or provide these fields.
    GroupVersionKind() GroupVersionKind
}
```

换句话说，Go中的Kubernetes对象是一种数据结构，可以：

- 返回*并*设置组版儿童
- 被*深刻复制*

一个*深拷贝*是数据结构的克隆，使其不与原始对象共享任何内存。它用于代码必须在不修改原始对象的情况下改变对象的任何地方。有关如何在Kubernetes中实现深层复制的详细信息，请参阅有关代码生成的[“全局标记”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#global-tags)。

简而言之，对象存储其类型并允许克隆。

## TypeMeta

虽然`runtime.Object`是只有一个界面，我们想知道它是如何实际实现的。来自*k8s.io/api的* Kubernetes对象`schema.ObjectKind`通过嵌入`metav1.TypeMeta`包*k8s.io/apimachinery/meta/v1中*的结构来实现类型getter和setter ：

```
// TypeMeta describes an individual object in an API response or request
// with strings representing the type of the object and its API schema version.
// Structures that are versioned or persisted should inline TypeMeta.
//
// +k8s:deepcopy-gen=false
type TypeMeta struct {
    // Kind is a string value representing the REST resource this object
    // represents. Servers may infer this from the endpoint the client submits
    // requests to.
    // Cannot be updated.
    // In CamelCase.
    // +optional
    Kind string `json:"kind,omitempty" protobuf:"bytes,1,opt,name=kind"`

    // APIVersion defines the versioned schema of this representation of an
    // object. Servers should convert recognized schemas to the latest internal
    // value, and may reject unrecognized values.
    // +optional
    APIVersion string `json:"apiVersion,omitempty"`
}
```

有了这个，Go中的pod声明如下所示：

```
// Pod is a collection of containers that can run on a host. This resource is
// created by clients and scheduled onto hosts.
type Pod struct {
    metav1.TypeMeta `json:",inline"`
    // Standard object's metadata.
    // +optional
    metav1.ObjectMeta `json:"metadata,omitempty"`

    // Specification of the desired behavior of the pod.
    // +optional
    Spec PodSpec `json:"spec,omitempty"`

    // Most recently observed status of the pod.
    // This data may not be up to date.
    // Populated by the system.
    // Read-only.
    // +optional
    Status PodStatus `json:"status,omitempty"`
}
```

如您所见，`TypeMeta`嵌入式。此外，pod类型具有JSON标记，也声明`TypeMeta`为内联。

###### 注意

这个`",inline"`标签实际上与Golang JSON en / decoders一起使用：嵌入式结构自动内联。

这在[YAML en / decoder *go-yaml / yaml中*](http://bit.ly/2ZuPZy2)是不同的，它在非常早期的Kubernetes代码中与JSON并行使用。我们从那时起继承了[内联标签](http://bit.ly/2IUGwcC)，但今天它只是文档而没有任何影响。

在YAML串行foudn *k8s.io/apimachinery/pkg/runtime/serializer/yaml*使用*sigs.k8s.io/yaml*编组和取消编组功能。而这些依次编码和解码YAML `interface{}`，并使用JSON编码器进入Golang API结构并进行解码。

这匹配了一个pod的YAML表示，所有Kubernetes用户都知道：[2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#idm46336865870888)

```
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: example
spec:
  containers:
  - name: hello
    image: debian:latest
    command:
    - /bin/sh
    args:
    - -c
    - echo "hello world"; sleep 10000
```

版本存储在`TypeMeta.APIVersion`，种类中`TypeMeta.Kind`。

##### 核心小组因历史原因而异

荚很早就添加到Kubernetes的许多其他类型都是*核心组的*一部分- 也称为*遗留组* - 由空字符串表示。因此，`apiVersion`只是设置为“ `v1`。”

最终，API组被添加到Kubernetes中，并且以斜杠分隔的组名称被添加到`apiVersion`。在这种情况下`apps`，版本将是`apps/v1`。因此，该`apiVersion`字段实际上是错误的名称; 它存储API组名称和版本字符串。这是出于历史原因，因为`apiVersion`只有核心组才会定义，而这些其他API组都不存在。

运行[“创建和使用客户端”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#rest-client-config)的示例以从群集中获取pod时，请注意客户端返回的pod对象实际上并未设置类型和版本集。`client-go`基于in -ased的应用程序的惯例是这些字段在内存中是空的，只有当它们被封送到JSON或protobuf时，它们才会在线上填充实际值。然而，这是由客户端自动完成的，或者更准确地说，是由版本控制序列化器完成的。

##### 幕后花絮：GO TYPE类型，包，种类和群组名称是如何相关的？

You可能想知道客户端如何知道填写该`TypeMeta`字段的种类和API组。虽然这个问题起初听起来微不足道，但它不是：

- 看起来这种类型只是Go类型名称，可以通过反射从对象派生。这基本上是正确的 - 可能在99％的情况下 - 但也有例外（在[第4章中，](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#ch_crds)您将了解自适应资源，但这不起作用）。
- 看起来该组只是Go包名称（`apps`API组的类型在*k8s.io/api/apps*中声明）。这通常匹配，但并非在所有情况下都匹配：核心组具有空组名称字符串，如我们所见。`rbac.authorization.k8s.io`例如，组的类型位于*k8s.io/api/rbac中*，而不是*k8s.io/api/rbac.authorization.k8s.io中*。

如何填写该`TypeMeta`领域的问题的正确答案涉及方案的概念，将在[“方案”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#scheme)中更详细地讨论。

换句话说，`client-go`基于应用程序检查Golang类型的对象以确定手头的对象。这可能与其他框架不同，例如Operator SDK（请参阅[“Operator SDK”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch06.html#operator-sdk)）。

## ObjectMeta

在除此之外`TypeMeta`，大多数顶级对象都有一个类型的字段`metav1.ObjectMeta`，同样来自*k8s.io/apimachinery/pkg/meta/v1*包：

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

在JSON或YAML这些字段在*元数据*下。例如，对于上一个pod，`metav1.ObjectMeta`存储：

```
metadata:
  namespace: default
  name: example
```

通常，它包含所有元级别信息，如名称，命名空间，资源版本（不要与API组版本混淆），多个时间戳，以及众所周知的标签和注释是其中的一部分`ObjectMeta`。有关字段的深入讨论，请参阅[“类型剖析”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#anatomy-of-CRD-types)`ObjectMeta`。

前面在[“乐观并发”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#optimistic-concurrency)讨论了资源版本。它几乎不会从`client-go`代码中读取或写入。但它是Kubernetes中的一个领域，使整个系统工作。`resourceVersion`是的一部分`ObjectMeta`，因为具有嵌入的每个对象`ObjectMeta`对应于一个键`etcd`的其中`resourceVersion`值起源。

## 规格和状态

最后，差不多了每个顶级对象都有一个`spec`和一个`status`部分。此约定来自Kubernetes API的声明性质：`spec`用户需求，并且`status`是该愿望的结果，通常由系统中的控制器填充。有关Kubernetes中控制器的详细讨论，请参阅[“控制器和操作员”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#ch_controllers-operators)。

系统中的`spec`和`status`惯例只有少数例外- 例如，核心组中的端点，或RBAC对象`ClusterRole`。

# 客户端集

在在[“创建和使用客户端”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#rest-client-config)的介绍性示例中，我们看到它`kubernetes.NewForConfig(config)`为我们提供了一个*客户端集*。客户端集提供对多个API组和资源的客户端的访问。在的情况下`kubernetes.NewForConfig(config)`，从*k8s.io/client-go/kubernetes*，我们送获得的中定义的所有API组和资源*k8s.io/api*。除了少数例外情况，例如`APIServices`（对于聚合的API服务器）和`CustomResourceDefinition`（参见[第4章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#ch_crds)） - Kubernetes API服务器提供的整套资源。

在[第5章中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#ch_autocodegen)，我们将解释如何从API类型（在本例中为*k8s.io/api*）实际生成这些客户端集。使用自定义API的第三方项目不仅使用Kubernetes客户端集。所有客户端集的共同点是：REST配置（例如，返回者`clientcmd.BuildConfigFromFlags("", *kubeconfig)`，如示例中所示）。

该客户端设置*k8s.io/client-go/kubernetes/typed中的*主界面，*Kubernetes*本地资源如下所示：

```
type Interface interface {
    Discovery() discovery.DiscoveryInterface
    AppsV1() appsv1.AppsV1Interface
    AppsV1beta1() appsv1beta1.AppsV1beta1Interface
    AppsV1beta2() appsv1beta2.AppsV1beta2Interface
    AuthenticationV1() authenticationv1.AuthenticationV1Interface
    AuthenticationV1beta1() authenticationv1beta1.AuthenticationV1beta1Interface
    AuthorizationV1() authorizationv1.AuthorizationV1Interface
    AuthorizationV1beta1() authorizationv1beta1.AuthorizationV1beta1Interface

    ...
}
```

在这个界面中曾经有过无版本的方法 - 例如，`Apps() appsv1.AppsV1Interface`但是从基于Kubernetes 1.14的`client-go`11.0 开始，它们被弃用了。如前所述，对应用程序使用的API组的版本非常明确是一种很好的做法。

##### 过去版本化的客户和内部客户

在过去，Kubernetes有所谓的*内部*客户。这些对象称为“内部”的对象使用了通用的内存版本，其中包含与线上版本之间的转换。

希望是从正在使用的实际API版本中抽象出控制器代码，并能够通过单行更改切换到另一个版本。实际上，实现转换的巨大额外复杂性以及此转换代码所需的有关对象语义的知识量导致了这种模式不值得的结论。

此外，客户端和API服务器之间从未进行任何类型的自动协商。即使使用内部类型和客户端，控制器也硬编码到线路上的特定版本。因此，对于使用版本化API类型的客户端和服务器之间的版本偏差，使用内部类型的控制器不再兼容。

在最近的Kubernetes版本中，大量代码被重写以完全摆脱这些内部版本。今天，有没有内部版本*k8s.io/api*可用，也没有客户*k8s.io/client-go*。

一切客户端集还允许访问发现客户端（它将被使用`RESTMappers`;请参阅[“REST映射”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#RESTMapping)和[“从命令行使用API”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#api-cli)）。

背后每个`GroupVersion`方法（例如`AppsV1beta1`），我们找到API组的资源 - 例如：

```
type AppsV1beta1Interface interface {
    RESTClient() rest.Interface
    ControllerRevisionsGetter
    DeploymentsGetter
    StatefulSetsGetter
}
```

同 `RESTClient`作为通用*REST客户端*，每个资源有一个接口，如：

```
// DeploymentsGetter has a method to return a DeploymentInterface.
// A group's client should implement this interface.
type DeploymentsGetter interface {
    Deployments(namespace string) DeploymentInterface
}

// DeploymentInterface has methods to work with Deployment resources.
type DeploymentInterface interface {
    Create(*v1beta1.Deployment) (*v1beta1.Deployment, error)
    Update(*v1beta1.Deployment) (*v1beta1.Deployment, error)
    UpdateStatus(*v1beta1.Deployment) (*v1beta1.Deployment, error)
    Delete(name string, options *v1.DeleteOptions) error
    DeleteCollection(options *v1.DeleteOptions, listOptions v1.ListOptions) error
    Get(name string, options v1.GetOptions) (*v1beta1.Deployment, error)
    List(opts v1.ListOptions) (*v1beta1.DeploymentList, error)
    Watch(opts v1.ListOptions) (watch.Interface, error)
    Patch(name string, pt types.PatchType, data []byte, subresources ...string)
        (result *v1beta1.Deployment, err error)
    DeploymentExpansion
}
```

根据资源的范围 - 即，是集群还是命名空间作用域 - 访问者（此处`DeploymentGetter`）可能有也可能没有`namespace`参数。

在`DeploymentInterface`可以访问资源的所有受支持的动词。其中大多数是不言自明的，但接下来将描述需要额外评论的那些。

## 状态子资源：UpdateStatus

部署有一个所谓的*状态子资源*。这意味着`UpdateStatus`使用后缀的额外HTTP端点`/status`。虽然*/ apis / apps / v1beta1 / namespaces / ns/ deploymentments /name* endpoint上的更新只能更改*部署*的规范，但端点/ *apis / apps / v1beta1 / namespaces / ns/ deployments name/ status*只能更改对象的状态。这对于为规范更新（由人完成）和状态更新（由控制器完成）设置不同的权限非常有用。

通过默认`client-gen`（参见[“client-gen Tags”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#clientgen-tags)）生成`UpdateStatus()`方法。该方法的存在并不能保证资源实际上支持子资源。当我们在[“子资源”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#crd-subresources)使用CRD时，这将非常重要。

## 列表和删除

`DeleteCollection` 允许我们一次删除命名空间的多个对象。该`ListOptions`参数允许我们定义应删除哪些对象使用*字段*或*标签选择器*：

```
type ListOptions struct {
    ...

    // A selector to restrict the list of returned objects by their labels.
    // Defaults to everything.
    // +optional
    LabelSelector string `json:"labelSelector,omitempty"`
    // A selector to restrict the list of returned objects by their fields.
    // Defaults to everything.
    // +optional
    FieldSelector string `json:"fieldSelector,omitempty"`

    ...
}
```

## 手表

`Watch` 给一个事件接口，用于对象的所有更改（添加，删除和更新）。`watch.Interface`从*k8s.io/apimachinery/pkg/watch*返回的*内容*如下所示：

```
// Interface can be implemented by anything that knows how to watch and
// report changes.
type Interface interface {
    // Stops watching. Will close the channel returned by ResultChan(). Releases
    // any resources used by the watch.
    Stop()

    // Returns a chan which will receive all the events. If an error occurs
    // or Stop() is called, this channel will be closed, in which case the
    // watch should be completely cleaned up.
    ResultChan() <-chan Event
}
```

`watch`接口的结果通道返回三种事件：

```
// EventType defines the possible types of events.
type EventType string

const (
    Added    EventType = "ADDED"
    Modified EventType = "MODIFIED"
    Deleted  EventType = "DELETED"
    Error    EventType = "ERROR"
)

// Event represents a single event to a watched resource.
// +k8s:deepcopy-gen=true
type Event struct {
    Type EventType

    // Object is:
    //  * If Type is Added or Modified: the new state of the object.
    //  * If Type is Deleted: the state of the object immediately before
    //    deletion.
    //  * If Type is Error: *api.Status is recommended; other types may
    //    make sense depending on context.
    Object runtime.Object
}
```

虽然直接使用这个界面很诱人，但实际上它实际上不鼓励有利于线人（参见[“线人和缓存”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#informers)）。Informers是此事件接口和带有索引查找的内存缓存的组合。这是迄今为止最常见的手表用例。在引擎线下，首先调用`List`客户端获取所有对象的集合（作为缓存的基线），然后`Watch`更新缓存。它们正确处理错误情况 - 即从网络问题或其他群集问题中恢复。

## 客户扩展

`DeploymentExpansion` 是实际上是一个空接口。它用于添加自定义客户端行为，但现在很难用于Kubernetes。相反，客户端生成器允许我们以声明方式添加自定义方法（请参阅[“client-gen标签”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#clientgen-tags)）。

再次注意，所有的这些方法`DeploymentInterface`既不期望在有效信息`TypeMeta`的字段`Kind`和`APIVersion`，也没有设定这些字段`Get()`和`List()`（见[“TypeMeta”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#TypeMeta)）。这些字段仅在线路上填充实际值。

## 客户选项

它值得看看我们在创建客户端集时可以设置的不同选项。在[“版本控制和兼容性”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#versioning-capability)之前的注释中，我们看到我们可以切换到原生Kubernetes类型的protobuf线格式。Protobuf比JSON（空间和客户端和服务器的CPU负载）更有效，因此更受欢迎。

出于调试目的和指标的可读性，区分访问API服务器的不同客户端通常很有帮助。至这样做，我们可以在REST配置中设置*用户代理*字段。默认值为`binary/version (os/arch) kubernetes/commit`; 例如，`kubectl`将使用像这样的用户代理`kubectl/v1.14.0 (darwin/amd64) kubernetes/d654b49`。如果该模式不足以进行设置，则可以像这样定制：

```
cfg, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
cfg.AcceptContentTypes = "application/vnd.kubernetes.protobuf,application/json"
cfg.UserAgent = fmt.Sprintf(
    "book-example/v1.0 (%s/%s) kubernetes/v1.0",
    runtime.GOOS, runtime.GOARCH
)
clientset, err := kubernetes.NewForConfig(cfg)
```

其他REST配置中经常被覆盖的值是客户端*速率限制*和*超时的值*：

```
// Config holds the common attributes that can be passed to a Kubernetes
// client on initialization.
type Config struct {
    ...

    // QPS indicates the maximum QPS to the master from this client.
    // If it's zero, the created RESTClient will use DefaultQPS: 5
    QPS float32

    // Maximum burst for throttle.
    // If it's zero, the created RESTClient will use DefaultBurst: 10.
    Burst int

    // The maximum length of time to wait before giving up on a server request.
    // A value of zero means no timeout.
    Timeout time.Duration

    ...
}
```

该`QPS`值默认为`5`请求 每秒，有一阵爆冷`10`。

该timeout没有默认值，至少在客户端REST配置中没有。默认情况下，Kubernetes API服务器将在60秒后超时不是*长时间运行*请求的每个请求。长时间运行的请求可以是监视请求，也可以是对*/ exec*，*/ portforward*或*/ proxy*等子资源的无限请求。

##### 优雅的关机和对连接错误的适应能力

要求分为长期运行和非长期运行。手表是长时间运行，同时`GET`，`LIST`，`UPDATE`等都是非长期运行。许多子资源（例如，用于日志流，执行，端口转发）也是长期运行的。

重新启动Kubernetes API服务器时（例如，在更新期间），它会等待最多60秒才能正常关闭。在此期间，它完成非长时间运行的请求，然后终止。当它终止时，长时间运行的请求（如正在进行的监视连接）将被切断。

非长时间运行的请求无论如何都会受到60秒的限制（然后它们会超时）。因此，从客户端的角度来看，关闭是优雅的。

通常，应始终为不成功的请求准备应用程序代码，并且应该以对应用程序不致命的方式进行响应。在分布式系统的世界中，那些连接错误是正常的，没什么好担心的。但需要特别注意小心处理错误情况并从中恢复。

错误处理尤为重要手表。手表长时间运行，但它们可能随时出现故障。下一节中描述的告密者提供了一个围绕监视的弹性实现，并且可以优雅地处理错误 - 也就是说，它们可以通过新连接从断开连接中恢复。应用程序代码通常不会注意到。

# 告密者和缓存

该[“客户端集”中的](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#clientsets)客户端接口包括`Watch`动词，动词提供对对象的更改（添加，删除，更新）作出反应的事件接口。Informers为最常见的手表用例提供了更高级别的编程接口：内存中缓存和按名称或内存中的其他属性快速，索引查找对象。

每次需要对象时访问API服务器的控制器都会在系统上产生高负载。使用informers进行内存中缓存是解决此问题的方法。此外，线人几乎可以实时响应对象的变化，而不需要轮询请求。

[图3-5](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#informers-figure)显示了线人的概念模式; 具体来说，他们：

- 从API服务器获取输入作为事件。
- 提供一个类似客户端的接口，`Lister`用于从内存缓存中获取和列出对象。
- 注册事件处理程序以进行添加，删除和更新。
- 实现内存缓存使用*商店*。

![线人](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0305.png)

###### 图3-5。线人

线人还有高级错误行为：当长时间运行的监视连接发生故障时，它们会通过尝试另一个监视请求从中恢复，从而在不丢失任何事件的情况下拾取事件流。如果中断很长，并且API服务器丢失了事件，因为`etcd`在新的监视请求成功之前将它们从数据库中清除，则告密者将 重新安置所有物体。

在*relists旁边*，有一个可配置的重新*同步周期，*用于内存缓存和业务逻辑之间的协调：每次经过此周期后，将为所有对象调用已注册的事件处理程序。常见值以分钟为单位（例如，10或30分钟）。

###### 警告

重新同步纯粹是在内存中，*不会触发对服务器的调用*。这曾经是不同的，但[最终被改变了，](http://bit.ly/2FmeMge)因为手表机制的错误行为已经得到了足够的改进，使得不需要重复。

所有这些高级且经过实战验证的错误行为是使用informer而不是`Watch()`直接使用客户端方法推出自定义逻辑的一个很好的理由。Informer在Kubernetes本身随处可见，是Kubernetes API设计中的主要架构概念之一。

虽然告密者比轮询更受欢迎，但它们会在API服务器上产生负载。一个二进制文件应该只为每个GroupVersionResource实例化一个informer。至通过使用*共享的informer工厂*，我们可以实现一个告密*者*。

共享的informer工厂允许在应用程序中为相同的资源共享信息。换句话说，不同的控制回路可以使用与引擎盖下的API服务器相同的手表连接。对于例如，`kube-controller-manager`其中一个主要的Kubernetes集群组件（参见[“API服务器”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#api-server)）具有更大的两位数控制器。但是对于每个资源（例如，pod），在该过程中只有一个告密者。

###### 小费

始终使用共享的informer工厂来实例化informers。不要尝试手动实例化informers。开销很小，并且不使用共享信息器的非平凡控制器二进制文件可能正在为某个地方的同一资源打开多个监视连接。

从一个开始REST配置（请参阅[“创建和使用客户端”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#rest-client-config)），使用客户端集创建共享的informer工厂很容易。告密者由代码生成器生成，并作为`client-go`k8s.io/client-go/informers中标准Kubernetes资源的*一部分提供*：

```
import (
    ...
    "k8s.io/client-go/informers"
)
...
clientset, err := kubernetes.NewForConfig(config)
informerFactory := informers.NewSharedInformerFactory(clientset, time.Second*30)
podInformer := informerFactory.Core().V1().Pods()
podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: func(new interface{}) {...},
    UpdateFunc: func(old, new interface{}) {...},
    DeleteFunc: func(obj interface{}) {...},
})
informerFactory.Start(wait.NeverStop)
informerFactory.WaitForCacheSync(wait.NeverStop)
pod, err := podInformer.Lister().Pods("programming-kubernetes").Get("client-go")
```

该示例显示了如何获取pod的共享informer。

你可以看到informers允许添加三种情况的事件处理程序*添加*，*更新*和*删除*。这些通常用于触发控制器的业务逻辑- 即再次处理某个对象（请参阅[“控制器和操作员”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#ch_controllers-operators)）。这些处理程序通常只是将修改后的对象添加到工作队列中。

##### 其他事件处理程序和内部存储更新逻辑

不要将这些处理程序与informer内部存储库的更新逻辑（可通过示例最后一行中的lister访问）混淆。告密者将始终更新其商店，但刚刚描述的其他事件处理程序是可选的，并且意味着由线人的消费者使用。

另请注意，可以添加许多事件处理程序。存在整个共享信息工厂概念只是因为这在具有许多控制循环的控制器二进制文件中非常常见，每个控制循环都安装事件处理程序以将对象添加到它们自己的工作队列中。

注册处理程序后，必须启动共享的线程工厂。在引擎盖下有Go例程来执行对API服务器的实际调用。该`Start`方法（使用停止通道来控制生命周期）启动这些Go例程，该`WaitForCacheSync()`方法使代码等待对`List`客户端的第一次调用完成。如果控制器逻辑要求填充缓存，则此`WaitForCacheSync`调用至关重要。

通常，幕后手表的事件界面导致一定的滞后。在具有适当容量规划的设置中，这种滞后并不是很大。当然，使用指标衡量这种滞后是一种很好的做法。但是滞后存在，所以应用程序逻辑的构建方式使得滞后不会损害代码的行为。

###### 警告

告密者的滞后可导致控制器`client-go`直接在API服务器上进行的更改与告密者所知的世界状态之间的竞争。

如果控制器更改了对象，则同一进程中的告密者必须等到相应的事件到达，然后更新内存中的存储。此过程不是即时的，并且可能在前一个更改变为可见之前通过另一个触发器启动另一个控制器工作循环运行。

在该示例中，重新同步间隔30秒导致发送到注册的完整事件集`UpdateFunc`，使得控制器逻辑能够将其状态与API服务器的状态协调。通过比较该`ObjectMeta.resourceVersion`字段，可以区分真实更新和重新同步。

###### 小费

选择良好的重新同步间隔取决于上下文。例如，30秒非常短。在许多情况下，几分钟，甚至30分钟，是一个不错的选择。在最坏的情况下，30分钟意味着需要30分钟才能通过协调修复代码中的错误（例如，由于错误处理错误导致的信号丢失）。

另请注意，示例调用中的最后一行`Get("client-go")`纯粹是在内存中; 无法访问API服务器。无法直接修改内存存储中的对象。相反，客户端集必须用于对资源的任何写访问。然后，线程将从API服务器获取事件并更新其内存存储。

##### 永远不要改变告密者的对象

记住从列表传递给事件处理程序的任何对象都是由线人拥有的，这一点非常重要。如果你以任何方式改变它，你就冒着引入难以调试的风险将一致性问题缓存到您的应用程序中。在更改对象之前，请始终执行深层复制（请参阅[“Go中的Kubernetes对象”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#kube-objects)）。

一般来说：在变异对象之前，总是问自己谁拥有这个对象或其中的数据结构。根据经验：

- 告密者和姐妹们拥有他们返回的物品。因此，消费者必须在变异之前进行深层复制。
- 客户端返回呼叫者拥有的新鲜对象。
- 转换返回共享对象。如果调用者拥有输入对象，则它不拥有输出。

`NewSharedInformerFactory`示例中的informer构造函数将资源的所有对象缓存在商店的所有名称空间中。如果这对于应用程序来说太多了，那么有一个具有更大灵活性的替代构造函数：

```
// NewFilteredSharedInformerFactory constructs a new instance of
// sharedInformerFactory. Listers obtained via this sharedInformerFactory will be
// subject to the same filters as specified here.
func NewFilteredSharedInformerFactory(
    client versioned.Interface, defaultResync time.Duration,
    namespace string,
    tweakListOptions internalinterfaces.TweakListOptionsFunc
) SharedInformerFactor

type TweakListOptionsFunc func(*v1.ListOptions)
```

它使我们能够指定一个命名空间，并传递一个`TweakListOptionsFunc`，这可能发生变异的`ListOptions`用来列出并观看使用对象结构`List`和`Watch`客户端的调用。例如，它可用于设置*标签*或*字段选择器*。

告密者是控制器的基石之一。在[第6章中，](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch06.html#ch_operator-solutions)我们将看到`client-go`基于典型的控制器的外观。在客户端和线人之后，第三个主要构建块是工作队列。我们现在来看看吧。

## 工作队列

一个工作队列是一种数据结构。您可以按队列预定义的顺序添加元素并从队列中取出元素。形式上，这种队列称为*优先级队列*。`client-go`为在[*k8s.io/client-go/util/workqueue*](http://bit.ly/2IV0JPz)中构建控制器提供了一个强大的实现。

更准确地说，该包装包含许多用于不同目的的变体。所有变体实现的基本接口如下所示：

```
type Interface interface {
    Add(item interface{})
    Len() int
    Get() (item interface{}, shutdown bool)
    Done(item interface{})
    ShutDown()
    ShuttingDown() bool
}
```

这里`Add(item)`添加一个项目，`Len()`给出长度，并`Get()`返回一个具有最高优先级的项目（并且它会阻塞直到一个可用）。当控制器完成处理后，返回的每个项目都`Get()`需要`Done(item)`调用。同时，重复`Add(item)`只会将项目标记为脏，以便在`Done(item)`被调用时对其进行读取。

以下队列类型派生自此通用接口：

- `DelayingInterface`可以在以后添加项目。这样可以更容易在故障后重新排队项目而不会在热循环中结束：

  ```
  type DelayingInterface interface {
      Interface
      // AddAfter adds an item to the workqueue after the
      // indicated duration has passed.
      AddAfter(item interface{}, duration time.Duration)
  }
  ```

- `RateLimitingInterface`速率限制项目被添加到队列中。它扩展了`DelayingInterface:`

  ```
  type RateLimitingInterface interface {
      DelayingInterface
  
      // AddRateLimited adds an item to the workqueue after the rate
      // limiter says it's OK.
      AddRateLimited(item interface{})
  
      // Forget indicates that an item is finished being retried.
      // It doesn't matter whether it's for perm failing or success;
      // we'll stop the rate limiter from tracking it. This only clears
      // the `rateLimiter`; you still have to call `Done` on the queue.
      Forget(item interface{})
  
      // NumRequeues returns back how many times the item was requeued.
      NumRequeues(item interface{}) int
  }
  ```

  这里最有趣的是`Forget(item)`方法：它重置给定项目的后退。通常，在成功处理项目时将调用它。

  速率限制算法可以传递给构造函数`NewRateLimitingQueue`。在同一个包中定义了几个速率限制器，例如`BucketRateLimiter`，the `ItemExponentialFailureRateLimiter`，the `ItemFastSlowRateLimiter`和the `MaxOfRateLimiter`。有关更多详细信息，请参阅包文档。大多数控制器只使用这些`DefaultControllerRateLimiter() *RateLimiter`功能，这给出了：

  - 指数退避从5毫秒开始并上升到1,000秒，使每个错误的延迟加倍
  - 最大速率为每秒10项，爆发100项

根据上下文，您可能希望自定义值。对于某些控制器应用，每个项目最大后退1,000秒是很多的。

# API机械深度

该API Machinery存储库实现了Kubernetes类型系统的基础知识。但究竟什么是这种类型的系统呢？什么是开始的类型？

该术语*类型*实际上并不存在于API Machinery的术语中。相反，它指的是*种类*。

## 种

种分为API组和版本，正如我们在[“API术语”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#terminology)已经看到的那样。因此，API Machinery存储库中的核心术语是GroupVersionKind，简称*GVK*。

在Go中，每个GVK对应一个Go类型。相反，Go类型可以属于多个GVK。

种类没有正式映射到HTTP路径一对一。许多种类都有HTTP REST端点，用于访问给定类型的对象。但是也有没有任何HTTP端点的种类（例如，[*admission.k8s.io/*](http://bit.ly/2XJXBQD) v1beta1.AdmissionReview，用于调用webhook）。还有许多端点返回的类型 - 例如，[*meta.k8s.io / v1.Status*](http://bit.ly/31Ktjvz)，由所有端点返回以报告非对象状态，如错误。

通过约定，种类在[CamelCase中](http://bit.ly/31IqMSC)被格式化为[单词](http://bit.ly/31IqMSC)并且通常是单数的。根据具体情况，它们的具体格式不同。对于CustomResourceDefinition类型，它必须是DNS路径标签（RFC 1035）。

## 资源

在与“种类”并行，正如我们在[“API术语”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#terminology)看到的那样，存在*资源*的概念。资源再次分组和版本化，导致术语*GroupVersionResource*或*简称GVR*。

每个GVR对应一个HTTP（基本）路径。GVR用于标识Kubernetes API的REST端点。例如，GVR *apps / v1.deployments*映射到*/ apis / apps / v1 / namespaces / namespace/ deploymentments*。

客户端库使用此映射来构造访问GVR的HTTP路径。

##### 了解资源是命名空间还是群集范围

You必须知道GVR是命名空间还是群集范围才能知道HTTP路径。例如，部署是命名空间，因此将命名空间作为其HTTP路径的一部分。其他GVR，例如*rbac.authorization.k8s.io/v1.clusterroles*，是集群范围的; 例如，可以在*apis / rbac.authorization.k8s.io / v1 / clusterroles中*访问集群角色。

通过惯例，资源是小写和复数，通常对应于并行类型的多个单词。它们必须符合DNS路径标签格式（RFC 1025）。由于资源直接映射到HTTP路径，因此这并不奇怪。

## REST映射

该将GVK映射到GVR称为*REST映射*。

A `RESTMapper`是[Golang接口](http://bit.ly/2Y7wYS8)，使我们能够为GVK请求GVR：

```
RESTMapping(gk schema.GroupKind, versions ...string) (*RESTMapping, error)
```

`RESTMapping`右侧的类型如下所示：

```
type RESTMapping struct {
    // Resource is the GroupVersionResource (location) for this endpoint.
    Resource schema.GroupVersionResource.

    // GroupVersionKind is the GroupVersionKind (data format) to submit
    // to this endpoint.
    GroupVersionKind schema.GroupVersionKind

    // Scope contains the information needed to deal with REST Resources
    // that are in a resource hierarchy.
    Scope RESTScope
}
```

此外，a `RESTMapper`还提供了许多便利功能：

```
// KindFor takes a partial resource and returns the single match.
// Returns an error if there are multiple matches.
KindFor(resource schema.GroupVersionResource) (schema.GroupVersionKind, error)

// KindsFor takes a partial resource and returns the list of potential
// kinds in priority order.
KindsFor(resource schema.GroupVersionResource) ([]schema.GroupVersionKind, error)

// ResourceFor takes a partial resource and returns the single match.
// Returns an error if there are multiple matches.
ResourceFor(input schema.GroupVersionResource) (schema.GroupVersionResource, error)

// ResourcesFor takes a partial resource and returns the list of potential
// resource in priority order.
ResourcesFor(input schema.GroupVersionResource) ([]schema.GroupVersionResource, error)

// RESTMappings returns all resource mappings for the provided group kind
// if no version search is provided. Otherwise identifies a preferred resource
// mapping for the provided version(s).
RESTMappings(gk schema.GroupKind, versions ...string) ([]*RESTMapping, error)
```

这里，部分GVR意味着并非所有字段都被设置。例如，假设您输入`**kubectl get pods**`。在这种情况下，组和版本丢失。一个`RESTMapper`有足够的信息可能仍然设法将其映射到`v1 Pods`的那种。

对于前面的部署示例，`RESTMapper`了解部署（更多关于这意味着什么）将把[*apps / v1.Deployment*](http://bit.ly/2IujaLU)映射到*apps / v1.deployments*作为命名空间资源。

`RESTMapper`接口有多种不同的实现方式。客户端应用程序最重要的一个是基于发现的*k8s.io/client-go/restmapper*[`DeferredDiscoveryRESTMapper`](http://bit.ly/2XroxUq)包：它使用来自Kubernetes API服务器的发现信息来动态构建REST映射。它还可以与非核心资源（如自定义资源）一起使用。

## 方案

该我们希望在座的Kubernetes类型系统的情况下最终核心概念是[*计划*](http://bit.ly/2N1PGJB)在包*k8s.io/apimachinery/pkg/runtime*。

一个方案将Golang世界与GVKs独立于实现的世界联系起来。方案的主要特征是将Golang类型映射到可能的GVK：

```
func (s *Scheme) ObjectKinds(obj Object) ([]schema.GroupVersionKind, bool, error)
```

正如我们在看到[“围棋Kubernetes对象”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#kube-objects)，一个objec t可返回其组，通过T个样，他`GetObjectKind() schema.ObjectKind`的方法。但是，这些值在大多数时间都是空的，因此对于识别而言是无用的。

相反，该方案采用给定对象的Golang类型反射并将其映射到Golang类型的已注册GVK。当然，要使用它，Golang类型必须像这样注册到方案中：

```
scheme.AddKnownTypes(schema.GroupVersionKind{"", "v1", "Pod"}, &Pod{})
```

该方案不仅用于注册Golang类型及其GVK，还用于存储转换函数和默认值的列表（参见[图3-6](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#scheme-spider)）。我们将在[第8章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#ch_custom-api-servers)中更详细地讨论转换和违约者。它也是实现编码器和解码器的数据源。

![该方案将Golang数据类型与GVK，转换和违约者联系起来](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0306.png)

###### 图3-6。该方案将Golang数据类型与GVK，转换和违约者联系起来

对于Kubernetes核心类型，在*k8s.io/client-go/kubernetes/scheme*包[中的`client-go`客户端集中](http://bit.ly/2FkXDn2)有一个[预定义的方案](http://bit.ly/2FkXDn2)，所有类型都已预先注册。实际上，每个客户端集都由 `client-gen`代码生成器（参见[第5章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#ch_autocodegen)）具有子包，`scheme`其中包含客户端集中所有组和版本中的所有类型。

同该计划我们深入探讨了API Machinery的概念。如果你只记得关于这些概念的一件事，那就让它[如图3-7所示](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#scheme-restmapper-http-path)。

![从Golang类型到GVK到GVR再到HTTP路径 - 简而言之，API机制](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0307.png)

###### 图3-7。从Golang类型到GVK到GVR再到HTTP路径 - 简而言之

# Vendoring

我们本章在看到*k8s.io/client-go*，*k8s.io/api*和*k8s.io/apimachinery*是中央在Golang Kubernetes编程。Golang采用*vendoring*包括在第三方应用程序的源代码库，这些库。

Vendoring是Golang社区的一个移动目标。在撰写本文时，几种销售工具很常见，例如*godeps*，*dep*和*glide*。与此同时，Go 1.12正在获得对Go模块的支持，这些模块可能会成为未来Go社区的标准销售方法，但目前尚未在Kubernetes生态系统中做好准备。

现在大多数项目都使用`dep`或`glide`。Kubernetes本身在*github.com/kubernetes/kubernetes*中跳转到了1.15开发周期的Go模块。以下注释与所有这些销售工具相关。

每个*k8s.io/**存储库中受支持的依赖项版本的真实来源是随附的*Godeps / Godeps.json*文件。重要的是要强调任何其他依赖项选择都可能破坏库的功能。

有关*k8s.io/client-go，k8s.io* / *api*和*k8s.io/apimachinery*的已发布标记以及哪些标记彼此兼容，请参阅[“客户端库”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#client-go)。

## 滑行

项目using `glide`可以使用它在任何依赖项更改时读取*Godeps / Godeps.json*文件的能力。事实证明这非常可靠：开发人员只需要声明正确的*k8s.io/client-go*版本，并`glide`选择正确版本的*k8s.io/apimachinery，k8s.io* / *api*和其他依赖项。

对于GitHub上的一些项目，*glide.yaml*文件可能如下所示：

```
package: github.com/book/example
import:
- package: k8s.io/client-go
  version: v10.0.0
...
```

有了它，`glide install -v`将*k8s.io/client-go*及其依赖项下载到本地*供应商/*包中。这里，`-v`意味着从销售的库中删除*供应商/*包。这是我们的目的所必需的。

如果您`client-go`通过编辑*glide.yaml*更新到新版本，`glide update -v`将再次以正确的版本下载新的依赖项。

## DEP

`dep`通常被认为比...更强大和更先进`glide`。长期以来它被视为继任者`glide`的生态系统，似乎注定要*在*围棋vendoring工具。在撰写本文时，其未来尚不清楚，Go模块似乎是前进的道路。

在上下文中`client-go`，了解以下几个限制非常重要`dep`：

- `dep`在第一次运行时会读取*Godeps / Godeps.json*`dep init`。
- `dep`在以后的调用中不会读取*Godeps / Godeps.json*`dep ensure -update`。

这意味着在*Godep.toml中*更新版本`client-go`时，依赖关系的解析可能是错误的。这很不幸，因为它要求开发人员显式地并且通常手动声明*所有*依赖项。`client-go`

一个工作且一致的*Godep.toml*文件如下所示：

```
[[constraint]]
  name = "k8s.io/api"
  version = "kubernetes-1.13.0"

[[constraint]]
  name = "k8s.io/apimachinery"
  version = "kubernetes-1.13.0"

[[constraint]]
  name = "k8s.io/client-go"
  version = "10.0.0"

[prune]
  go-tests = true
  unused-packages = true

# the following overrides are necessary to enforce
# the given version, even though our
# code does not import the packages directly.
[[override]]
  name = "k8s.io/api"
  version = "kubernetes-1.13.0"

[[override]]
  name = "k8s.io/apimachinery"
  version = "kubernetes-1.13.0"

[[override]]
  name = "k8s.io/client-go"
  version = "10.0.0"
```

###### 警告

不不仅*Gopkg.toml*宣布明确的版本都*k8s.io/apimachinery*和*k8s.io/api*，它也有它们覆盖。在没有从这两个存储库中显式导入包的情况下启动项目时，这是必需的。在这种情况下，没有这些覆盖`dep`会忽略开头的约束，开发人员会从一开始就得到错误的依赖。

即使这里显示的*Gopkg.toml*文件在技术上也不正确，因为它不完整，因为它没有声明*对*所需的*所有*其他库的依赖性`client-go`。在过去，上游图书馆打破了编译`client-go`。因此，如果您使用`dep`依赖项管理，请为此做好准备。

## 去模块

去模块是Golang中依赖管理的未来。它们在Go 1.11中得到[初步支持](http://bit.ly/2FmBp3Y)，并进一步稳定在1.12。许多命令，如`go run`和`go get`，通过设置`GO111MODULE=on`环境变量来使用Go模块。在Go 1.13中，这将是默认设置。

Go模块由项目根目录中的*go.mod*文件驱动。以下是[第8章中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#ch_custom-api-servers)*github.com/programming-kubernetes/pizza-apiserver*项目的*go.mod*文件的摘录：

```
模块github.com/programming-kubernetes/pizza-apiserver

要求（
    ...
    k8s.io/api v0.0.0-20190222213804-5cb15d344471 //间接
    k8s.io/apimachinery v0.0.0-20190221213512-86fb29eff628
    k8s.io/apiserver v0.0.0-20190319190228-a4358799e4fe
    k8s.io/client-go v2.0.0-alpha.0.0.20190307161346-7621a5ebb88b +不兼容
    k8s.io/klog v0.2.1-0.20190311220638-291f19f84ceb
    k8s.io/kube-openapi v0.0.0-20190320154901-c59034cc13d5 //间接
    k8s.io/utils v0.0.0-20190308190857-21c4ce38f2a7 //间接
    sigs.k8s.io/yaml v1.1.0 //间接
)
```

`client-go` `v11.0.0`匹配的Kubernetes 1.14及更早版本没有对Go模块的明确支持。仍然可以将Go模块与Kubernetes库一起使用，如前面的示例所示。

但是，只要`client-go`和其他Kubernetes存储库不发送*go.mod*文件（至少在Kubernetes 1.15之前），必须手动选择正确的版本。也就是说，你需要匹配的依赖的修订所有依赖的完整列表*Godeps / Godeps.json*在`client-go`。

另请注意前一个示例中不太易读的修订版。它们是从现有标签派生的伪版本，或者`v0.0.0`如果没有标签则用作前缀。更糟糕的是，您可以在该文件中引用标记版本，但Go模块命令将替换下次运行时使用伪版本的命令。

通过`client-go` `v12.0.0`匹配Kubernetes 1.15，我们发布了一个*go.mod*文件并弃用了对所有其他销售工具的支持（参见[相应的提案文档](http://bit.ly/2IZ9MPg)）。随附的*go.mod*文件包含所有依赖项，您的项目*go.mod*文件不再需要手动列出所有传递依赖项。在以后的版本中，也可能会更改标记方案以修复丑陋的伪修订并用适当的替换它们semver标签。但在撰写本文时，尚未完全实施或决定。

# 摘要

在本章中，我们的重点是Go中的Kubernetes编程接口。我们讨论了访问众所周知的核心类型的Kubernetes API，即每个Kubernetes集群附带的API对象。

有了这个，我们已经介绍了Kubernetes API的基础知识及其在Go中的表示。现在，我们已准备好继续讨论自定义资源这一主题，这是运营商的支柱之一。

[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#idm46336866123400-marker)请参阅[ “API术语”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#terminology)。

[2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#idm46336865870888-marker) `kubectl` `explain` `pod`允许您在API服务器中查询对象的模式，包括字段文档。