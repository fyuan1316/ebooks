# 第8章自定义API服务器

如作为CustomResourceDefinitions的替代方案，您可以使用自定义API服务器。自定义API服务器可以使用与主Kubernetes API服务器相同的方式为API组提供资源。与CRD相比，使用自定义API服务器几乎没有任何限制。

本章首先列出了CRD可能不适合您的用例的一些原因。它描述了聚合模式，可以使用自定义API服务器扩展Kubernetes API表面。最后，您将学习使用Golang实际实现自定义API服务器。

# 自定义API服务器的用例

一个可以使用自定义API服务器代替CRD。它可以完成CRD可以做的所有事情，并提供几乎无限的灵活性。当然，这需要付出代价：开发和运营的复杂性。

让我们看一下截至本文撰写时CRD的一些限制（当Kubernetes 1.14是稳定版本时）。会议厅文件：

- 用`etcd`他们的存储介质（或任何Kubernetes API服务器使用）。
- 不支持protobuf，只支持JSON。
- 仅支持两种子资源：*/ status*和*/ scale*（参见[“子资源”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#crd-subresources)）。
- 不支持优雅删除。[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#idm46336853170760)终结器可以模拟此但不允许自定义优雅的删除时间。
- 显着增加Kubernetes API服务器的CPU负载，因为所有算法都以通用方式实现（例如，验证）。
- 仅为API端点实现标准CRUD语义。
- 做不支持同居资源（即不同API组中的资源或共享存储的不同名称的资源）。[2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#idm46336853165528)

一个相反，自定义API服务器没有这些限制。自定义API服务器：

- 可以使用任何存储介质。有自定义API服务器，例如：
  - 所述[度量API服务器](http://bit.ly/2FvgfAV)，其存储数据在内存中的最大性能
  - API服务器镜像[OpenShift中](http://redhat.com/openshift)的Docker注册表
  - API服务器写入时间序列数据库
  - API服务器镜像云API
  - API服务器镜像其他API对象，例如[OpenShift](http://redhat.com/openshift)中反映Kubernetes命名空间的项目
- 可以像所有本地Kubernetes资源一样提供protobuf支持。为此，您必须使用[go-to-protobuf](http://bit.ly/31OLSie)创建一个*.proto*文件，然后使用protobuf编译器生成序列化程序，然后将其编译为二进制文件。`protoc`
- 可以提供任何自定义子资源; 例如，Kubernetes API服务器提供*/ exec*，*/ logs*，*/ port-forward*等，其中大多数使用非常自定义的协议，如WebSockets或HTTP / 2流。
- 可以像Kubernetes对pod一样实现优雅删除。`kubectl`等待删除，用户甚至可以提供 定制优雅的终止期。
- 可以使用Golang以最有效的方式实现所有操作，如验证，准入和转换，而无需通过webhook往返，这会增加进一步的延迟。这对于高性能用例或者存在大量对象很重要。想想拥有数千个节点的巨大集群中的pod对象，以及两个数量级的pod。
- 可以实现自定义语义，例如核心v1 `Service`类型中的服务IP的原子预留。在创建服务时，分配并直接返回唯一的服务IP。在某种程度上，像这样的特殊语义当然可以通过准入webhooks来实现（参见[“Admission Webhooks”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#admission-webhooks)），尽管这些webhooks永远无法可靠地知道传递的对象是否实际创建或更新：它们被乐观地调用，但是稍后请求管道中的步骤可能会取消请求。换句话说：webhook中的副作用很棘手，因为如果请求失败，则没有撤消触发器。
- 可以提供具有公共存储机制（即，公共`etcd`密钥路径前缀）的资源，但是存在于不同的API组中或者以不同的方式命名。例如，Kubernetes在API组中存储部署和其他资源`extensions/v1`，然后将它们移动到更具体的API组`apps/v1`。

换句话说，自定义API服务器是CRD仍然有限的情况下的解决方案。在转换到新语义时不要破坏资源兼容性的过渡场景中，自定义API服务器通常更加灵活。

# 示例：比萨餐厅

至了解自定义API服务器的实现方式，在本节中，我们将介绍一个示例项目：实现比萨餐厅API的自定义API服务器。我们来看看要求。

我们想在`restaurant.programming-kubernetes.info`API组中创建两种类型：

- `Topping`

  披萨配料（例如萨拉米香肠，马苏里拉奶酪或番茄）

- `Pizza`

  餐厅提供的比萨饼类型

浇头是集群范围的资源，仅保留一个单位顶部成本的浮点值。一个实例很简单：

```
apiVersion: restaurant.programming-kubernetes.info/v1alpha1
kind: Topping
metadata:
  name: mozzarella
spec:
  cost: 1.0
```

每个披萨都可以有任意数量的配料; 例如：

```
apiVersion: restaurant.programming-kubernetes.info/v1alpha1
kind: Pizza
metadata:
  name: margherita
spec:
  toppings:
  - mozzarella
  - tomato
```

排序列表是有序的（与YAML或JSON中的任何列表一样），但顺序对于类型的语义并不重要。无论如何，顾客都会得到相同的披萨。我们希望在列表中允许重复，以便允许比如带有额外奶酪的比萨饼。

所有这些都可以通过CRD轻松实现。现在让我们添加一些超出基本CRD功能的要求：[3](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#idm46336853099640)

- 我们希望仅允许具有相应`Topping`对象的披萨规格中的浇头。
- 我们还想假设我们首先将此API作为`v1alpha1`版本引入，但最终了解到我们希望`v1beta1`在同一API 的版本中另外表示浇头。

换句话说，我们希望有两个版本并在它们之间无缝转换。

可以在[本书的GitHub存储库中](http://bit.ly/2x9C3gR)找到此API作为自定义API服务器的完整实现。在本章的其余部分，我们将介绍该项目的所有主要部分并了解其工作原理。在这个过程中，您将在不同的视角中看到前一章中介绍的许多概念：即Klangnetes API服务器后面的Golang实现。CRD中强调的一些设计决策也将变得更加清晰。

因此，即使您不打算使用自定义API服务器的路径，我们也强烈建议您仔细阅读本章。也许这里提出的概念将来也可用于CRD，在这种情况下，了解自定义API服务器将对您有用。

# 架构：聚合

之前 进入技术实现细节，我们希望在Kubernetes集群的上下文中获取自定义API服务器体系结构的更高级别视图。

自定义API服务器是为API组提供服务的进程，通常使用通用API服务器库[*k8s.io/apiserver构建*](http://bit.ly/2X3joNX)。这些进程可以在集群内部或外部运行。在前一种情况下，它们在pods内部运行，前面有一项服务。

该称为主Kubernetes API服务器`kube-apiserver`始终是`kubectl`其他API客户端的第一联系人。由自定义API服务器提供服务的API组由`kube-apiserver`进程代理到自定义API服务器进程。换句话说，该`kube-apiserver`进程知道所有自定义API服务器及其服务的API组，以便能够向它们代理正确的请求。

该执行此代理的组件位于`kube-apiserver`进程内部并被调用[`kube-aggregator`](http://bit.ly/2X10C9W)。将API请求代理到自定义API服务器的过程称为*API聚合*。

让我们看一下针对自定义API服务器的请求路径，但是进入Kubernetes API服务器TCP套接字（参见[图8-1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregation-kube-apiserver)）：

1. Kubernetes API服务器收到请求。
2. 它们通过处理程序链，包括身份验证，审计日志记录，模拟，最大限度飞行限制，授权等（图中只是一个草图，并不完整）。
3. 由于Kubernetes API服务器知道聚合API，因此它可以拦截对HTTP路径*/ apis /的aggregated-API-group-name*请求。
4. Kubernetes API服务器将请求转发给自定义API服务器。

![Kubernetes主要的API服务器`kube-apiserver`，带有集成的`kube-aggregator`](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0801.png)

###### 图8-1。Kubernetes主要的API服务器kube-apiserver，带有集成的kube-aggregator

所述`kube-aggregator`的HTTP路径下代理请求的API组版本（即，一切下*的/ apis / group-name/version*）。它不必知道API组版本中实际提供的资源。

相反，它`kube-aggregator`为所有聚合的自定义API服务器本身提供发现端点*/ apis*和*/ apis /group-name*（它使用下一节中解释的已定义顺序）并返回结果，而不与聚合的自定义API服务器通信。相反，它使用来自`APIService`资源的信息。让我们详细看一下这个过程。

## API服务

对于要知道自定义API服务器所服务的API组的Kubernetes API服务器，`APIService`必须在`apiregistration.k8s.io/v1`API组中创建一个对象。这些对象仅列出API组和版本，而不是资源或任何进一步的详细信息：

```
apiVersion: apiregistration.k8s.io/v1beta1
kind: APIService
metadata:
  name: name
spec:
  group: API-group-name
  version: API-group-version
  service:
    namespace: custom-API-server-service-namespace
    name: -API-server-service
  caBundle: base64-caBundle
  insecureSkipTLSVerify: bool
  groupPriorityMinimum: 2000
  versionPriority: 20
```

该名称是任意的，但为了清楚起见，我们建议您使用标识API组名称和版本的名称 - 例如，*group-name-version*。

该服务可以是群集中的普通[`ClusterIP`服务](http://bit.ly/2X0zEEu)，也可以是具有`ExternalName`群集外自定义API服务器的给定DNS名称的服务。在这两种情况下，端口必须是443.不支持其他服务端口（在撰写本文时）。服务目标端口映射允许任何选定的，优选非限制的更高端口用于自定义API服务器pod，因此这不是主要限制。

证书颁发机构（CA）捆绑包用于Kubernetes API服务器以信任所联系的服务。请注意，API请求可以包含机密数据。为避免中间人攻击，强烈建议您设置`caBundle`字段而不使用`insecureSkipTLSVerify`替代方法。这对于任何生产群集尤其重要，包括证书轮换机制。

最后，`APIService`对象中有两个优先级。这些有一些棘手的语义，在Golang代码文档中描述了`APIService`类型：

```
// GroupPriorityMininum is the priority this group should have at least. Higher
// priority means that the group is preferred by clients over lower priority ones.
// Note that other versions of this group might specify even higher
// GroupPriorityMinimum values such that the whole group gets a higher priority.
//
// The primary sort is based on GroupPriorityMinimum, ordered highest number to
// lowest (20 before 10). The secondary sort is based on the alphabetical
// comparison of the name of the object (v1.bar before v1.foo). We'd recommend
// something like: *.k8s.io (except extensions) at 18000 and PaaSes
// (OpenShift, Deis) are recommended to be in the 2000s
GroupPriorityMinimum int32 `json:"groupPriorityMinimum"`

// VersionPriority controls the ordering of this API version inside of its
// group. Must be greater than zero. The primary sort is based on
// VersionPriority, ordered highest to lowest (20 before 10). Since it's inside
// of a group, the number can be small, probably in the 10s. In case of equal
// version priorities, the version string will be used to compute the order
// inside a group. If the version string is "kube-like", it will sort above non
// "kube-like" version strings, which are ordered lexicographically. "Kube-like"
// versions start with a "v", then are followed by a number (the major version),
// then optionally the string "alpha" or "beta" and another number (the minor
// version). These are sorted first by GA > beta > alpha (where GA is a version
// with no suffix such as beta or alpha), and then by comparing major version,
// then minor version. An example sorted list of versions:
// v10, v2, v1, v11beta2, v10beta3, v3beta1, v12alpha1, v11alpha2, foo1, foo10.
VersionPriority int32 `json:"versionPriority"`
```

换句话说，该`GroupPriorityMinimum`值确定组优先级的位置。如果`APIService`不同版本的多个对象不同，则最高值规则。

第二个优先级只是在彼此之间对版本进行排序，以定义动态客户端要使用的首选版本。

以下是`GroupPriorityMinimum`本机Kubernetes API组的值列表：

```
var apiVersionPriorities = map[schema.GroupVersion]priority{
    {Group: "", Version: "v1"}: {group: 18000, version: 1},
    {Group: "extensions", Version: "v1beta1"}: {group: 17900, version: 1},
    {Group: "apps", Version: "v1beta1"}:                         {group: 17800, version: 1},
    {Group: "apps", Version: "v1beta2"}:                         {group: 17800, version: 9},
    {Group: "apps", Version: "v1"}:                              {group: 17800, version: 15},
    {Group: "events.k8s.io", Version: "v1beta1"}:                {group: 17750, version: 5},
    {Group: "authentication.k8s.io", Version: "v1"}:             {group: 17700, version: 15},
    {Group: "authentication.k8s.io", Version: "v1beta1"}:        {group: 17700, version: 9},
    {Group: "authorization.k8s.io", Version: "v1"}:              {group: 17600, version: 15},
    {Group: "authorization.k8s.io", Version: "v1beta1"}:         {group: 17600, version: 9},
    {Group: "autoscaling", Version: "v1"}:                       {group: 17500, version: 15},
    {Group: "autoscaling", Version: "v2beta1"}:                  {group: 17500, version: 9},
    {Group: "autoscaling", Version: "v2beta2"}:                  {group: 17500, version: 1},
    {Group: "batch", Version: "v1"}:                             {group: 17400, version: 15},
    {Group: "batch", Version: "v1beta1"}:                        {group: 17400, version: 9},
    {Group: "batch", Version: "v2alpha1"}:                       {group: 17400, version: 9},
    {Group: "certificates.k8s.io", Version: "v1beta1"}:          {group: 17300, version: 9},
    {Group: "networking.k8s.io", Version: "v1"}:                 {group: 17200, version: 15},
    {Group: "networking.k8s.io", Version: "v1beta1"}:            {group: 17200, version: 9},
    {Group: "policy", Version: "v1beta1"}:                       {group: 17100, version: 9},
    {Group: "rbac.authorization.k8s.io", Version: "v1"}:         {group: 17000, version: 15},
    {Group: "rbac.authorization.k8s.io", Version: "v1beta1"}:    {group: 17000, version: 12},
    {Group: "rbac.authorization.k8s.io", Version: "v1alpha1"}:   {group: 17000, version: 9},
    {Group: "settings.k8s.io", Version: "v1alpha1"}:             {group: 16900, version: 9},
    {Group: "storage.k8s.io", Version: "v1"}:                    {group: 16800, version: 15},
    {Group: "storage.k8s.io", Version: "v1beta1"}:               {group: 16800, version: 9},
    {Group: "storage.k8s.io", Version: "v1alpha1"}:              {group: 16800, version: 1},
    {Group: "apiextensions.k8s.io", Version: "v1beta1"}:         {group: 16700, version: 9},
    {Group: "admissionregistration.k8s.io", Version: "v1"}:      {group: 16700, version: 15},
    {Group: "admissionregistration.k8s.io", Version: "v1beta1"}: {group: 16700, version: 12},
    {Group: "scheduling.k8s.io", Version: "v1"}:                 {group: 16600, version: 15},
    {Group: "scheduling.k8s.io", Version: "v1beta1"}:            {group: 16600, version: 12},
    {Group: "scheduling.k8s.io", Version: "v1alpha1"}:           {group: 16600, version: 9},
    {Group: "coordination.k8s.io", Version: "v1"}:               {group: 16500, version: 15},
    {Group: "coordination.k8s.io", Version: "v1beta1"}:          {group: 16500, version: 9},
    {Group: "auditregistration.k8s.io", Version: "v1alpha1"}:    {group: 16400, version: 1},
    {Group: "node.k8s.io", Version: "v1alpha1"}:                 {group: 16300, version: 1},
    {Group: "node.k8s.io", Version: "v1beta1"}:                  {group: 16300, version: 9},
}
```

因此，使用`2000`类似PaaS的API意味着将它们放在此列表的末尾。[4](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#idm46336852602264)

API组的顺序在REST映射过程中起作用`kubectl`（请参阅[“REST映射”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#RESTMapping)）。这意味着它对用户体验有实际影响。如果存在冲突的资源名称或短名称，则具有最高`GroupPriorityMinimum`值的名称将获胜。

此外，在使用自定义API服务器替换API组版本的特殊情况下，可能会使用此优先级排序。例如，您可以通过将自定义API服务放在`GroupPriorityMinimum`值低于上表中的位置的位置，将原生Kubernetes API组替换为已修改的组（无论出于何种原因）。

再次注意，Kubernetes API服务器不需要知道发现端点*/ apis*和*/ apis /的group-name*任何资源列表，也不需要知道代理。资源列表仅通过第三个发现端点*/ apis / group-name/返回version*。但正如我们在上一节中看到的那样，此端点由聚合的自定义API服务器提供，而不是由`kube-aggregator`。

## 自定义API服务器的内部结构

一个自定义API服务器类似于组成Kubernetes API服务器的大多数部分，当然具有不同的API组实现，并且没有嵌入式`kube-aggregator`或嵌入式`apiextension-apiserver`（为CRD提供服务）。这导致几乎相同的架构图（如图[8-2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregation-aggregated-apiserver)所示）与[图8-1中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregation-kube-apiserver)的相似 ：

![基于k8s.io/apiserver的聚合自定义API服务器](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0802.png)

###### 图8-2。基于k8s.io/apiserver的聚合自定义API服务器

我们观察了很多事情。聚合API服务器：

- 具有与Kubernetes API服务器相同的基本内部结构。
- 有它自己的 处理程序链，包括身份验证，审计， 模拟，最大限度飞行限制和授权（我们将在本章中解释为什么这是必要的;例如，参见[“委托授权”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-authorization)）。
- 有自己的资源处理程序管道，包括 解码， 转换，许可，REST映射和 编码。
- 呼叫入场webhooks。
- 可能会写入`etcd`（但它可以使用不同的存储后端）。`etcd`群集不必与Kubernetes API服务器使用的群集相同。
- 有自己的自定义API组的方案和注册表实现。注册表实现可能有所不同，并可在任何程度上进行定制
- 再次验证。它通常执行客户端证书身份验证和基于令牌的身份验证，并通过`TokenAccessReview`请求回调Kubernetes API服务器。我们将在稍后更详细地讨论身份验证和信任体系结构。
- 有自己的审计。这意味着Kubernetes API服务器会审核某些字段，但仅限于元级别。对象级审计在聚合的自定义API服务器中完成。
- 使用`SubjectAccessReview`对Kubernetes API服务器的请求进行自己的身份验证。我们将在稍后详细讨论授权。

## 委托身份验证和信任

一个聚合自定义API服务器（基于[*k8s.io/apiserver*](http://bit.ly/2X3joNX)）构建在与Kubernetes API服务器相同的身份验证库上。它可以使用客户端证书或令牌来验证用户身份。

由于聚合的自定义API服务器在架构上位于Kubernetes API服务器后面（即，Kubernetes API服务器接收请求并将它们代理到聚合的自定义API服务器），因此Kubernetes API服务器已对请求进行了身份验证。所述Kubernetes API服务器存储的认证，也就是说，用户名和组成员的HTTP请求报头，通常的结果`X-Remote-User`和`X-Remote-Group`（这些可以用配置`--requestheader-username-headers`和`--requestheader-group-headers`标志）。

聚合的自定义API服务器必须知道何时信任这些标头; 否则，任何其他呼叫者可以声称已完成身份验证并可以设置这些标头。这由特殊请求标头客户端CA处理。它存储在配置映射*kube-system / extension-apiserver-authentication*（filename *requestheader-client-ca-file*）中。这是一个例子：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: extension-apiserver-authentication
  namespace: kube-system
data:
  client-ca-file: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
  requestheader-allowed-names: '["aggregator"]'
  requestheader-client-ca-file: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
  requestheader-extra-headers-prefix: '["X-Remote-Extra-"]'
  requestheader-group-headers: '["X-Remote-Group"]'
  requestheader-username-headers: '["X-Remote-User"]'
```

使用此信息，具有默认设置的聚合自定义API服务器将进行身份验证：

- 使用与给定*client-ca文件*匹配的客户端证书的*客户端*
- 由Kubernetes API服务器预先验证的客户端，其请求使用给定的*requestheader-client-ca-file*转发，其用户名和组成员身份存储在给定的HTTP头中`X-Remote-Group`，`X-Remote-User`

最后但并非最不重要的是，有一种称为`TokenAccessReview`前锋的机制承载令牌（通过HTTP头接收）返回Kubernetes API服务器以验证它们是否有效。默认情况下禁用令牌访问查看机制，但可以选择启用它; 请参阅[“选项和配置模式以及启动管道”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-options-config)。`Authorization: bearer *token*`

我们将在以下部分中看到如何实际设置委派身份验证。虽然我们在这里详细介绍了这种机制，但在聚合的自定义API服务器中，这主要由*k8s.io/apiserver*库自动完成。但是知道幕后发生的事情肯定是有价值的，特别是在涉及安全的情况下。

## 委托授权

后身份验证已完成，每个请求都必须经过授权。授权基于用户名和组列表。该 Kubernetes中的默认授权机制是基于角色的访问控制（RBAC）。

RBAC将身份映射到角色，将角色映射到授权规则，最终接受或拒绝请求。我们不会详细介绍RBAC授权对象（如角色和集群角色，角色绑定和集群角色绑定）的所有详细信息（有关详细信息，请参阅[“获取权限”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch07.html#crds-rbac)）。从架构的角度来看，知道聚合的自定义API服务器通过`SubjectAccessReview`s 授权使用委托授权就足够了。它不会评估RBAC规则本身，而是将评估委托给Kubernetes API服务器。

##### 为什么聚合API服务器总是需要执行另一个授权步骤

Kubernetes API服务器收到并转发到聚合自定义API服务器的每个请求都会通过身份验证和授权（请参[见图8-1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregation-kube-apiserver)）。这意味着聚合的自定义API服务器可以跳过此类请求的委派授权部分。

但这种预授权不能保证，可能会消失在任何时候（有计划地分割`kube-aggregator`，从`kube-apiserver`更好的安全性，并在未来更好的扩展性）。此外，直接发送到聚合自定义API服务器的请求（例如，通过客户端证书或令牌访问查看进行身份验证）不会通过Kubernetes API服务器，因此未经过预授权。

换句话说，跳过委托授权会打开一个安全漏洞，因此非常不鼓励。

现在让我们更详细地看一下委托授权。

一个主题访问审核是根据请求从聚合的自定义API服务器发送到Kubernetes API服务器（如果它在其授权缓存中找不到答案）。以下是此类评论对象的示例：

```
apiVersion: authorization.k8s.io/v1
kind: SubjectAccessReview
spec:
  resourceAttributes:
    group: apps
    resource: deployments
    verb: create
    namespace: default
    version: v1
    name: example
  user: michael
  groups:
  - system:authenticated
  - admins
  - authors
```

Kubernetes API服务器从聚合的自定义API服务器接收此信息，评估集群中的RBAC规则，并做出决定，返回一个`SubjectAccessReview`设置了状态字段的对象; 例如：

```
apiVersion: authorization.k8s.io/v1
kind: SubjectAccessReview
status:
  allowed: true
  denied: false
  reason: "rule foo allowed this request"
```

这里注意，有可能是两个`allowed`和`denied`是`false`。这意味着Kubernetes API服务器无法做出决定，在这种情况下，聚合自定义API服务器中的另一个授权者可以做出决定（API服务器实现逐个查询的授权链，委派授权是授权者之一在那个链中）。这可用于建模非标准授权逻辑 - 即，在某些情况下，如果没有RBAC规则，而是使用外部授权系统。

请注意，出于性能原因，委派授权机制在每个聚合自定义API服务器中维护本地缓存。默认情况下，它缓存1,024个授权条目：

- `5` 允许的授权请求的分钟到期时间
- `30` 被拒绝的授权请求的秒到期

这些值可以通过`--authorization-webhook-cache-authorized-ttl`和自定义`--authorization-webhook-cache-unauthorized-ttl`。

我们将在以下部分中看到如何在代码中设置委派授权。同样，与身份验证一样，在聚合的自定义API服务器内部委托授权主要由*k8s.io/apiserver*库自动完成。

# 编写自定义API服务器

在在前面的部分中，我们研究了聚合API服务器的体系结构。在本节中，我们将了解Golang中聚合的自定义API服务器的实现。

主要的Kubernetes API服务器是通过*k8s.io/apiserver*库实现的。自定义API服务器将使用完全相同的代码。主要区别在于我们的自定义API服务器将在集群中运行。这意味着它可以假设`kube-apiserver`群集中有a 可用，并使用它来执行委派授权并检索其他kube-native资源。

我们还假设`etcd`群集可用并准备好由聚合的自定义API服务器使用。这`etcd`是专用的还是与Kubernetes API服务器共享并不重要。我们的自定义API服务器将使用不同的`etcd`密钥空间来避免冲突。

本章中的代码示例引用[了GitHub上的示例代码](http://bit.ly/2x9C3gR)，因此请查看完整的源代码。我们将在此处仅显示最有趣的摘录，但您可以随时查看完整的示例项目，对其进行实验，并且非常重要的学习 - 在真实的群集中运行它。

该`pizza-apiserver`项目实现了[“示例：比萨餐厅”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregation-example)显示的示例API 。

## 选项和配置模式和启动管道

1. 该*k8s.io/apiserver*库使用*选项和配置模式*来创建正在运行的API服务器。

我们将从几个绑定到标志的选项结构开始。从*k8s.io/apiserver*获取它们并添加我们的自定义选项。来自*k8s.io/apiserver的*选项结构可以在特殊用例的代码中进行调整，并且提供的标志可以应用于标志集以便用户可访问。

在这个[例子中，](http://bit.ly/2x9C3gR)我们非常简单地将所有内容都基于`RecommendedOptions`。这些推荐选项为简单API的“普通”聚合自定义API服务器所需的一切设置，如下所示：

```
import (
    ...
    informers "github.com/programming-kubernetes/pizza-apiserver/pkg/
    generated/informers/externalversions"
)

const defaultEtcdPathPrefix = "/registry/restaurant.programming-kubernetes.info"

type CustomServerOptions struct {
    RecommendedOptions *genericoptions.RecommendedOptions
    SharedInformerFactory informers.SharedInformerFactory
}

func NewCustomServerOptions(out, errOut io.Writer) *CustomServerOptions {
    o := &CustomServerOptions{
        RecommendedOptions: genericoptions.NewRecommendedOptions(
            defaultEtcdPathPrefix,
            apiserver.Codecs.LegacyCodec(v1alpha1.SchemeGroupVersion),
            genericoptions.NewProcessInfo("pizza-apiserver", "pizza-apiserver"),
        ),
    }

    return o
}
```

该`CustomServerOptions`嵌入`RecommendedOptions`和顶部添加一个字段。`NewCustomServerOptions`是`CustomServerOptions`使用默认值填充结构的构造函数。

让我们看看一些更有趣的细节：

- `defaultEtcdPathPrefix`是`etcd`我们所有键的前缀。作为关键空间，我们使用*/registry/pizza-apiserver.programming-kubernetes.info*，明显不同于Kubernetes键。
- `SharedInformerFactory`是我们自己的CR的全流程共享informer工厂，以避免相同资源的不必要的informer（参见[图3-5](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#informers-figure)）。请注意，它是从我们项目中生成的informer代码导入的，而不是从`client-go`。
- `NewRecommendedOptions` 为具有默认值的聚合自定义API服务器设置所有内容。

我们来看看`NewRecommendedOptions`：

```
return &RecommendedOptions{
    Etcd:           NewEtcdOptions(storagebackend.NewDefaultConfig(prefix, codec)),
    SecureServing:  sso.WithLoopback(),
    Authentication: NewDelegatingAuthenticationOptions(),
    Authorization:  NewDelegatingAuthorizationOptions(),
    Audit:          NewAuditOptions(),
    Features:       NewFeatureOptions(),
    CoreAPI:        NewCoreAPIOptions(),
    ExtraAdmissionInitializers:
      func(c *server.RecommendedConfig) ([]admission.PluginInitializer, error) {
          return nil, nil
      },
    Admission:      NewAdmissionOptions(),
    ProcessInfo:    processInfo,
    Webhook:        NewWebhookOptions(),
}
```

如有必要，所有这些都可以调整。例如，如果需要自定义默认服务端口，则`RecommendedOptions.SecureServing.SecureServingOptions.BindPort`可以进行设置。

让我们简要介绍一下现有的内容 选项结构：

- `Etcd`配置读写的存储堆栈`etcd`。
- `SecureServing` 配置HTTPS周围的一切（即端口，证书等）
- `Authentication`设置委派身份验证，如[“委派身份验证和信任”中所述](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-authentication)。
- `Authorization`按照[“委派授权”中的说明](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-authorization)设置委派授权。
- `Audit`设置审计输出堆栈。默认情况下禁用此选项，但可以将其设置为输出审核日志文件或将审核事件发送到外部后端。
- `Features` 提供配置 alpha和beta特征的特征门。
- `CoreAPI`保存kubeconfig文件的路径以访问主API服务器。默认使用群集内配置。
- `Admission`是一堆变异和验证的准入插件，可以为每个传入的API请求执行。这可以通过自定义代码内准入插件进行扩展，也可以针对自定义API服务器调整默认准入链。
- `ExtraAdmissionInitializers` 允许我们添加更多初始化者入学。初始化程序通过自定义API服务器实现例如线程或客户端的管道。有关自定义录取的更多信息，请参阅[“录取”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-admission)。
- `ProcessInfo`保存事件对象创建的信息（即进程名称和命名空间）。我们已经`pizza-apiserver`为两个值设置了它。
- `Webhook`配置webhook的操作方式（例如，身份验证和入场webhook的一般设置）。它为在集群内运行的自定义API服务器设置了良好的默认值。对于群集外部的API服务器，这将是配置它如何到达webhook的地方。

选项与标志相结合; 也就是说，它们通常与标志处于相同的抽象级别。根据经验，选项不包含“运行”数据结构。它们在启动期间使用，然后转换为配置或服务器对象，然后运行它们。

可以通过该`Validate() error`方法验证选项。此方法还将检查用户提供的标志值是否具有逻辑意义。

可以完成选项以设置默认值，默认值不应显示在标志的帮助文本中，但这些默认值是获取完整选项集所必需的。

通过该`Config() (*apiserver.Config, error)`方法将选项转换为服务器配置（“config”）。这是通过从推荐的默认配置开始，然后将选项应用于它来完成的：

```
func (o *CustomServerOptions) Config() (*apiserver.Config, error) {
    err := o.RecommendedOptions.SecureServing.MaybeDefaultWithSelfSignedCerts(
        "localhost", nil, []net.IP{net.ParseIP("127.0.0.1")},
    )
    if err != nil {
        return nil, fmt.Errorf("error creating self-signed cert: %v", err)
    }

    [... omitted o.RecommendedOptions.ExtraAdmissionInitializers ...]

    serverConfig := genericapiserver.NewRecommendedConfig(apiserver.Codecs)
    err = o.RecommendedOptions.ApplyTo(serverConfig, apiserver.Scheme);
    if err != nil {
        return nil, err
    }

    config := &apiserver.Config{
        GenericConfig: serverConfig,
        ExtraConfig:   apiserver.ExtraConfig{},
    }
    return config, nil
}
```

这里创建的配置包含可运行的数据结构; 换句话说，配置是运行时对象，与对应于标志的选项形成对比。`o.RecommendedOptions.SecureServing.MaybeDefaultWithSelfSignedCerts`如果用户未传递预生成证书的标志，该行将创建自签名证书。

正如我们所描述的，`genericapiserver.NewRecommendedConfig`返回默认的推荐配置，并`RecommendedOptions.ApplyTo`根据标志（和其他自定义选项）进行更改。

`pizza-apiserver`项目本身的配置结构只是`RecommendedConfig`我们的示例自定义API服务器的包装：

```
type ExtraConfig struct {
    // Place your custom config here.
}

type Config struct {
    GenericConfig *genericapiserver.RecommendedConfig
    ExtraConfig   ExtraConfig
}

// CustomServer contains state for a Kubernetes custom api server.
type CustomServer struct {
    GenericAPIServer *genericapiserver.GenericAPIServer
}

type completedConfig struct {
    GenericConfig genericapiserver.CompletedConfig
    ExtraConfig   *ExtraConfig
}

type CompletedConfig struct {
    // Embed a private pointer that cannot be instantiated outside of
    // this package.
    *completedConfig
}
```

如果需要更多运行自定义API服务器的状态，`ExtraConfig`则可以放置它。

与选项结构类似，配置有一个`Complete() CompletedConfig`设置默认值的方法。因为有必要实际调用`Complete()`底层配置，所以通常通过引入未导出的`completedConfig`数据类型来强制通过类型系统实现。这里的想法是只有一个电话`Complete()`可以`Config`变成一个`completeConfig`。如果没有完成此调用，编译器将会抱怨：

```
func (cfg *Config) Complete() completedConfig {
    c := completedConfig{
        cfg.GenericConfig.Complete(),
        &cfg.ExtraConfig,
    }

    c.GenericConfig.Version = &version.Info{
        Major: "1",
        Minor: "0",
    }

    return completedConfig{&c}
}
```

最后，完成的配置可以`CustomServer`通过`New()`构造函数转换为运行时结构：

```
// New returns a new instance of CustomServer from the given config.
func (c completedConfig) New() (*CustomServer, error) {
    genericServer, err := c.GenericConfig.New(
        "pizza-apiserver",
        genericapiserver.NewEmptyDelegate(),
    )
    if err != nil {
        return nil, err
    }

    s := &CustomServer{
        GenericAPIServer: genericServer,
    }

    [ ... omitted API installation ...]

    return s, nil
}
```

请注意，我们在此有意省略了API安装部分。我们将在[“API安装”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-api-install)回顾这一点（即，在启动期间如何将*注册表连接*到自定义API服务器）。注册表实现API组的API和存储语义。我们将在[“注册表和策略”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-registry)看到餐厅API组。

`CustomServer`最终可以使用该`Run(stopCh <-chan struct{}) error`方法启动该对象。这是通过`Run`我们示例中的选项方法调用的。那就是`CustomServerOptions.Run`：

- 创建配置
- 完成配置
- 创造 `CustomServer`
- 呼叫 `CustomServer.Run`

这是代码：

```
func (o CustomServerOptions) Run(stopCh <-chan struct{}) error {
    config, err := o.Config()
    if err != nil {
        return err
    }

    server, err := config.Complete().New()
    if err != nil {
        return err
    }

    server.GenericAPIServer.AddPostStartHook("start-pizza-apiserver-informers",
        func(context genericapiserver.PostStartHookContext) error {
            config.GenericConfig.SharedInformerFactory.Start(context.StopCh)
            o.SharedInformerFactory.Start(context.StopCh)
            return nil
        },
    )

    return server.GenericAPIServer.PrepareRun().Run(stopCh)
}
```

该`PrepareRun()`调用连接了OpenAPI规范，并可能执行其他API后安装操作。调用它之后，该`Run`方法启动实际的服务器。它会阻塞直到`stopCh`关闭。

这个示例还连接一个名为*的启动后挂钩*`start-pizza-apiserver-informers`。顾名思义，在HTTPS服务器启动和监听后调用启动后挂钩。在这里，它启动了共享的informer工厂。

请注意，即使是自定义API服务器本身提供的资源的本地进程内信息，也可以通过HTTPS向localhost接口说明。因此，在服务器启动且HTTPS端口正在侦听之后启动它们是有意义的。

另请注意，只有在所有启动后挂钩成功完成后，*/ healthz*端点才会返回成功。

随着所有小管道部件的到位，`pizza-apiserver`项目将所有内容包装成一个`cobra`命令：

```
// NewCommandStartCustomServer provides a CLI handler for 'start master' command
// with a default CustomServerOptions.
func NewCommandStartCustomServer(
    defaults *CustomServerOptions,
    stopCh <-chan struct{},
) *cobra.Command {
    o := *defaults
    cmd := &cobra.Command{
        Short: "Launch a custom API server",
        Long:  "Launch a custom API server",
        RunE: func(c *cobra.Command, args []string) error {
            if err := o.Complete(); err != nil {
                return err
            }
            if err := o.Validate(); err != nil {
                return err
            }
            if err := o.Run(stopCh); err != nil {
                return err
            }
            return nil
        },
    }

    flags := cmd.Flags()
    o.RecommendedOptions.AddFlags(flags)

    return cmd
}
```

使用`NewCommandStartCustomServer`该`main()`过程的方法非常简单：

```
func main() {
    logs.InitLogs()
    defer logs.FlushLogs()

    stopCh := genericapiserver.SetupSignalHandler()
    options := server.NewCustomServerOptions(os.Stdout, os.Stderr)
    cmd := server.NewCommandStartCustomServer(options, stopCh)
    cmd.Flags().AddGoFlagSet(flag.CommandLine)
    if err := cmd.Execute(); err != nil {
        klog.Fatal(err)
    }
}
```

特别注意调用`SetupSignalHandler`：它连接Unix信号处理。开`SIGINT`（在终端中按Ctrl-C时触发）`SIGKILL`，停止通道关闭。停止通道将传递给正在运行的自定义API服务器，并在停止通道关闭时关闭。因此，当接收到其中一个信号时，主循环将启动关闭。在终止之前运行请求完成（默认情况下最多60秒）的意义上，此关闭是优雅的。它还确保将所有请求发送到审计后端，并且不会删除任何审计数据。毕竟，`cmd.Execute()`将返回并且该过程将终止。

## 第一个开始

现在我们已经准备好了第一次启动自定义API服务器的所有内容。假设您在*〜/ .kube / config中*配置了一个集群，您可以将其用于委派身份验证和授权：

```
$ cd $GOPATH/src/github.com/programming-kubernetes/pizza-apiserver
 $ etcd &
$ go run。--etcd-servers localhost：2379 \
    --authentication-kubeconfig~ / .kube / config \
    --authorization-kubeconfig~ / .kube / config\
    --kubeconfig~ / .kube / config
I0331 11：33：25.702320    64244plugins.go：158 ]
  加载3突变接纳控制器(小号)成功以下顺序：
     NamespaceLifecycle，MutatingAdmissionWebhook，PizzaToppings。
I0331 11：33：25.702344    64244plugins.go：161 ]
  加载1验证许可控制器(小号)成功以下顺序：
     ValidatingAdmissionWebhook。
I0331 11：33：25.714148    64244secure_serving.go：116 ]上的安全服务[:: ]：443
```

它将启动并开始提供通用API端点：

```
$ curl -k https：// localhost：443 / healthz
好
```

我们也可以列出 发现端点，但结果还不是很令人满意 - 我们还没有创建API，所以发现是空的：

```
$ curl -k https：// localhost：443 / apis
 {
  "kind": "APIGroupList",
   "groups"：[]
}
```

我们来看一个更高层次：

- 我们已经使用推荐的选项和配置启动了自定义API服务器。
- 我们有一个标准的处理程序链，包括委托身份验证，委派授权和审计。
- 我们有一台HTTPS服务器正在运行并提供对通用端点的请求：*/ logs*，*/ metrics*，*/ version*，*/ healthz*和*/ apis*。

[图8-3](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregation-kube-apiserver_without)显示了10,000英尺的距离。

![没有API的自定义API服务器](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0803.png)

###### 图8-3。没有API的自定义API服务器

## 内部类型和转换

现在我们已经设置了一个运行的自定义API服务器，是时候实际实现API了。在此之前，我们必须了解API版本以及如何在API服务器内处理它们。

每个API服务器都提供许多资源和版本（参见[图2-3](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#gvr)）。有些资源有多个版本。为了使资源的多个版本成为可能，API服务器 转换版本。

为避免版本之间必要转换的二次增长，API服务器使用 实现实际API逻辑时的*内部版本*。内部版本也经常被调用 *集线器版本，*因为它是一种集合，每个其他版本都转换为和从中转换（[见图8-4](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregation-version-star)）。内部API逻辑仅针对该集线器版本实现一次。

![从集线器版本转换到集线器版本](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0804.png)

###### 图8-4。从集线器版本转换到集线器版本

[图8-5](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregation-conversions-figure)显示了API服务器如何在API请求的生命周期中使用内部版本：

- 用户使用特定版本（例如，`v1`）发送请求。
- API服务器解码有效负载并将其转换为内部版本。
- API服务器通过许可和验证传递内部版本。
- API逻辑是为注册表中的内部版本实现的。
- `etcd`读取和写入版本化对象（例如，`v2`- 存储版本）; 也就是说，它从内部版本转换为内部版本。
- 最后，在这种情况下，结果将转换为请求版本`v1`。

![在请求的生命周期中转换API对象](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0805.png)

###### 图8-5。在请求的生命周期中转换API对象

在内部集线器版本和之间的每个边缘外部版本，进行转换。在[图8-6中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregation-conversions-points)，您可以计算每个请求处理程序的转换次数。在写入操作（如创建和更新）中，至少完成四次转换，如果在群集中部署了许可webhook，则会进行更多转换。如您所见，转换是每个API实现中的关键操作。

![请求生命周期内的转换和默认](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0806.png)

###### 图8-6。在请求的生命周期中进行转换和默认

在除转换外，[图8-6](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregation-conversions-points)还显示了何时发生*默认*情况。默认是填写未指定字段值的过程。默认与转换高度耦合，并且当来自用户的请求，来自`etcd`或来自接纳webhook时，始终在外部版本上完成，但从从集线器转换到外部版本时永远不会。

###### 警告

转变对API服务器机制至关重要。所有转换（来回）在*圆形*转换的意义上必须是正确的也是至关重要的。Roundtrippable意味着我们可以在版本图中来回转换（[图8-4](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregation-version-star)），从随机值开始，我们永远不会丢失任何信息; 也就是说，转换是双向的，或者是一对一的。例如，我们必须能够从随机（但有效）`v1`对象转到内部集线器类型，然后`v1alpha1`返回到内部集线器类型，然后返回到`v1`。生成的对象必须等同于原始对象。

制作可环绕的类型通常需要经过深思熟虑; 它几乎总是驱动新版本的API设计，并且还影响旧类型的扩展，以便存储新版本携带的信息。

简而言之：正确地进行往返是很困难的。请参阅[“往返测试”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-roundtrip)以了解如何有效地测试往返。

在API服务器的生命周期中，默认逻辑可能会发生变化。想象一下，您为类型添加了一个新字段。用户可能将旧对象存储在磁盘上，或者`etcd`可能具有旧对象。如果该新字段具有默认值，则在将旧的存储对象发送到API服务器时或者当用户从中检索其中一个旧对象时设置此字段值`etcd`。看起来新字段永远存在，而实际上API服务器中的默认进程在处理请求期间设置字段值。

## 编写API类型

如我们已经看到，要向自定义API服务器添加API，我们必须编写内部集线器版本类型和外部版本类型，并在它们之间进行转换。这就是我们现在要看的[披萨示例项目](http://bit.ly/2x9C3gR)。

API类型在传统上是为*PKG的/ apis /group-name*包与该项目的*PKG的/ apis / group-name/types.go*内部类型和*PKG的/ apis / group-name/ version/types.go*用于外部的版本）。因此，对于我们的示例，*pkg / apis / restaurant*，*pkg / apis / restaurant / v1alpha1 / types.go*和*pkg / apis / restaurant / v1beta1 / types.go*。

转换将在*pkg / apis / group-name/ version/zz_generated.conversion.go*（用于`conversion-gen`输出）和*pkg / apis / group-name/ version/conversion.go中创建，*用于开发人员编写的自定义转换。

以类似的方式，将为*pkg / apis /* */* */zz_generated.defaults.go*和*pkg / apis /* */* */defaults.go上的*`defaulter-gen`输出创建默认代码，以获取开发人员编写的自定义默认代码。在我们的例子中，我们有*pkg / apis / restaurant / v1alpha1 / defaults.go*和*pkg / apis / restaurant / v1beta1 / defaults.go*。*group-nameversion**group-nameversion*

我们将详细介绍转换和默认[“转换”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-conversion)和[“默认”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-defaulting)。

除了转换和默认之外，我们已经在[“剖析类型”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#anatomy-of-CRD-types)看到了CustomResourceDefinitions的大部分过程。我们的自定义API服务器中的外部版本的本机类型的定义方式完全相同。

另外，对于内部类型，集线器类型，我们有*pkg / apis / group-name/types.go*。主要区别在于后者`SchemeGroupVersion`在*register.go*文件中引用`runtime.APIVersionInternal`（这是一个快捷方式`"__internal"`）。

```
// SchemeGroupVersion is group version used to register these objects
var SchemeGroupVersion = schema.GroupVersion{Group: GroupName, Version:
runtime.APIVersionInternal}
```

和外部类型文件之间的另一个区别是缺少JSON和protobuf标签。`pkg/apis/*group-name*/types.go`

###### 小费

某些生成器使用JSON标记来检测*types.go*文件是用于外部版本还是内部版本。因此，在复制和粘贴外部类型时，请始终删除这些标记，以便创建或更新内部类型。

最后但并非最不重要的是，有一个帮助程序可以将API组的所有版本安装到方案中。传统上，此助手位于*pkg / apis / group-name/install/install.go中*。对于我们的自定义API服务器*pkg / apis / restaurant / install / install.go*，它看起来很简单：

```
// Install registers the API group and adds types to a scheme
func Install(scheme *runtime.Scheme) {
    utilruntime.Must(restaurant.AddToScheme(scheme))
    utilruntime.Must(v1beta1.AddToScheme(scheme))
    utilruntime.Must(v1alpha1.AddToScheme(scheme))
    utilruntime.Must(scheme.SetVersionPriority(
        v1beta1.SchemeGroupVersion,
        v1alpha1.SchemeGroupVersion,
    ))
}
```

因为我们有多个版本，所以必须定义优先级。此订单将用于确定资源的默认存储版本。它曾经在内部客户端（返回内部版本对象的客户端）中的版本选择中发挥作用;请参阅注释[“过去的版本化客户端和内部客户端”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#internal-clients)。但是内部客户已被弃用并且正在消失。即使API服务器内的代码将来也会使用外部版本客户端。

## 转换

转变获取一个版本中的对象并将其转换为另一个版本中的对象。转换是通过转换函数实现的，其中一些是手工编写的（*按惯例*放入*pkg / apis / group-name/ / versionconversion.go*），另一些是自动生成的[`conversion-gen`](http://bit.ly/31RewiP)（按惯例放入*pkg / apis / group-name/ version/zz_generated.conversion.go*）。

使用该方法通过方案（参见[“Scheme”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#scheme)）启动转换`Convert()`，传递源对象`in`和目标对象`out`：

```
func (s *Scheme) Convert(in, out interface{}, context interface{}) error
```

将`context`被描述为如下：

```
// ...an optional field that callers may use to pass info to conversion functions.
```

它仅在非常特殊的情况下使用，通常是`nil`。在本章的后面，我们将介绍转换函数范围，它允许我们从转换函数中访问此上下文。

为了进行实际转换，该方案了解所有Golang API类型，它们的类型GroupVersionKinds，以及GroupVersionKinds之间的转换函数。为此，`conversion-gen`寄存器通过本地方案构建器生成转换函数。在我们的示例自定义API服务器中，*zz_generated.conversion.go*文件的开头如下：

```
func init() {
    localSchemeBuilder.Register(RegisterConversions)
}

// RegisterConversions adds conversion functions to the given scheme.
// Public to allow building arbitrary schemes.
func RegisterConversions(s *runtime.Scheme) error {
    if err := s.AddGeneratedConversionFunc(
        (*Topping)(nil),
        (*restaurant.Topping)(nil),
        func(a, b interface{}, scope conversion.Scope) error {
            return Convert_v1alpha1_Topping_To_restaurant_Topping(
                a.(*Topping),
                b.(*restaurant.Topping),
                scope,
            )
        },
    ); err != nil {
        return err
    }
    ...
    return nil
}

...
```

`Convert_v1alpha1_Topping_To_restaurant_Topping()`生成该功能。它需要一个`v1alpha1`对象并将其转换为内部类型。

###### 注意

前面的复杂类型转换将类型转换函数转换为统一类型`func(a, b interface{}, scope conversion.Scope) error`。该方案使用后者类型，因为它可以在不使用反射的情况下调用它们。由于许多必要的分配，反思很慢。

在手写转换*conversion.go*在一定意义生成过程中优先考虑`conversion-gen`跳过一代的类型，如果它与包找到了手写功能*Convert_ source-package-basename_KindTo_ target-package-basename_Kind*转换功能命名模式。例如：

```
func Convert_v1alpha1_PizzaSpec_To_restaurant_PizzaSpec(
    in *PizzaSpec,
    out *restaurant.PizzaSpec,
    s conversion.Scope,
) error {
    ...

    return nil
}
```

在最简单的情况下，转换函数只是将值从源复制到目标对象。但是对于将`v1alpha1`披萨规范转换为内部类型的前一个示例，简单复制是不够的。我们必须调整不同的结构，实际上如下所示：

```
func Convert_v1alpha1_PizzaSpec_To_restaurant_PizzaSpec(
    in *PizzaSpec,
    out *restaurant.PizzaSpec,
    s conversion.Scope,
) error {
    idx := map[string]int{}
    for _, top := range in.Toppings {
        if i, duplicate := idx[top]; duplicate {
            out.Toppings[i].Quantity++
            continue
        }
        idx[top] = len(out.Toppings)
        out.Toppings = append(out.Toppings, restaurant.PizzaTopping{
            Name: top,
            Quantity: 1,
        })
    }

    return nil
}
```

显然，没有代码生成可以如此聪明，以至于可以预见用户在定义这些不同类型时的意图。

请注意，在转换期间，源对象绝不能变异。但这是完全正常的，并且通常出于性能原因，强烈建议在类型匹配时重用目标对象中的源的数据结构。

这非常重要，我们在警告中重申它，因为它不仅对转换的实现有影响，而且对转换的调用者和转换输出的消费者也有影响。

###### 警告

转换函数不得改变源对象，但允许输出与源共享数据结构。这意味着转换输出的使用者必须确保在原始对象不得变异的情况下不要改变对象。

例如，假设您`pod *core.Pod`在内部版本中有一个，并将其转换为`v1`as `podv1 *corev1.Pod`，并对结果进行变更`podv1`。这也可能会改变原作`pod`。如果`pod`来自informer，这是非常危险的，因为informers有一个共享缓存，并且变异`pod`使得缓存不一致。

因此，请注意转换的这种属性，并在必要时进行深层复制，以避免意外和潜在危险的突变。

虽然这种数据结构的共享会带来一些风险，但它也可以避免在许多情况下进行不必要的分配。生成的代码到目前为止，生成器比较源和目标结构，并使用Golang的`unsafe`包通过简单的类型转换将指针转换为相同内存布局的结构。因为`v1beta1`我们示例中的披萨的内部类型和类型具有相同的内存布局，所以我们得到：

```
func autoConvert_restaurant_PizzaSpec_To_v1beta1_PizzaSpec(
    in *restaurant.PizzaSpec,
    out *PizzaSpec,
    s conversion.Scope,
) error {
    out.Toppings = *(*[]PizzaTopping)(unsafe.Pointer(&in.Toppings))
    return nil
}
```

在机器语言级别，这是一个NOOP，因此它可以尽可能快。它避免在这种情况下分配切片并逐项复制`in`到`out`。

最后但并非最不重要的是，关于转换函数的第三个参数的一些说法：转换范围`conversion.Scope`。

转换范围提供对许多转换元级别值的访问。例如，它允许我们`context`通过以下方式访问传递给方案`Convert(in, out interface{}, context interface{}) error`方法的值：

```
s.Meta().Context
```

它还允许我们通过`s.Convert`或不考虑注册的转换函数来调用子类型的方案转换`s.DefaultConvert`。

但是，在大多数转换情况下，根本不需要使用范围。为简单起见，您可以忽略它的存在，直到您遇到需要比源和目标对象更多的上下文的棘手情况。

## 违约

违约是API请求生命周期中的步骤，它为传入对象（来自客户端或来自`etcd`）中的省略字段设置默认值。例如，pod有一个`restartPolicy`字段。如果用户未指定它，则默认值为`Always`。

想象一下，我们在2014年左右使用了一个非常古老的Kubernetes版本。该领域`restartPolicy`刚刚在当时的最新版本中引入了该系统。升级群集后，如果`etcd`没有该`restartPolicy`字段，则会有一个窗格。一`kubectl get pod`会从中读取旧pod，`etcd`默认代码将添加默认值`Always`。从用户的角度来看，神奇的老吊舱突然有了新的`restartPolicy`领域。

请参阅[图8-6](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregation-conversions-points)，了解Kubernetes请求管道中今天发生的默认操作。请注意，默认仅针对外部类型而非内部类型。

现在让我们看一下默认的代码。默认是由*k8s.io/apiserver*代码通过该方案启动的，类似于转换。因此，我们必须将默认函数注册到我们的自定义类型的方案中。

同样，与转换类似，大多数默认代码只是使用[`defaulter-gen`](http://bit.ly/2J108vK)二进制文件生成的。它遍历API类型并在*pkg / apis / group-name/ version/zz_generated.defaults.go中*创建默认函数。除了为子结构调用默认函数之外，默认情况下代码不会执行任何操作。

您可以通过遵循默认函数命名模式来定义自己的默认逻辑：`SetDefaults*Kind*`

```
func SetDefaultsKind(obj *Type) {
    ...
}
```

此外，与转换不同，我们必须手动调用本地方案构建器上生成的函数的注册。遗憾的是，这不是自动完成的：

```
func init() {
    localSchemeBuilder.Register(RegisterDefaults)
}
```

这里`RegisterDefaults`是在包 *pkg / apis / group-name/ version/zz_generated.defaults.go中生成的*。

对于默认代码，了解用户何时设置字段以及何时不设置字段至关重要。在许多情况下，这并不是那么清楚。

Golang对于每种类型都没有值，如果在传递的JSON或protobuf中找不到字段，则设置它们。想象一下`true`布尔字段的默认值`foo`。零值是`false`。不幸的是，不清楚是否`false`由于用户的输入而设置，或者因为`false`只是布尔值的零值。

为了避免这种情况，通常必须在Golang API类型中使用指针类型（例如，`*bool`在前面的例子中）。用户提供的`false`将导致`nil`指向`false`值的非布尔指针，并且用户提供的`true`将导致非`nil`布尔指针和`true`值。没有提供的字段导致`nil`。这可以在默认代码中检测到：

```
func SetDefaultsKind(obj *Type) {
    if obj.Foo == nil {
        x := true
        obj.Foo = &x
    }
}
```

这给出了所需的语义：“foo默认为true”。

###### 小费

这种使用指针的技巧适用于像字符串这样的基本类型。对于地图和数组，如果不识别`nil`地图/数组和空地图/数组，通常很难达到可循环性。因此，Kubernetes中用于地图和数组的大多数默认程序在两种情况下都应用默认值，即解决编码和解码错误。

## 往返测试

获得转换对，很难。往返测试是一项必不可少的工具，可以在随机测试中自动检查转换是否按计划进行，并且在转换为所有已知组版本时不会丢失数据。

往返测试通常与*install.go*文件一起放置（例如，放入*pkg / apis / restaurant / install / roundtrip_test.go*），然后从API Machinery调用往返测试函数：

```
import (
    ...
    "k8s.io/apimachinery/pkg/api/apitesting/roundtrip"
    restaurantfuzzer "github.com/programming-kubernetes/pizza-apiserver/pkg/apis/
    restaurant/fuzzer"
)

func TestRoundTripTypes(t *testing.T) {
    roundtrip.RoundTripTestForAPIGroup(t, Install, restaurantfuzzer.Funcs)
}
```

在内部，`RoundTripTestForAPIGroup`调用使用`Install`函数将API组安装到临时方案中。然后，它使用给定的模糊器在内部版本中创建随机对象，然后将它们转换为某个外部版本并返回到内部版本。生成的对象必须与原始对象等效。所有外部版本的测试都进行了数百次或数千次。

一个*模糊*器为内部类型及其子类型返回一组随机函数函数的函数。在我们的示例中，模糊器放在包*pkg / apis / restaurant / fuzzer / fuzzer.go中，*并包含spec结构的随机函数：

```
// Funcs returns the fuzzer functions for the restaurant api group.
var Funcs = func(codecs runtimeserializer.CodecFactory) []interface{} {
    return []interface{}{
        func(s *restaurant.PizzaSpec, c fuzz.Continue) {
            c.FuzzNoCustom(s) // fuzz first without calling this function again

            // avoid empty Toppings because that is defaulted
            if len(s.Toppings) == 0 {
                s.Toppings = []restaurant.PizzaTopping{
                    {"salami", 1},
                    {"mozzarella", 1},
                    {"tomato", 1},
                }
            }

            seen := map[string]bool{}
            for i := range s.Toppings {
                // make quantity strictly positive and of reasonable size
                s.Toppings[i].Quantity = 1 + c.Intn(10)

                // remove duplicates
                for {
                    if !seen[s.Toppings[i].Name] {
                        break
                    }
                    s.Toppings[i].Name = c.RandString()
                }
                seen[s.Toppings[i].Name] = true
            }
        },
    }
}
```

如果没有给出随机函数功能，底层库[*github.com/google/gofuzz*](http://bit.ly/2KJrb27)通常会尝试通过设置基类型的随机值并递归地潜入指针，结构，映射和切片来模糊对象，最终调用自定义随机函数函数它们是由开发人员提供的。

当为其中一种类型编写随机函数函数时，`c.FuzzNoCustom(s)`首先调用是很方便的。它使给定对象随机化，`s`并为子结构调用自定义函数，但不为`s`自身调用。然后，开发人员可以限制和修复随机值以使对象有效。

###### 警告

为了覆盖尽可能多的有效对象，使模糊器尽可能通用非常重要。如果模糊限制器过于严格，则测试覆盖范围会很差。在许多情况下，在Kubernetes的开发过程中，没有发现回归，因为现场的模糊器不好。

另一方面，模糊器只需要考虑验证的对象，并且是外部版本中可定义的实际对象的投影。通常，您必须以`c.FuzzNoCustom(s)`随机对象变为有效的方式限制设置的随机值。例如，如果验证将拒绝任意字符串，则持有URL的字符串不必为任意值进行往返。

我们前面的`PizzaSpec`示例首先通过以下方式调用`c.FuzzNoCustom(s)`并修复对象：

- 违反`nil`配料的情况
- 为每个顶部设置合理的数量（没有它，转换`v1alpha1`将在复杂性中爆炸，将大量数量引入字符串列表）
- 标准化顶部名称，因为我们知道披萨规格中的重复顶部将永远不会往返（对于内部类型，请注意v1alpha1类型有重复）

## 验证

传入的对象在被反序列化，默认并转换为内部版本后不久即被验证。[图8-5](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregation-conversions-figure)显示了之前如何进行验证 改变录入插件和验证 许可插件，早在实际创建或更新逻辑执行之前。

这意味着验证必须仅针对内部版本实施一次，而不是针对所有外部版本实施。这样做的好处是它显然可以节省实现工作并确保版本之间的一致性。另一方面，这意味着验证错误不涉及外部版本。实际上可以通过Kubernetes资源观察到这一点，但在实践中它并没有什么大不了的。

在本节中，我们将介绍验证函数的实现。自定义API服务器的连接 - 即从配置通用注册表的策略调用验证 - 将在下一节中介绍。换句话说，[图8-5](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregation-conversions-figure)略微误导，有利于视觉简洁性。

现在，它应该足以查看策略内部验证的切入点：

```
func (pizzaStrategy) Validate(
    ctx context.Context, obj runtime.Object,
) field.ErrorList {
    pizza := obj.(*restaurant.Pizza)
    return validation.ValidatePizza(pizza)
}
```

这将调用API组验证包中的验证功能。`Validate*Kind*(obj` `**Kind*) field.ErrorList``pkg/apis/*group*/*validation*`

验证函数返回错误列表。它们通常以相同的样式编写，将返回值附加到错误列表，同时递归潜入类型，每个结构一个验证函数：

```
// ValidatePizza validates a Pizza.
func ValidatePizza(f *restaurant.Pizza) field.ErrorList {
    allErrs := field.ErrorList{}

    errs := ValidatePizzaSpec(&f.Spec, field.NewPath("spec"))
    allErrs = append(allErrs, errs...)

    return allErrs
}

// ValidatePizzaSpec validates a PizzaSpec.
func ValidatePizzaSpec(
    s *restaurant.PizzaSpec,
    fldPath *field.Path,
) field.ErrorList {
    allErrs := field.ErrorList{}

    prevNames := map[string]bool{}
    for i := range s.Toppings {
        if s.Toppings[i].Quantity <= 0 {
            allErrs = append(allErrs, field.Invalid(
                fldPath.Child("toppings").Index(i).Child("quantity"),
                s.Toppings[i].Quantity,
                "cannot be negative or zero",
            ))
        }
        if len(s.Toppings[i].Name) == 0 {
            allErrs = append(allErrs, field.Invalid(
                fldPath.Child("toppings").Index(i).Child("name"),
                s.Toppings[i].Name,
                "cannot be empty",
            ))
        } else {
            if prevNames[s.Toppings[i].Name] {
                allErrs = append(allErrs, field.Invalid(
                    fldPath.Child("toppings").Index(i).Child("name"),
                    s.Toppings[i].Name,
                    "must be unique",
                ))
            }
            prevNames[s.Toppings[i].Name] = true
        }
    }

    return allErrs
}
```

请注意如何使用`Child`和`Index`调用字段路径。字段路径是JSON路径，在出错时打印。

通常还有一组额外的验证函数，这些函数在更新时稍有不同（前面的集用于创建）。在我们的示例API服务器中，它可能如下所示：

```
func (pizzaStrategy) ValidateUpdate(
    ctx context.Context,
    obj, old runtime.Object,
) field.ErrorList {
    objPizza := obj.(*restaurant.Pizza)
    oldPizza := old.(*restaurant.Pizza)
    return validation.ValidatePizzaUpdate(objPizza, oldPizza)
}
```

这可用于验证是否未更改只读字段。通常，更新验证也会调用正常的验证函数，并且只会添加与更新相关的检查。

###### 注意

验证是在创建时限制对象名称的正确位置 - 例如，仅限单个字，或不包含任何非字母数字字符。

实际上，任何`ObjectMeta`字段在技术上都可以以自定义方式受到限制，尽管这对于许多字段来说并不可取，因为它可能会破坏核心API机制行为。许多资源限制名称，例如，名称将显示在其他系统或需要特殊格式名称的其他上下文中。

但即使`ObjectMeta`自定义API服务器中存在特殊验证，通用注册表也会在自定义验证通过后对任何情况下的通用规则进行验证。这允许我们首先从自定义代码返回更具体的错误消息。

## 注册和战略

到目前为止，我们已经了解了如何定义和验证API类型。下一步是为这些API类型实现REST逻辑。[图8-7](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-registry-figure)显示了注册表作为API组实现的核心部分。*k8s.io/apiserver中*的通用REST请求处理程序代码调用注册表。

![资源存储和通用注册表](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0807.png)

###### 图8-7。资源存储和通用注册表

### 通用注册表

该REST逻辑通常由所谓的*通用注册表实现*。顾名思义，它是*k8s.io/apiserver/pkg/registry/rest*包中注册表接口的通用实现。

通用注册表实现“普通”资源的默认REST行为。几乎所有Kubernetes资源都使用此实现。只有少数，特别是那些没有持久化对象（例如，`SubjectAccessReview`参见[“委托授权”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-authorization)），具有自定义实现。

在*k8s.io/apiserver/pkg/registry/rest/rest.go中，*您会发现许多接口，松散地对应于HTTP谓词和某些API功能。如果接口由注册表实现，则API端点代码将提供某些REST功能。由于通用注册表实现了大多数*k8s.io/apiserver/pkg/registry/rest*接口，因此使用它的资源将支持所有默认的Kubernetes HTTP谓词（请参阅[“API服务器的HTTP接口”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch02.html#api-server-http-interface)）。以下是使用Kubernetes源代码中的GoDoc描述实现的接口列表：

- `CollectionDeleter`

  可以删除RESTful资源集合的对象

- `Creater`

  可以创建RESTful对象实例的对象

- `CreaterUpdater`

  必须支持创建和更新操作的存储对象

- `Exporter`

  一个知道如何剥离RESTful资源以进行导出的对象

- `Getter`

  可以检索命名的RESTful资源的对象

- `GracefulDeleter`

  一个知道如何传递删除选项以允许延迟删除RESTful对象的对象

- `Lister`

  可以检索与提供的字段和标签条件匹配的资源的对象

- `Patcher`

  支持get和update的存储对象

- `Scoper`

  必须指定的对象，并指示资源的范围

- `Updater`

  可以更新RESTful对象实例的对象

- `Watcher`

  应由所有希望提供通过`Watch`API 监视更改的存储对象实现的对象

我们来看看其中一个接口`Creater`：

```
// Creater is an object that can create an instance of a RESTful object.
type Creater interface {
    // New returns an empty object that can be used with Create after request
    // data has been put into it.
    // This object must be a pointer type for use with Codec.DecodeInto([]byte,
    // runtime.Object)
    New() runtime.Object

    // Create creates a new version of a resource.
    Create(
        ctx context.Context,
        obj runtime.Object,
        createValidation ValidateObjectFunc,
        options *metav1.CreateOptions,
    ) (runtime.Object, error)
}
```

实现此接口的注册表将能够创建对象。与此相反`NamedCreater`，新对象的名称来自`ObjectMeta.Name`或通过生成`ObjectMeta.GenerateName`。如果注册表实现`NamedCreater`，则名称也可以通过HTTP路径传递。

重要的是要了解实现的接口确定在将API安装到自定义API服务器时创建的API端点将支持哪些谓词。有关如何在代码中完成此操作，请参阅[“API安装”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-api-install)。

### 战略

该通用注册表可以使用称为*策略*的对象在一定程度上进行自定义。正如我们在[“验证”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-validation)看到的那样，该策略为验证等功能提供了回调。

该策略使用其GoDoc描述实现此处列出的REST策略接口（有关其定义，请参阅*k8s.io/apiserver/pkg/registry/rest*）：

- `RESTCreateStrategy`

  定义最小验证，接受的输入和名称生成行为，以创建遵循Kubernetes API约定的对象。

- `RESTDeleteStrategy`

  定义遵循Kubernetes API约定的对象的删除行为。

- `RESTGracefulDeleteStrategy`

  必须由支持正常删除的注册表实现。

- `GarbageCollectionDeleteStrategy`

  必须由默认情况下想要孤立依赖的注册表实现。

- `RESTExportStrategy`

  定义如何导出Kubernetes对象。

- `RESTUpdateStrategy`

  定义更新遵循Kubernetes API约定的对象的最小验证，接受的输入和名称生成行为。

让我们再看一下创作案例的策略：

```
type RESTCreateStrategy interface {
    runtime.ObjectTyper
    // The name generator is used when the standard GenerateName field is set.
    // The NameGenerator will be invoked prior to validation.
    names.NameGenerator

    // NamespaceScoped returns true if the object must be within a namespace.
    NamespaceScoped() bool
    // PrepareForCreate is invoked on create before validation to normalize
    // the object. For example: remove fields that are not to be persisted,
    // sort order-insensitive list fields, etc. This should not remove fields
    // whose presence would be considered a validation error.
    //
    // Often implemented as a type check and an initailization or clearing of
    // status. Clear the status because status changes are internal. External
    // callers of an api (users) should not be setting an initial status on
    // newly created objects.
    PrepareForCreate(ctx context.Context, obj runtime.Object)
    // Validate returns an ErrorList with validation errors or nil. Validate
    // is invoked after default fields in the object have been filled in
    // before the object is persisted. This method should not mutate the
    // object.
    Validate(ctx context.Context, obj runtime.Object) field.ErrorList
    // Canonicalize allows an object to be mutated into a canonical form. This
    // ensures that code that operates on these objects can rely on the common
    // form for things like comparison. Canonicalize is invoked after
    // validation has succeeded but before the object has been persisted.
    // This method may mutate the object. Often implemented as a type check or
    // empty method.
    Canonicalize(obj runtime.Object)
}
```

该嵌入`ObjectTyper`识别对象; 也就是说，它检查注册表是否支持请求中的对象。这对于创建正确类型的对象很重要（例如，通过“foo”资源，只应创建“Foo”资源）。

该`NameGenerator`显然从产生的名称`ObjectMeta.GenerateName`领域。

通过`NamespaceScoped`该策略可以返回任何支持集群范围或命名空间资源`false`或`true`。

`PrepareForCreate`在验证之前使用传入对象调用该方法。

`Validate`我们之前在[“验证”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-validation)看到的方法：它是验证函数的入口点。

最后，该`Canonicalize`方法进行归一化（例如，切片的分类）。

### 将策略连接到通用注册表

策略对象插入通用注册表实例。以下是[GitHub上](http://bit.ly/2Y0Mtyn)我们的自定义API服务器的REST存储构造函数：

```
// NewREST returns a RESTStorage object that will work against API services.
func NewREST(
    scheme *runtime.Scheme,
    optsGetter generic.RESTOptionsGetter,
) (*registry.REST, error) {
    strategy := NewStrategy(scheme)

    store := &genericregistry.Store{
        NewFunc:       func() runtime.Object { return &restaurant.Pizza{} },
        NewListFunc:   func() runtime.Object { return &restaurant.PizzaList{} },
        PredicateFunc: MatchPizza,

        DefaultQualifiedResource: restaurant.Resource("pizzas"),

        CreateStrategy: strategy,
        UpdateStrategy: strategy,
        DeleteStrategy: strategy,
    }
    options := &generic.StoreOptions{
        RESTOptions: optsGetter,
        AttrFunc: GetAttrs,
    }
    if err := store.CompleteWithOptions(options); err != nil {
        return nil, err
    }
    return &registry.REST{store}, nil
}
```

它实例化通用注册表对象`genericregistry.Store`并设置几个字段。其中许多字段都是可选字段，`store.CompleteWithOptions`如果开发人员未设置它们，则会默认显示它们。

您可以看到自定义的策略是首先通过实例化的`NewStrategy`构造函数，然后插入到注册表中`create`，`update`和`delete`运营商。

此外，`NewFunc`设置为创建新对象实例，并将该`NewListFunc`字段设置为创建新对象列表。将`PredicateFunc`选择器（可以传递给列表请求）转换为谓词函数，过滤运行时对象。

返回的对象是一个REST注册表，只是[我们](http://bit.ly/2Rxcv6G)在通用注册表对象周围的[示例项目中的](http://bit.ly/2Rxcv6G)一个简单包装器，以使类型成为我们自己的类型：

```
type REST struct {
  *genericregistry.Store
}
```

有了这个，我们就可以实例化我们的API并将其连接到自定义API服务器。在下一节中，我们将了解如何从中创建HTTP处理程序。

## API安装

激活 API服务器中的API，需要两个步骤：

1. 必须将API版本安装到API类型中 （以及转换和默认功能'）服务器方案。
2. 必须将API版本安装到服务器HTTP多路复用器（mux）中。

第一步通常使用`init`API服务器引导中集中的某个功能来完成。这是在我们的示例自定义API服务器中的*pkg / apiserver / apiserver.go*中完成的，其中定义了`serverConfig`和`CustomServer`对象（请参阅[“选项和配置模式和启动管道”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-options-config)）：

```
import (
    ...
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/runtime/serializer"

    "github.com/programming-kubernetes/pizza-apiserver/pkg/apis/restaurant/install"
)

var (
    Scheme = runtime.NewScheme()
    Codecs = serializer.NewCodecFactory(Scheme)
)
```

然后，对于应该提供的每个API组，我们调用该`Install()`函数：

```
func init() {
    install.Install(Scheme)
}
```

由于技术原因，我们还必须在计划中添加一些与发现相关的类型（这可能会在未来版本的*k8s.io/apiserver中消失*）：

```
func init() {
    // we need to add the options to empty v1
    // TODO: fix the server code to avoid this
    metav1.AddToGroupVersion(Scheme, schema.GroupVersion{Version: "v1"})
    // TODO: keep the generic API server from wanting this
    unversioned := schema.GroupVersion{Group: "", Version: "v1"}
    Scheme.AddUnversionedTypes(unversioned,
        &metav1.Status{},
        &metav1.APIVersions{},
        &metav1.APIGroupList{},
        &metav1.APIGroup{},
        &metav1.APIResourceList{},
    )
}
```

有了这个，我们在全局方案中注册了我们的API类型，包括转换和默认函数。换句话说，[图8-3](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregation-kube-apiserver_without)的空方案现在知道关于我们类型的所有内容。

第二步是将API组添加到HTTP多路复用器。嵌入到我们的`CustomServer`struct中的通用API服务器代码提供了该`InstallAPIGroup(apiGroupInfo *APIGroupInfo) error`方法，该方法为API组设置整个请求管道。

我们唯一要做的就是提供一个正确填充的`APIGroupInfo`结构。我们`New()` `(*CustomServer, error)`在`completedConfig`类型的构造函数中执行此操作：

```
// New returns a new instance of CustomServer from the given config.
func (c completedConfig) New() (*CustomServer, error) {
    genericServer, err := c.GenericConfig.New("pizza-apiserver",
      genericapiserver.NewEmptyDelegate())
    if err != nil {
        return nil, err
    }

    s := &CustomServer{
        GenericAPIServer: genericServer,
    }

    apiGroupInfo := genericapiserver.NewDefaultAPIGroupInfo(restaurant.GroupName,
      Scheme, metav1.ParameterCodec, Codecs)

    v1alpha1storage := map[string]rest.Storage{}

    pizzaRest := pizzastorage.NewREST(Scheme, c.GenericConfig.RESTOptionsGetter)
    v1alpha1storage["pizzas"] = customregistry.RESTInPeace(pizzaRest)

    toppingRest := toppingstorage.NewREST(
        Scheme, c.GenericConfig.RESTOptionsGetter,
    )
    v1alpha1storage["toppings"] = customregistry.RESTInPeace(toppingRest)

    apiGroupInfo.VersionedResourcesStorageMap["v1alpha1"] = v1alpha1storage

    v1beta1storage := map[string]rest.Storage{}

    pizzaRest = pizzastorage.NewREST(Scheme, c.GenericConfig.RESTOptionsGetter)
    v1beta1storage["pizzas"] = customregistry.RESTInPeace(pizzaRest)

    apiGroupInfo.VersionedResourcesStorageMap["v1beta1"] = v1beta1storage

    if err := s.GenericAPIServer.InstallAPIGroup(&apiGroupInfo); err != nil {
        return nil, err
    }

    return s, nil
}
```

该`APIGroupInfo`有我们在定制的通用注册表引用[“注册表和战略”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-registry)通过一个策略。对于每个组版本和资源，我们使用实现的构造函数创建注册表的实例。

该`customregistry.RESTInPeace`包装只是当注册表构造返回一个错误，恐慌帮手：

```
func RESTInPeace(storage rest.StandardStorage, err error) rest.StandardStorage {
    if err != nil {
        err = fmt.Errorf("unable to create REST storage: %v", err)
        panic(err)
    }
    return storage
}
```

注册表本身与版本无关，因为它在内部对象上运行; 请参[见图8-5](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregation-conversions-figure)。因此，我们为每个版本调用相同的注册表构造函数。

`InstallAPIGroup`最后的调用将我们引导到一个完整的自定义API服务器，可以为我们的自定义API组提供服务，[如图8-7](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-registry-figure)所示。

在完成所有这些繁重的管道之后，是时候看看我们的新API组了。为此，我们启动服务器，如[“第一次启动”中所示](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-first-start)。但这次发现信息不是空的，而是显示我们新注册的资源：

```
$ 卷曲-k的https：//本地主机：443 / API的
 {
  "kind"："APIGroupList"，
   "groups"：[
    {
      "name"："restaurant.programming-kubernetes.info"，
       "versions"：[
        {
          "groupVersion"："restaurant.programming-kubernetes.info/v1beta1"，
           "version"："v1beta1"
        }，
         {
          "groupVersion"："restaurant.programming-kubernetes.info/v1alpha1"，
           "version"："v1alpha1"
        }
      ]，
       "preferredVersion"：{
        "groupVersion"："restaurant.programming-kubernetes.info/v1beta1"，
         "version"："v1beta1"
      }，
       "serverAddressByClientCIDRs"：[
        {
          "clientCIDR"："0.0.0.0/0"，
           "serverAddress"：":443"
        }
      ]
    }
  ]
}
```

有了这个，我们几乎达到了服务餐厅API的目标。我们已经连接了API组版本，转换已到位，验证工作正常。

缺少的是检查披萨中提到的顶部确实存在于群集中。我们可以在验证函数中添加它。但传统上这些只是格式验证功能，它们是静态的，不需要运行其他资源。

相反，在接纳中实施更复杂的检查 - 下一节的主题。

## 入场

一切请求在被解组，默认并转换为内部类型后通过了一系列准入插件; 请参阅 [图8-2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregation-aggregated-apiserver)。更确切地说，请求通过两次入场：

- 变异的插件
- 验证插件

入场插件可以是变异和验证，因此可能会被录取机制调用两次：

- 一旦进入突变阶段，就会依次调用所有变异插件
- 进入验证阶段后，为所有验证插件调用（可能并行化）

更确切地说，插件可以实现变异和验证准入接口，两种情况都有两种不同的方法。

###### 注意

在分离变异和验证之前，每个插件只有一个调用。几乎不可能密切关注每个插件做了哪些突变以及哪些突变 因此，允许插入顺序有意义地导致用户的一致行为。

这种两步架构至少可确保在所有插件的最后完成验证，从而保证一致性。

除此之外链（即两个准入阶段的插件顺序）是相同的。始终为两个阶段启用或禁用插件。

入场插件，至少是本章所述的Golang中实现的插件，可以使用内部类型。相比之下，webhook允许插件（参见[“Admission Webhooks”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#admission-webhooks)）基于外部类型，并涉及到webhook和back的转换（在变异webhooks的情况下）。

但毕竟这个理论，让我们进入代码。

### 履行

准入插件是一种实现：

- 准入插件界面 `Interface`
- 可选的 `MutatingInterface`
- 可选的 `ValidatingInterface`

这三个都可以在包*k8s.io/apiserver/pkg/admission中*找到：

```
// Operation is the type of resource operation being checked for
// admission control
type Operation string.

// Operation constants
const (
    Create  Operation = "CREATE"
    Update  Operation = "UPDATE"
    Delete  Operation = "DELETE"
    Connect Operation = "CONNECT"
)

// Interface is an abstract, pluggable interface for Admission Control
// decisions.
type Interface interface {
    // Handles returns true if this admission controller can handle the given
    // operation where operation can be one of CREATE, UPDATE, DELETE, or
    // CONNECT.
    Handles(operation Operation) bool.
}

type MutationInterface interface {
    Interface

    // Admit makes an admission decision based on the request attributes.
    Admit(a Attributes, o ObjectInterfaces) (err error)
}

// ValidationInterface is an abstract, pluggable interface for Admission Control
// decisions.
type ValidationInterface interface {
    Interface

    // Validate makes an admission decision based on the request attributes.
    // It is NOT allowed to mutate.
    Validate(a Attributes, o ObjectInterfaces) (err error)
}
```

您会看到该`Interface`方法`Handles`负责对操作进行过滤。变通插件被调用via `Admit`，验证插件被调用via `Validate`。

在`ObjectInterfaces`可以访问通常由方案实施的助手：

```
type ObjectInterfaces interface {
    // GetObjectCreater is the ObjectCreater for the requested object.
    GetObjectCreater() runtime.ObjectCreater
    // GetObjectTyper is the ObjectTyper for the requested object.
    GetObjectTyper() runtime.ObjectTyper
    // GetObjectDefaulter is the ObjectDefaulter for the requested object.
    GetObjectDefaulter() runtime.ObjectDefaulter
    // GetObjectConvertor is the ObjectConvertor for the requested object.
    GetObjectConvertor() runtime.ObjectConvertor
}
```

传递给插件的属性（通过`Admit`或`Validate`两者）基本上包含从对实现高级检查很重要的请求中提取的所有信息：

```
// Attributes is an interface used by AdmissionController to get information
// about a request that is used to make an admission decision.
type Attributes interface {
    // GetName returns the name of the object as presented in the request.
    // On a CREATE operation, the client may omit name and rely on the
    // server to generate the name. If that is the case, this method will
    // return the empty string.
    GetName() string
    // GetNamespace is the namespace associated with the request (if any).
    GetNamespace() string
    // GetResource is the name of the resource being requested. This is not the
    // kind. For example: pods.
    GetResource() schema.GroupVersionResource
    // GetSubresource is the name of the subresource being requested. This is a
    // different resource, scoped to the parent resource, but it may have a
    // different kind.
    // For instance, /pods has the resource "pods" and the kind "Pod", while
    // /pods/foo/status has the resource "pods", the sub resource "status", and
    // the kind "Pod" (because status operates on pods). The binding resource for
    // a pod, though, may be /pods/foo/binding, which has resource "pods",
    // subresource "binding", and kind "Binding".
    GetSubresource() string
    // GetOperation is the operation being performed.
    GetOperation() Operation
    // IsDryRun indicates that modifications will definitely not be persisted for
    // this request. This is to prevent admission controllers with side effects
    // and a method of reconciliation from being overwhelmed.
    // However, a value of false for this does not mean that the modification will
    // be persisted, because it could still be rejected by a subsequent
    // validation step.
    IsDryRun() bool
    // GetObject is the object from the incoming request prior to default values
    // being applied.
    GetObject() runtime.Object
    // GetOldObject is the existing object. Only populated for UPDATE requests.
    GetOldObject() runtime.Object
    // GetKind is the type of object being manipulated. For example: Pod.
    GetKind() schema.GroupVersionKind
    // GetUserInfo is information about the requesting user.
    GetUserInfo() user.Info

    // AddAnnotation sets annotation according to key-value pair. The key
    // should be qualified, e.g., podsecuritypolicy.admission.k8s.io/admit-policy,
    //  where "podsecuritypolicy" is the name of the plugin, "admission.k8s.io"
    // is the name of the organization, and "admit-policy" is the key
    // name. An error is returned if the format of key is invalid. When
    // trying to overwrite annotation with a new value, an error is
    // returned. Both ValidationInterface and MutationInterface are
    // allowed to add Annotations.
    AddAnnotation(key, value string) error
}
```

在变异的情况下 - 也就是说，在`Admit(a Attributes) error`方法的实现中- 属性可以是变异的，或者更确切地说，是从`GetObject() runtime.Object`can 返回的对象。

在验证案例中，不允许变异。

这两种情况都允许调用`AddAnnotation(key, value string) error`，这允许我们添加最终在API服务器的审计输出中的注释。这有助于理解入口插件为何突变或拒绝请求。

通过`nil`从`Admit`或返回非错误来发出拒绝信号`Validate`。

###### 小费

改变准入插件以验证验证准入阶段的变化是一种很好的做法。原因是其他插件，包括webhook准入插件，可能会进一步增加更改。如果准入插件保证满足某些不变量，则只有验证步骤才能确保确实如此。

许可插件必须`Handles(operation Operation) bool`从`admission.Interface`接口实现该方法。在同一个包中有一个帮助器`Handler`。它可以通过嵌入到自定义许可插件中来实例化`NewHandler(ops ...Operation) *Handler`并实现该`Handles`方法`Handler`：

```
type CustomAdmissionPlugin struct {
    *admission.Handler
    ...
}
```

入场插件应该始终 首先检查传递的对象的GroupVersionKind：

```
func (d *PizzaToppingsPlugin) Admit(
    a admission.Attributes,
    o ObjectInterfaces,
) error {
    // we are only interested in pizzas
    if a.GetKind().GroupKind() != restaurant.Kind("Pizza") {
        return nil
    }

    ...
}
```

同样对于验证案例：

```
func (d *PizzaToppingsPlugin) Validate(
    a admission.Attributes,
    o ObjectInterfaces,
) error {
    // we are only interested in pizzas
    if a.GetKind().GroupKind() != restaurant.Kind("Pizza") {
        return nil
    }

    ...
}
```

##### 为什么API服务器管道不会预过滤对象

对于本机许可插件，没有注册机制使得支持对象的信息可用于API服务器机器，以便仅为其支持的对象调用插件。一个原因是Kubernetes API服务器中的许多插件（发明了许可机制）支持大量对象。

完整的示例许可实现如下所示：

```
// Admit ensures that the object in-flight is of kind Pizza.
// In addition checks that the toppings are known.
func (d *PizzaToppingsPlugin) Validate(
    a admission.Attributes,
    _ admission.ObjectInterfaces,
) error {
    // we are only interested in pizzas
    if a.GetKind().GroupKind() != restaurant.Kind("Pizza") {
        return nil
    }

    if !d.WaitForReady() {
        return admission.NewForbidden(a, fmt.Errorf("not yet ready"))
    }

    obj := a.GetObject()
    pizza := obj.(*restaurant.Pizza)
    for _, top := range pizza.Spec.Toppings {
        err := _, err := d.toppingLister.Get(top.Name)
        if err != nil && errors.IsNotFound(err) {
            return admission.NewForbidden(
                a,
                fmt.Errorf("unknown topping: %s", top.Name),
            )
        }
    }

    return nil
}
```

它采取以下步骤：

1. 检查传递的对象是否正确
2. 在线人准备好之前禁止访问
3. 通过提交者线人验证披萨规范中提到的每个顶部实际上作为`Topping`群集中的对象存在

请注意，列表器只是informer内存存储的接口。所以这些`Get`电话会很快。

### 注册

入场插件必须注册。这是通过一个`Register`功能完成的：

```
func Register(plugins *admission.Plugins) {
    plugins.Register(
        "PizzaTopping",
        func(config io.Reader) (admission.Interface, error) {
            return New()
        },
    )
}
```

此功能被添加到插件列表中`RecommendedOptions`（参见[“选项和配置模式和启动管道”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-options-config)）：

```
func (o *CustomServerOptions) Complete() error {
    // register admission plugins
    pizzatoppings.Register(o.RecommendedOptions.Admission.Plugins)

    // add admisison plugins to the RecommendedPluginOrder
    oldOrder := o.RecommendedOptions.Admission.RecommendedPluginOrder
    o.RecommendedOptions.Admission.RecommendedPluginOrder =
        append(oldOrder, "PizzaToppings")

    return nil
}
```

在这里，`RecommendedPluginOrder`列表预填充了通用许可插件，每个API服务器应该保持启用，以成为集群中的良好API约定公民。

最好不要触摸订单。一个原因是获得正确的订单远非微不足道。当然，如果插件行为是严格必要的，那么在列表末尾以外的位置添加自定义插件就可以了。

自定义API服务器的用户将能够以通常的许可禁用自定义许可插件链配置标志（`--disable-admission-plugins`例如）。默认情况下，我们自己的插件已启用，因为我们没有明确禁用它。

入场可以使用配置文件配置插件。为此，我们解析前面显示`io.Reader`的`Register`函数的输出。将`--admission-control-config-file`允许我们的配置文件传递给插件，像这样：

```
kind: AdmissionConfiguration
apiVersion: apiserver.k8s.io/v1alpha1
plugins:
- name: CustomAdmissionPlugin
  path: custom-admission-plugin.yaml
```

或者，我们可以进行内联配置，将所有入场配置集中在一个地方：

```
kind: AdmissionConfiguration
apiVersion: apiserver.k8s.io/v1alpha1
plugins:
- name: CustomAdmissionPlugin
  configuration:
    your-custom-yaml-inline-config
```

我们简要地提到我们的入场插件使用配料通知器来检查披萨中提到的浇头是否存在。我们还没有谈到如何将它连接到准入插件。我们现在就这样做。

### 管道资源

入场插件通常需要客户端和线人或其他资源来实现其行为。我们可以使用插件初始化器来完成此资源管道。

有一些标准插件初始化器。如果您的插件想要被他们调用，则必须使用回调方法实现某些接口（有关详细信息，请参阅*k8s.io/apiserver/pkg/admission/initializer*）：

```
// WantsExternalKubeClientSet defines a function that sets external ClientSet
// for admission plugins that need it.
type WantsExternalKubeClientSet interface {
    SetExternalKubeClientSet(kubernetes.Interface)
    admission.InitializationValidator
}

// WantsExternalKubeInformerFactory defines a function that sets InformerFactory
// for admission plugins that need it.
type WantsExternalKubeInformerFactory interface {
    SetExternalKubeInformerFactory(informers.SharedInformerFactory)
    admission.InitializationValidator
}

// WantsAuthorizer defines a function that sets Authorizer for admission
// plugins that need it.
type WantsAuthorizer interface {
    SetAuthorizer(authorizer.Authorizer)
    admission.InitializationValidator
}

// WantsScheme defines a function that accepts runtime.Scheme for admission
// plugins that need it.
type WantsScheme interface {
    SetScheme(*runtime.Scheme)
    admission.InitializationValidator
}
```

实现其中一些，并在启动期间调用插件，以便访问Kubernetes资源或API服务器全局方案。

此外，`admission.InitializationValidator`应该实现接口以最终检查插件是否正确设置：

```
// InitializationValidator holds ValidateInitialization functions, which are
// responsible for validation of initialized shared resources and should be
// implemented on admission plugins.
type InitializationValidator interface {
    ValidateInitialization() error
}
```

标准初始化程序很棒，但我们需要访问toppings informer。那么，让我们来看看如何添加我们自己的初始化器。初始化程序包括：

- 甲`Wants*`接口（例如，`WantsRestaurantInformerFactory`），其应当由承认插件来实现：

  ```
  // WantsRestaurantInformerFactory defines a function that sets
  // InformerFactory for admission plugins that need it.
  type WantsRestaurantInformerFactory interface {
      SetRestaurantInformerFactory(informers.SharedInformerFactory)
      admission.InitializationValidator
  }
  ```

- 初始化器结构，实现`admission.PluginInitializer`：

  ```
  func (i restaurantInformerPluginInitializer) Initialize(
      plugin admission.Interface,
  ) {
      if wants, ok := plugin.(WantsRestaurantInformerFactory); ok {
          wants.SetRestaurantInformerFactory(i.informers)
      }
  }
  ```

  换句话说，该`Initialize()`方法检查传递的插件是否实现了相应的自定义初始化程序`Wants*`接口。如果是这种情况，初始化程序将调用插件上的方法。

- 初始化构造函数的管道（参见[“选项和配置模式和启动管道”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-options-config)）：`RecommendedOptions.Extra\AdmissionInitializers`

  ```
  func (o *CustomServerOptions) Config() (*apiserver.Config, error) {
      ...
      o.RecommendedOptions.ExtraAdmissionInitializers =
          func(c *genericapiserver.RecommendedConfig) (
              []admission.PluginInitializer, error,
          ) {
              client, err := clientset.NewForConfig(c.LoopbackClientConfig)
              if err != nil {
                  return nil, err
              }
              informerFactory := informers.NewSharedInformerFactory(
                  client, c.LoopbackClientConfig.Timeout,
              )
              o.SharedInformerFactory = informerFactory
              return []admission.PluginInitializer{
                  custominitializer.New(informerFactory),
              }, nil
          }
  
      ...
  }
  ```

  这段代码创建了一个餐厅API组的loopback客户端，创建一个相应的informer工厂，将其存储在选项中`o`，并为其返回一个插件初始化程序。

##### 同步线人

如果在接收插件中使用告密者，在实际`Admit()`或`Validate()`功能中使用它们之前，请务必首先检查告密者是否已同步。`Forbidden`在此之前拒绝具有错误的请求。

使用[“实现”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#admission-plug-in-implementation)`Handler`描述的辅助结构，我们可以轻松地使用该函数：`Handler.WaitForReady()`

```
if !d.WaitForReady() {
    return admission.NewForbidden(
        a, fmt.Errorf("not yet ready to handle request"),
    )
}
```

要`HasSynced()`在此`WaitForReady()`方法中包含自定义的informer 方法，请将其从初始化程序实现添加到就绪函数中，如下所示：

```
func (d *PizzaToppingsPlugin) SetRestaurantInformerFactory(
f informers.SharedInformerFactory) {
    d.toppingLister = f.Restaurant().V1Alpha1().Toppings().Lister()
    d.SetReadyFunc(f.Restaurant().V1Alpha1().Toppings().Informer().HasSynced)
}
```

正如所承诺的那样，准入是完成餐厅API组的自定义API服务器的最后一步。现在我们希望看到它在行动，但不是人为地在本地机器上，而是在真正的Kubernetes集群中。这意味着我们必须查看聚合的自定义API服务器的部署。

# 部署自定义API服务器

在[“API服务”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregation-apiservices)，我们看到了该`APIService`对象，该对象用于注册自定义API服务器API组版本Kubernetes API服务器内的聚合器：

```
apiVersion: apiregistration.k8s.io/v1beta1
kind: APIService
metadata:
  name: name
spec:
  group: API-group-name
  version: API-group-version
  service:
    namespace: custom-API-server-service-namespace
    name: custom-API-server-service
  caBundle: base64-caBundle
  insecureSkipTLSVerify: bool
  groupPriorityMinimum: 2000
  versionPriority: 20
```

该`APIService`对象指向服务。通常，此服务将是普通的群集IP服务：即，使用pod将自定义API服务器部署到群集中。该服务将请求转发给pod。

让我们看看Kubernetes清单来实现这一点。

## 部署清单

我们具有以下清单（[在GitHub上的示例代码中找到](http://bit.ly/2J6CVIz)），它将成为自定义API服务的集群内部署的一部分：

- 的`APIService`两个版本`v1alpha1`：

  ```
  apiVersion: apiregistration.k8s.io/v1beta1
  kind: APIService
  metadata:
    name: v1alpha1.restaurant.programming-kubernetes.info
  spec:
    insecureSkipTLSVerify: true
    group: restaurant.programming-kubernetes.info
    groupPriorityMinimum: 1000
    versionPriority: 15
    service:
      name: api
      namespace: pizza-apiserver
    version: v1alpha1
  ```

  ......和`v1beta1`：

  ```
  apiVersion: apiregistration.k8s.io/v1beta1
  kind: APIService
  metadata:
    name: v1alpha1.restaurant.programming-kubernetes.info
  spec:
    insecureSkipTLSVerify: true
    group: restaurant.programming-kubernetes.info
    groupPriorityMinimum: 1000
    versionPriority: 15
    service:
      name: api
      namespace: pizza-apiserver
    version: v1alpha1
  ```

  请注意我们设置`insecureSkipTLSVerify`。这对于开发是可以的，但对于任何生产部署都是不适 我们将在[“证书和信任”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-certs)看到如何解决这个问题。

- 一个`Service`在集群中运行的自定义API服务器实例的面前：

  ```
  apiVersion: v1
  kind: Service
  metadata:
    name: api
    namespace: pizza-apiserver
  spec:
    ports:
    - port: 443
      protocol: TCP
      targetPort: 8443
    selector:
      apiserver: "true"
  ```

- A `Deployment`（如此处所示）或`DaemonSet`自定义API服务器pod：

  ```
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: pizza-apiserver
    namespace: pizza-apiserver
    labels:
      apiserver: "true"
  spec:
    replicas: 1
    selector:
      matchLabels:
        apiserver: "true"
    template:
      metadata:
        labels:
          apiserver: "true"
      spec:
        serviceAccountName: apiserver
        containers:
        - name: apiserver
          image: quay.io/programming-kubernetes/pizza-apiserver:latest
          imagePullPolicy: Always
          command: ["/pizza-apiserver"]
          args:
          - --etcd-servers=http://localhost:2379
          - --cert-dir=/tmp/certs
          - --secure-port=8443
          - --v=4
        - name: etcd
          image: quay.io/coreos/etcd:v3.2.24
          workingDir: /tmp
  ```

- 服务和部署的命名空间：

  ```
  apiVersion: v1
  kind: Namespace
  metadata:
    name: pizza-apiserver
  spec: {}
  ```

通常，聚合的API服务器被部署到为控制平面pod保留的一些节点，通常称为*主*节点。在这种情况下，a `DaemonSet`是每个主节点运行一个自定义API服务器实例的不错选择。这导致高可用性设置。请注意，API服务器是无状态的，这意味着它们可以轻松地多次部署，并且不需要进行领导者选举。

有了这些表现，我们差不多完成了。然而，通常情况下，安全部署需要更多考虑。您可能已经注意到pod（通过前面的部署定义）使用自定义服务帐户`apiserver`。这可以通过另一个清单创建：

```
kind: ServiceAccount
apiVersion: v1
metadata:
  name: apiserver
  namespace: pizza-apiserver
```

此服务帐户需要许多权限，我们可以通过RBAC对象添加这些权限。

## 设置RBAC

该 API服务的服务帐户首先需要一些通用权限才能参与：

- 命名空间生命周期

  只能在现有命名空间中创建对象，并在删除命名空间时删除对象。为此，API服务器必须获取，列出和查看名称空间。

- 入场webhooks

  经由配置入场网络挂接`MutatingWebhookConfigurations`和`ValidatedWebhookConfigurations`独立地从每个API服务器调用。为此，我们的自定义API服务器中的准入机制必须获取，列出和查看这些资源。

我们通过创建RBAC集群角色来配置：

```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: aggregated-apiserver-clusterrole
rules:
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "watch", "list"]
- apiGroups: ["admissionregistration.k8s.io"]
  resources: ["mutatingwebhookconfigurations", "validatingwebhookconfigurations"]
  verbs: ["get", "watch", "list"]
```

并将其绑定到我们的服务帐户`apiserver`通过`ClusterRoleBinding`：

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: pizza-apiserver-clusterrolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: aggregated-apiserver-clusterrole
subjects:
- kind: ServiceAccount
  name: apiserver
  namespace: pizza-apiserver
```

对于委派身份验证和授权，必须将服务帐户绑定到预先存在的RBAC角色`extension-apiserver-authentication-reader`：

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pizza-apiserver-auth-reader
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: extension-apiserver-authentication-reader
subjects:
- kind: ServiceAccount
  name: apiserver
  namespace: pizza-apiserver
```

和预先存在的RBAC集群角色`system:auth-delegator`：

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: pizza-apiserver:system:auth-delegator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: apiserver
  namespace: pizza-apiserver
```

## 不安全地运行自定义API服务器

现在 在所有清单和RBAC设置完成后，让我们将API服务器部署到真正的集群。

从签[出GitHub存储库](http://bit.ly/2x9C3gR)，并配置`kubectl`了`cluster-admin`特权（这是必需的，因为RBAC规则永远不会升级访问）：

```
$ cd $GOPATH/src/github.com/programming-kubernetes/pizza-apiserver
 $ cd artifacts / deployment
 $ kubectl apply -f ns.yaml # create the namespace first
$ kubectl apply -f。       # creating all manifests described above
```

现在自定义API服务器正在启动：

```
$ kubectl get pods -A
NAMESPACE NAME READY STATUS AGE
pizza-apiserver pizza-apiserver-7779f8d486-8fpgj 0/2 ContainerCreating 1s
$ # some moments later
$ kubectl get pods -A
pizza-apiserver pizza-apiserver-7779f8d486-8fpgj 2/2运行75秒
```

什么时候它正在运行，我们仔细检查Kubernetes API服务器是否进行聚合（即代理请求）。首先检查`APIService`Kubernetes API服务器是否认为我们的自定义API服务器可用：

```
$ kubectl获取apiservices v1alpha1.restaurant.programming-kubernetes.info
名称服务可用
v1alpha1.restaurant.programming-kubernetes.info pizza-apiserver / api True
```

这看起来不错。让我们尝试列出比萨饼，启用日志记录以查看是否出现问题：

```
$ kubectl获得比萨饼--v =7
...
... GET https：// localhost：58727 / apis？timeout =32s
...
... GET https：// localhost：58727 / apis / restaurant.programming-kubernetes.info /
                                v1alpha1？超时=32秒
...
... GET https：// localhost：58727 / apis / restaurant.programming-kubernetes.info /
                                v1beta1 / namespaces / default / pizzas？limit =500
...请求标题：
...接受：application / json ;as=表;v=v1beta1 ;g=meta.k8s.io，application / json
...用户代理：kubectl / v1.15.0 (darwin / amd64 )kubernetes / f873d2a
...响应状态：200以6毫秒为单位确定
找不到资源。
```

这看起来非常好。我们看到`kubectl`查询发现信息以找出披萨是什么。它查询*restaurant.programming-kubernetes.info/v1beta1* API列出比萨饼。不出所料，还没有。但我们当然可以改变：

```
$ cd../examples
 $ # install toppings first
$ ls topping * |xargs -n 1kubectl create -f
 $ kubectl create -f pizza-margherita.yaml
pizza.restaurant.programming-kubernetes.info/margherita创建
$ kubectl得到披萨-o yaml margherita
apiVersion：restaurant.programming-kubernetes.info/v1beta1
亲切：披萨
元数据：
  creationTimestamp： "2019-05-05T13:39:52Z"
  名称：玛格丽塔
  命名空间：默认
  resourceVersion： "6"
  比萨饼/雏菊
  uid：42ab6e88-6f3b-11e9-8270-0e37170891d3
规格：
  配料：
  - 名称：莫扎里拉
    数量：1
  - 名字：番茄
    数量：1
状态： {}
```

这看起来很棒。但是玛格丽塔披萨很容易。让我们尝试通过创建一个没有列出任何浇头的空比萨来默认行动：

```
apiVersion：restaurant.programming-kubernetes.info/v1alpha1
亲切：披萨
元数据：
  名称：萨拉米香肠
规格：
```

我们的默认应该把它变成萨拉米香肠披萨萨拉米香肠。我们试试吧：

```
$ kubectl create -f empty-pizza.yaml
pizza.restaurant.programming-kubernetes.info/salami创建
$ kubectl得到披萨-o yaml salami
apiVersion：restaurant.programming-kubernetes.info/v1beta1
亲切：披萨
元数据：
  creationTimestamp： "2019-05-05T13:42:42Z"
  名称：萨拉米香肠
  命名空间：默认
  resourceVersion： "8"
  比萨饼/萨拉米香肠
  uid：a7cb7af2-6f3b-11e9-8270-0e37170891d3
规格：
  配料：
  - 名称：萨拉米香肠
    数量：1
  - 名称：莫扎里拉
    数量：1
  - 名字：番茄
    数量：1
状态： {}
```

这看起来像一个美味的萨拉米香肠披萨。

现在让我们检查一下我们的自定义插件是否正常工作。我们先删除所有比萨饼和浇头，然后尝试重新制作比萨饼：

```
$ kubectl删除比萨饼 - 全部
pizza.restaurant.programming-kubernetes.info "margherita"已删除
pizza.restaurant.programming-kubernetes.info "salami"删除了
 $ kubectl删除顶部--all
topping.restaurant.programming-kubernetes.info "mozzarella"已删除
topping.restaurant.programming-kubernetes.info "salami"已删除
topping.restaurant.programming-kubernetes.info "tomato"已删除
 $ kubectl create -f pizza-margherita.yaml
来自服务器的错误(禁止)：创建时出错"pizza-margherita.yaml"：
 pizzas.restaurant.programming-kubernetes.info "margherita"被禁止：
   未知的馅料：莫扎里拉奶酪
```

没有没有马苏里拉奶酪的玛格丽特，就像任何一家意大利餐厅一样。

看起来我们已经完成了我们在[“示例：比萨餐厅”中描述的内容](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregation-example)。但并不完全。安全。再次。我们没有照顾到合适的证书。恶意比萨卖家可能会尝试在我们的用户和自定义API服务器之间进行切换，因为Kubernetes API服务器只接受任何服务证书而不检查它们。我们来解决这个问题。

## 证书和信托

该`APIService`对象包含该`caBundle`字段。这配置如何聚合器（在Kubernetes API服务器内）信任自定义API服务器。此CA捆绑包包含用于验证聚合API服务器是否具有其声明的标识的证书（和中间证书）。对于任何严重部署，请将相应的CA捆绑包放入此字段中。

###### 警告

虽然`insecureSkipTLSVerify`允许在`APIService`禁用认证验证时使用，但在生产设置中使用它是一个坏主意。Kubernetes API服务器将请求发送到受信任的聚合API服务器。设置`insecureSkipTLSVerify`为`true`表示任何其他actor可以声称是聚合API服务器。这显然是不安全的，不应该在生产环境中使用。

[“委托身份验证和信任”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-authentication)描述了从自定义API服务器到Kubernetes API服务器的反向信任及其对请求的预[身份验证](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-authentication)。我们不需要做任何额外的事情。

回到披萨示例：为了确保安全，我们需要服务证书和部署中自定义API服务器的密钥。我们将两者放入一个`serving-cert`秘密并将其安装到*/var/run/apiserver/serving-cert/tls.{crt,key}*的吊舱中。然后我们使用*tls.crt*文件作为CA的`APIService`。这可以[在GitHub上的示例代码中找到](http://bit.ly/2XxtJWP)。

证书生成逻辑在[Makefile中](http://bit.ly/2KGn0nw)编写脚本。

请注意，在实际情况中，我们可能会有某种类型的集群或公司CA，我们可以插入其中`APIService`。

要查看它的实际效果，请先从新群集开始，或者只是重复使用前一个群集并应用新的安全清单：

```
$ cd../deployment-secure
 $ make
openssl req -new -x509 -subj "/CN=api.pizza-apiserver.svc"
  -nodes -newkey rsa：4096
  -keyout tls.key -out tls.crt -days 365
生成4096一点RSA私钥
......................++
................................................................++
写新的私钥 'tls.key'
...
$ ls * .yaml |xargs -n 1kubectl apply -f
clusterrolebinding.rbac.authorization.k8s.io/pizza-apiserver:system:auth-delegator不变
rolebinding.rbac.authorization.k8s.io/pizza-apiserver-auth-reader不变
已配置deployment.apps / pizza-apiserver
namespace / pizza-apiserver不变
clusterrolebinding.rbac.authorization.k8s.io/pizza-apiserver-clusterrolebinding不变
clusterrole.rbac.authorization.k8s.io/aggregated-apiserver-clusterrole不变
serviceaccount / apiserver不变
service / api不变
secret / serving-cert创建
apiservice.apiregistration.k8s.io/v1alpha1.restaurant.programming-kubernetes.info已配置
apiservice.apiregistration.k8s.io/v1beta1.restaurant.programming-kubernetes.info已配置
```

请注意`CN=api.pizza-apiserver.svc`证书中正确的通用名称。Kubernetes API服务器将请求代理到*api / pizza-apiserver*服务，因此必须将其DNS名称放入证书中。

我们仔细检查我们是否确实禁用了以下`insecureSkipTLSVerify`标志`APIService`：

```
$ kubectl获取apiservices v1alpha1.restaurant.programming-kubernetes.info -o yaml
apiVersion：apiregistration.k8s.io/v1
kind：APIService
元数据：
  name: v1alpha1.restaurant.programming-kubernetes.info
  ...
规格：
  caBundle：LS0tLS1C ......
  group：restaurant.programming-kubernetes.info
  groupPriorityMinimum：1000
  服务：
    名称：api
    命名空间：pizza-apiserver
  版本：v1alpha1
  versionPriority：15
状态：
  条件：
  - 最后的过渡时间： "2019-05-05T14:07:07Z"
    消息：所有检查都已通过
    理由：通过
    status "True"
    type::可用
伪影/ deploymen
```

这看起来像预期的那样：`insecureSkipTLSVerify`已经消失，`caBundle`字段中填充了我们证书的base64值并且：该服务仍然可用。

现在让我们看看是否`kubectl`仍然可以查询API：

```
$ kubectl获得比萨饼
找不到资源。
$ cd../examples
 $ ls topping * |xargs -n 1kubectl create -f
topping.restaurant.programming-kubernetes.info/mozzarella创建
topping.restaurant.programming-kubernetes.info/salami创建
topping.restaurant.programming-kubernetes.info/tomato created
$ kubectl create -f pizza-margherita.yaml
pizza.restaurant.programming-kubernetes.info/margherita创建
```

玛格丽塔披萨回来了。这次它完全安全。恶意披萨卖家没有机会开始中间人攻击。Buon appetito！

## 分享etcd

聚合API使用的服务器`RecommendOptions`（请参阅[“选项和配置模式和启动管道”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregated-apiserver-development-options-config)）`etcd`用于存储。这意味着任何自定义API服务器的部署都需要`etcd`群集可用。

该集群可以是集群内 - 例如，使用[`etcd`运营商](http://bit.ly/2JTz8SK)部署。此运算符允许我们以`etcd`声明方式启动和管理集群。运营商将进行更新，上下扩展和备份。这大大减少了操作开销。

或者，`etcd`群集控制平面的（即，`kube-apiserver`）可以使用。取决于环境 - 自我部署，内部部署或托管服务，如Google容器引擎（GKE） - 这可能是可行的，或者它可能是不可能的，因为用户根本无法访问集群（如果是这样的话）与GKE）。在可行的情况下，自定义API服务器必须使用与Kubernetes API服务器或其他`etcd`使用者使用的密钥路径不同的密钥路径。在我们的示例自定义API服务器中，它看起来像这样：

```
const defaultEtcdPathPrefix =
    "/registry/pizza-apiserver.programming-kubernetes.github.com"

func NewCustomServerOptions() *CustomServerOptions {
    o := &CustomServerOptions{
        RecommendedOptions: genericoptions.NewRecommendedOptions(
            defaultEtcdPathPrefix,
            ...
        ),
    }

    return o
}
```

此`etcd`路径前缀与使用不同组API名称的Kubernetes API服务器路径不同。

最后但并非最不重要的，`etcd`可以代理。项目[etcdproxy-controller](http://bit.ly/2Na2VrN)使用运算符模式实现此机制; 也就是说，`etcd`代理可以自动部署到集群并使用`EtcdProxy`对象进行配置。

该`etcd`代理会自动完成键映射，所以可以保证`etcd`关键前缀不会发生冲突。这使我们可以共享`etcd`多个聚合API服务器的集群，而无需担心一个聚合API服务器读取或更改另一个聚合API服务器的数据。这将提高`etcd`需要共享群集的环境中的安全性，例如，由于资源限制或避免操作开销。

根据上下文，必须选择其中一个选项。最后，聚合API服务器当然也可以使用其他存储后端，至少在理论上，因为它需要大量自定义代码来实现*k8s.io/apiserver*存储接口。

# 摘要

这是一个非常大的章节，你把它做到了最后。您已经掌握了很多关于Kubernetes中API的背景知识以及它们的实现方式。

我们看到了自定义API服务器的聚合如何适应Kubernetes集群的体系结构。我们了解了自定义API服务器如何从Kubernetes API服务器接收代理请求。我们已经了解了Kubernetes API服务器如何预先验证这些请求，以及如何使用外部版本和内部版本实现API组。我们学习了如何将对象解码为Golang结构，它们如何被默认，它们如何转换为内部类型，以及它们如何通过许可和验证并最终到达注册表。我们看到了如何将策略插入通用注册表以实现“普通”Kubernetes类REST资源，如何添加自定义许可以及如何使用自定义初始化程序配置自定义许可插件。`APIServices`。我们了解了如何配置RBAC规则以允许自定义API服务器完成其工作。我们讨论了如何`kubectl`查询API组。最后，我们学习了如何使用证书保护与自定义API服务器的连接。

这很多。现在，您可以更好地了解Kubernetes中的API以及它们的实现方式，并希望您有动力去做以下一项或多项：

- 实现您自己的自定义API服务器
- 了解Kubernetes的内部运作方式
- 将来有助于Kubernetes

我们希望您已经找到了一个很好的起点。

[1正常](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#idm46336853170760-marker)删除意味着客户端可以通过正常删除期间作为删除调用的一部分。实际的删除是由控制器`kubelet`通过强制删除异步完成（对pod执行的操作）。这样，豆荚有时间干净地关闭。

[2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#idm46336853165528-marker) Kubernetes使用同居来将资源（例如，从`extensions/v1beta1`API组部署）迁移到特定于主题的API组（例如，`apps/v1`）。CRD没有共享存储的概念。

[3](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#idm46336853099640-marker)我们将在[第9章中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#ch_advanced-topics)看到，最新Kubernetes版本中提供的CRD转换和录入webhooks也允许我们将这些功能添加到CRD中。

[4](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#idm46336852602264-marker) PaaS代表平台即服务。