# 第5章自动生成代码

在在本章中，您将学习如何在Go项目中使用Kubernetes代码生成器以自然的方式编写自定义资源。代码生成器在本地Kubernetes资源的实现中经常使用，我们将在这里使用相同的生成器。

# 为什么代码生成

Go是一种简单的语言设计。它缺乏更高级甚至类元编程的机制，以通用（即，类型无关）的方式表达不同数据类型的算法。“Go way”是使用外部代码生成。

在Kubernetes开发过程的早期阶段，随着更多资源被添加到系统中，必须重写越来越多的代码。代码生成使得维护此代码变得更加容易。很早就开始了[Gengo库](http://bit.ly/2L9kwNJ)，最终，基于[*Gengo*](http://bit.ly/2Kw8I8U)，[*k8s.io / code-generator*](http://bit.ly/2Kw8I8U)被开发为外部可用的发生器集合。我们将在以下部分中使用这些生成器进行CR。

# 召唤发电机

通常，代码生成器在每个控制器项目中以大致相同的方式调用。只有包，组名和API版本不同。调用脚本*k8s.io/code-generator/generate-groups.sh*或像*hack / update-codegen.sh*这样的bash脚本是从构建系统向CR Go类型添加代码生成的最简单方法（参见[本书的GitHub存储库）](http://bit.ly/2J0s2YL)）。

请注意，由于非常特殊的要求和历史原因，某些项目直接调用生成器二进制文件。对于为CR构建控制器的用例，从*k8s.io/code-generator*存储库调用*generate-groups.sh*脚本要容易得多：

```
$ vendor / k8s.io / code-generator / generate-groups.sh all \
    github.com/programming-kubernetes/cnat/cnat-client-go/pkg/generated
    github.com/programming-kubernetes/cnat/cnat-client-go/pkg/apis \
    cnat：v1alpha1 \
    --output-base "${GOPATH}/src" \
    --go-header-file"hack/boilerplate.go.txt"
```

在这里，`all`打电话的方式 CR的所有四个标准代码生成器：

- `deepcopy-gen`

  生成`func` `(t *T)` `DeepCopy()` `*T`和`func` `(t *T)` `DeepCopyInto(*T)`方法。

- `client-gen`

  创建类型化的客户端集。

- `informer-gen`

  为CR创建提供者，提供基于事件的界面以响应服务器上CR的更改。

- `lister-gen`

  为CR提供lister，为其提供只读缓存层`GET`和`LIST`请求。

最后两个是构建控制器的基础（参见[“控制器和操作员”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#ch_controllers-operators)）。这四个代码生成器构成了使用Kubernetes上游控制器使用的相同机制和包构建功能齐全的生产就绪控制器的强大基础。

###### 注意

*k8s.io/code-generator中*还有一些生成器，主要用于其他上下文。例如，如果您构建自己的聚合API服务器（请参阅[第8章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#ch_custom-api-servers)），除了版本化类型之外，您还将使用内部类型，并且必须定义默认函数。然后，您可以通过从*k8s.io/code-generator*调用[*generate-internal-groups.sh*](http://bit.ly/2L9kSE3)脚本来访问这两个生成器，它们将变得相关：

- `conversion-gen`

  创建用于转换的函数 内部和外部类型。

- `defaulter-gen`

  处理违约某些领域的问题。

现在让我们详细查看参数`generate-groups.sh`：

- 第二个参数是生成的客户端，列表和告密者的目标包名称。
- 第三个参数是API组的基础包。
- 第四个参数是以空格分隔的API组列表及其版本。
- `--output-base` 作为标志传递给所有生成器，以定义找到给定包的基本目录。
- `--go-header-file` 使我们能够将版权标题放入生成的代码中。

某些生成器，例如`deepcopy-gen`，直接在API组包内创建文件。这些文件遵循标准命名方案和*zz_generated。*前缀使得很容易将它们从版本控制系统中排除（例如，通过*.gitignore*文件），尽管大多数项目决定检查生成的文件，因为围绕代码生成器的Go工具没有很好地开发。[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#idm46336860111320)

如果项目遵循[*k8s.io/sample-controller*](http://bit.ly/2UppsTN)的模式- 这`sample-controller`是一个蓝图项目，复制由Kubernetes本身内置的许多控制器建立的模式 - 那么代码生成开始于：

```
$ 黑客/ update-codegen.sh
```

将`cnat`在本例`sample-controller+client-go`中的变体[“下面的示例控制器”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch06.html#cnat-client-go)去这条路线。

###### 小费

通常，除了[`hack/update-codegen.sh`](http://bit.ly/2J0s2YL)脚本之外，还有一个名为的第二个脚本[`hack/verify-codegen.sh`](http://bit.ly/2IXUWsy)。

此脚本调用`hack/update-codegen.sh`脚本并检查是否有任何更改，如果任何生成的文件不是最新的，则它将以非零返回码终止。

这在持续集成（CI）脚本中非常有用：如果开发人员意外修改了文件或者文件刚刚过时，CI会注意到并抱怨。

# 使用标记控制生成器

而一些代码生成器行为是通过前面描述的命令行标志控制的（特别是要处理的包），通过Go文件中的*标签*控制更多属性。标签是一种特殊格式的Go评论，格式如下：

```
// +some-tag
// +some-other-tag=value
```

有两种标签：

- `package`名为*doc.go*的文件中位于行上方的全局标记
- 类型声明上方的本地标记（例如，在结构定义之上）

根据相关标签，评论的位置可能很重要。

##### 准确地关注示例（包括注释块）

有许多标记必须位于类型（或全局标记的包行）上方的注释中，而其他标记必须与类型（或包行）分开，并且它们之间至少有一个空行。例如：

```
// +second-comment-block-tag

// +first-comment-block-tag
type Foo struct {
}
```

这种区别的原因是历史性的：Kubernetes中的API文档生成器用于不了解代码生成标记，而是仅导出第一个注释块。因此，该块中的标记将出现在API HTML文档中。

代码生成器标记解析逻辑并不总是非常一致，并且错误处理通常远非完美。虽然每个版本都有所改进，但要准备好非常精确地遵循现有示例 - 例如，空行可能很重要。

## 全球标签

全局标签被写入包的*doc.go*。典型的*pkg / apis / / group/ versiondoc.go*文件如下所示：

```
// +k8s:deepcopy-gen=package

// Package v1 is the v1alpha1 version of the API.
// +groupName=cnat.programming-kubernetes.info
package v1alpha1
```

该文件的第一行告诉`deepcopy-gen`默认情况下为该包中的每个类型创建深层复制方法。如果您的类型不需要深层复制，不需要，甚至不可能，您可以使用本地标签选择使用深层复制`// +k8s:deepcopy-gen=false`。如果您不启用软件包范围的深层复制，则必须通过以下方式为每个所需类型选择深层复制`// +k8s:deepcopy-gen=true`。

第二个标记`// +groupName=example.com`定义了完全限定的API组名称。如果Go父包名称与组名称不匹配，则此标记是必需的。

此处显示的文件实际上来自[`cnat client-go`示例*pkg / apis / cnat / v1alpha1 / doc.go*文件](http://bit.ly/2L6M9ad)（请参阅[“以下示例控制器”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch06.html#cnat-client-go)）。那里`cnat`是父包，但是`cnat.programming-kubernetes.info`是组名。

使用`// +groupName`标记，客户端生成器（请参阅[“通过client-gen创建的类型化客户端”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#clientgen-client)）将使用正确的HTTP路径*/apis/foo.project.example.com*生成客户端。除此之外，`+groupName`还有`+groupGoName`一个定义要使用的自定义Go标识符（用于变量和类型名称）而不是父包名称。例如，默认情况下，生成器将使用大写父包名称进行标识，在我们的示例中为`Cnat`。一个更好的标识符将是`CNAt`“Cloud Native At。” `// +groupGoName=CNAt`我们可以使用它而不是`Cnat`（虽然我们在这个例子中没有这样做 - 我们一直待在那里`Cnat`），以及该`client-gen`结果将如下所示：

```
type Interface interface {
    Discovery() discovery.DiscoveryInterface
    CNatV1() atv1alpha1.CNatV1alpha1Interface
}
```

## 本地标签

本地标签直接写在API类型之上或其上方的第二个注释块中。以下是该[示例](http://bit.ly/31QosJw)的*types.go*文件中的主要类型：[`cnat`](http://bit.ly/31QosJw)

```
// AtSpec defines the desired state of At
type AtSpec struct {
    // Schedule is the desired time the command is supposed to be executed.
    // Note: the format used here is UTC time https://www.utctime.net
    Schedule string `json:"schedule,omitempty"`
    // Command is the desired command (executed in a Bash shell) to be executed.
    Command string `json:"command,omitempty"`
    // Important: Run "make" to regenerate code after modifying this file
}

// AtStatus defines the observed state of At
type AtStatus struct {
    // Phase represents the state of the schedule: until the command is executed
    // it is PENDING, afterwards it is DONE.
    Phase string `json:"phase,omitempty"`
    // Important: Run "make" to regenerate code after modifying this file
}

// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// At runs a command at a given schedule.
type At struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   AtSpec   `json:"spec,omitempty"`
    Status AtStatus `json:"status,omitempty"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// AtList contains a list of At
type AtList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []At `json:"items"`
}
```

在以下部分中，我们将介绍此示例的标记。

###### 小费

在此示例中，API文档位于第一个注释块中，而我们将标记放入第二个注释块中。如果您使用某种工具为此目的提取Go文档注释，这有助于将标记保留在API文档之外。

## deepcopy-gen标签

深度复制方法生成通常默认情况下通过全局`// +k8s:deepcopy-gen=package`标记为所有类型启用（请参阅[“全局标记”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#global-tags)） - 也就是说，可能选择退出。但是，在前面的示例文件（实际上是整个包）中，所有API类型都需要深层复制方法。因此，我们不必在本地选择退出。

如果我们在API类型包中有一个帮助器结构（通常不鼓励保持API包清洁），我们必须禁用深度复制生成。例如：

```
// +k8s:deepcopy-gen=false
//
// Helper is a helper struct, not an API type.
type Helper struct {
    ...
}
```

## runtime.Object和DeepCopyObject

那里 是一个特殊的深拷贝标记，需要更多解释：

```
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
```

在[“Kubernetes Objects in Go”中，](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#kube-objects)我们看到`runtime.Object`s必须实现该`DeepCopyObject() runtime.Object`方法。原因是Kubernetes中的通用代码必须能够创建对象的深层副本。这种方法允许这样做。

##### 历史背景

在1.8之前，该方案（参见[“方案”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#scheme)）也保留了对特定于类型的深拷贝函数的引用，并且它具有基于反射的深拷贝实现。这两种机制都是导致许多重要且难以发现的错误的原因。因此，Kubernetes使用界面中的`DeepCopyObject`方法切换到静态深层复制`runtime.Object`。

该`DeepCopyObject()`方法除了调用生成的`DeepCopy`方法之外什么都不做。后者的签名因类型而异（`DeepCopy()` `*T`取决于`T`）。前者的签名总是`DeepCopyObject()` `runtime.Object`：

```
func (in *T) DeepCopyObject() runtime.Object {
    if c := in.DeepCopy(); c != nil {
        return c
    } else {
        return nil
    }
}
```

将本地标记放在顶级API类型上方以生成此方法。这告诉创建这样一个方法，调用。`//``+k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object``deepcopy-gen``deepcopy-gen``runtime.Object``DeepCopyObject()`

###### 小费

在前面的例子中，两个`At`和`AtList`因为它们被用作是顶级类型`runtime.Object`秒。

根据经验，顶级类型是`metav1.TypeMeta`嵌入的类型。

碰巧其他接口需要一种深度复制的方式。例如，如果API类型具有接口类型字段，则通常会出现这种情况`Foo`：

```
type SomeAPIType struct {
  Foo Foo `json:"foo"`
}
```

正如我们所看到的，API类型必须是可深度复制的，因此该字段也`Foo`必须进行深度复制。你怎么能这样做，在一个通用的方法（无类型强制转换），无添加`DeepCopyFoo() Foo`的`Foo`接口？

```
type Foo interface {
    ...
    DeepCopyFoo() Foo
}
```

在这种情况下，可以使用相同的标签：

```
// +k8s:deepcopy-gen:interfaces=<package>.Foo
type FooImplementation struct {
    ...
}
```

那里`runtime.Object`是实际使用此标记的Kubernetes源中的一些示例：

```
// +k8s:deepcopy-gen:interfaces=.../pkg/registry/rbac/reconciliation.RuleOwner
// +k8s:deepcopy-gen:interfaces=.../pkg/registry/rbac/reconciliation.RoleBinding
```

## 客户端标签

最后，那里是一些要控制的标签`client-gen`，其中一个我们在前面的例子中看到`At`和`AtList`：

```
// +genclient
```

它告诉`client-gen`为这种类型创建一个客户端（这总是选择加入）。请注意，您不必并且实际上*不得*将其置于`List`API对象的类型之上。

在我们的`cnat`示例中，我们使用*/ status*子资源并使用`UpdateStatus`客户端的方法更新CR的状态（请参阅[“状态子资源”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#status-subresource)）。存在没有状态或没有规范状态拆分的CR的实例。在这些情况下，以下标记可以避免生成该`UpdateStatus()`方法：

```
// +genclient:noStatus
```

###### 警告

没有这个标签，`client-gen`就会盲目地生成`UpdateStatus()`方法。但是，重要的是要理解，只有在CustomResourceDefinition清单中实际启用了*/ status*子资源时，spec-status拆分才有效（请参阅[“子资源”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#crd-subresources)）。

在客户端中单独存在该方法没有任何效果。没有更改清单的请求甚至会失败。

客户端生成器必须选择正确的HTTP路径，可以使用或不使用命名空间。对于群集范围的资源，您必须使用标记：

```
// +genclient:nonNamespaced
```

默认是生成命名空间客户端。同样，这必须与CRD清单中的范围设置相匹配。对于特殊用途客户端，您可能还希望详细控制提供的HTTP方法。您可以使用几个标记来执行此操作，例如：

```
// +genclient:noVerbs
// +genclient:onlyVerbs=create,delete
// +genclient:skipVerbs=get,list,create,update,patch,delete,watch
// +genclient:method=Create,verb=create,
// result=k8s.io/apimachinery/pkg/apis/meta/v1.Status
```

前三个应该是不言自明的，但最后一个需要一些解释。

上面写的这个标签的类型将是仅创建的，不会返回API类型本身，而是a `metav1.Status`。对于CR而言，这没有多大意义，但对于用Go编写的用户提供的API服务器（参见[第8章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#ch_custom-api-servers)），这些资源可以存在，并且它们在实践中存在。

`// +genclient:method=`标记的一个常见情况是添加了一种扩展资源的方法。在[“Scale subresource”中，](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#scale-subresource)我们描述了如何为CR启用*/ scale*子资源。以下标记创建相应的客户端方法：

```
// +genclient:method=GetScale,verb=get,subresource=scale,\
//    result=k8s.io/api/autoscaling/v1.Scale
// +genclient:method=UpdateScale,verb=update,subresource=scale,\
//    input=k8s.io/api/autoscaling/v1.Scale,result=k8s.io/api/autoscaling/v1.Scale
```

第一个标签创建了getter `GetScale`。第二个创建了setter `UpdateScale`。

###### 注意

所有CR */ scale*子资源都`Scale`从*autoscaling / v1*组接收和输出类型。在Kubernetes API中，由于历史原因，有些资源使用其他类型。

## informer-gen和lister-gen

都 `informer-gen`并`lister-gen`处理`// +genclient`标签`client-gen`。没有其他配置。选择客户端生成的每种类型都会自动获取与客户端匹配的*informer和listers*（如果通过*k8s.io/code-generator/generate-group.sh*脚本调用整个生成器套件）。

Kubernetes发电机的文档有很大的改进空间，随着时间的推移肯定会慢慢完善。有关不同生成器的更多信息，查看Kubernetes本身的示例通常很有帮助 - 例如，[k8s.io / api](http://bit.ly/2ZA6dWH)和[OpenShift API类型](http://bit.ly/2KxpKnc)。这两个存储库都有许多高级用例。

此外，请毫不犹豫地查看发电机本身。`deepcopy-gen`在[*main.go*](http://bit.ly/2x9HmN4)文件中有一些文档可用。`client-gen`在[Kubernetes贡献者文档中](http://bit.ly/2WYNlns)提供了一些[文档](http://bit.ly/2WYNlns)。`informer-gen`而`lister-gen`目前还没有进一步的文件，但*generate-groups.sh*显示[每个如何调用](http://bit.ly/31MeSHp)。

# 摘要

在本章中，我们向您展示了如何将Kubernetes代码生成器用于CR。有了这一点，我们现在转向更高级别的抽象工具 - 即编写自定义控制器和运算符的解决方案，使您能够专注于业务逻辑。

[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#idm46336860111320-marker) Go工具在需要时不会自动运行生成，并且缺乏定义源文件和生成文件之间依赖关系的方法。