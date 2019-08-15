# 第1章简介

编程Kubernetes对不同的人来说意味着不同的东西。在本章中，我们将首先确定本书的范围和重点。此外，我们将分享关于我们正在运营的环境以及您需要提供什么样的假设，理想情况下，从本书中获益最多。我们将通过编程Kubernetes，Kubernetes本地应用程序是什么来定义我们的意思，并通过查看具体示例，了解它们的特征。我们将讨论控制器和操作器的基础知识，以及事件驱动的Kubernetes原理如何控制平面功能。准备？我们来吧。

# 编程Kubernetes意味着什么？

我们假设您可以访问正在运行的Kubernetes集群，例如Amazon EKS，Microsoft AKS，Google GKE或其中一个OpenShift产品。

###### 小费

You将花费大量时间在笔记本电脑或桌面环境中进行*本地*开发; 也就是说，您正在开发的Kubernetes集群是本地的，而不是云或数据中心。在本地开发时，您可以使用多种选项。根据您的操作系统和其他首选项，您可以选择一个（或甚至更多）以下解决方案在本地运行Kubernetes：[kind](https://kind.sigs.k8s.io/)，[k3d](http://bit.ly/2Ja1LaH)或[Docker Desktop](https://dockr.ly/2PTJVLL)。[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#idm46336877072840)

我们还假设您是Go程序员 - 也就是说，您具有Go编程语言的经验或至少基本熟悉。现在是一个好时机，如果这些假设中的任何一个不适用于你，那么训练：对于Go，我们推荐Alan AA Donovan和Brian W. Kernighan（Addison-Wesley）[*的Go Go Programming Language*](https://www.gopl.io/)和Katherine的[*Go并发*](http://bit.ly/2tdCt5j) Cox-Buday（奥莱利）。对于 Kubernetes，查看以下一本或多本书：

- 曼宁的[*行动总督*](http://bit.ly/2Tb8Ydo)
- [*Kubernetes：Up and Running*，第2版](https://oreil.ly/2SaANU4)，Kelsey Hightower等。（O'Reilly）的
- [*与*](https://oreil.ly/2BaE1iq) John Arundel和Justin[ *Domingus*](https://oreil.ly/2BaE1iq)（O'Reilly）[*合作的Kubernetes*](https://oreil.ly/2BaE1iq)的[ *Cloud Native DevOps*](https://oreil.ly/2BaE1iq)
- Brendan Burns和Craig Tracey[*管理Kubernetes*](https://oreil.ly/2wtHcAm)（O'Reilly）
- SébastienGoasguen和Michael Hausenblas（O'Reilly）的[ *Kubernetes Cookbook*](http://bit.ly/2FTgJzk)

###### 注意

为什么我们专注于Go中的Kubernetes编程？好吧，类比可能在这里有用：Unix是用C编程语言编写的，如果你想为Unix编写应用程序或工具，你会默认为C.另外，为了扩展和定制Unix - 即使你是使用C以外的语言 - 您至少需要能够阅读C.

现在，Kubernetes和许多相关的云原生技术，从容器运行时到监控，如Prometheus，都是用Go编写的。我们相信大多数原生应用程序都是基于Go的，因此我们在本书中专注于它。如果您更喜欢其他语言，请关注[kubernetes-client](http://bit.ly/2xfSrfT) GitHub组织。它可能会继续以您最喜欢的编程语言包含客户端。

通过在本书的上下文中，“编程Kubernetes”我们的意思如下：您将开发一个Kubernetes本机应用程序，它直接与API服务器交互，查询资源状态和/或更新其状态。我们并不意味着运行现成的应用程序，如WordPress或Rocket Chat或您最喜欢的企业CRM系统，通常称为*商用现成*（COTS）应用程序。此外，在[第7章中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch07.html#ch_shipping)，我们并没有真正关注运营问题，而是主要关注开发和测试阶段。所以，简而言之，这本书是关于发展的真正的云原生应用程序。[图1-1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#apps-on-kube)可能会帮助您更好地吸收它。

![在Kubernetes上运行的不同类型的应用程序](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0101.png)

###### 图1-1。在Kubernetes上运行的不同类型的应用程序

如 你可以看到，有你可以使用的不同风格：

1. 拿一个像Rocket Chat这样的COTS并在Kubernetes上运行它。应用程序本身并不知道它在Kubernetes上运行，通常不必。Kubernetes控制应用程序的生命周期 - 查找节点以运行，提取图像，启动容器，执行运行状况检查，装载卷等等 - 就是这样。
2. 拿一个定制的应用程序，你从头开始编写的东西，无论是否考虑到Kubernetes作为运行时环境，并在Kubernetes上运行它。适用与COTS相同的操作方式。
3. 我们在本书中关注的案例是云本机或Kubernetes本机应用程序，它完全了解它在Kubernetes上运行并在某种程度上利用Kubernetes API和资源。

该 您根据Kubernetes API支付的价格得到回报：一方面您获得了可移植性，因为您的应用程序现在可以在任何环境中运行（从内部部署到任何公共云提供商），另一方面您可以从中受益Kubernetes提供的清洁，声明机制。

让我们现在转到一个具体的例子。

# 一个激励的例子

至演示了Kubernetes原生应用程序的强大功能，让我们假设您要实现`at`- 也就是说，在给定时间[安排执行命令](http://bit.ly/2L4VqzU)。

我们称这个[`cnat`](http://bit.ly/2RpHhON)或云原生`at`，它的工作原理如下。假设你想`echo "Kubernetes native rocks!"`在2019年7月3日凌晨2点执行命令。这是你要做的事情`cnat`：

```
$ cat cnat-rocks-example.yaml
apiVersion：cnat.programming-kubernetes.info/v1alpha1
亲切的：在
元数据：
  名称：cnrex
规格：
  时间表： "2019-07-03T02:00:00Z"
  容器：
  - 名称：shell
    图像：centos：7
    command:
    - "bin/bash"
    - "-c"
    - echo "Kubernetes native rocks!"

$ kubectl apply -f cnat-rocks-example.yaml
cnat.programming-kubernetes.info/cnrex创建
```

在幕后，涉及以下组件：

- 调用的自定义资源`cnat.programming-kubernetes.info/cnrex`，表示计划。
- 控制器在正确的时间执行调度的命令。

此外，`kubectl`CLI UX 的插件很有用，允许通过像`kubectl` `at` `"02:00 Jul 3"` `echo``"Kubernetes native rocks!"`We 这样的命令进行简单处理不会在本书中写这个，但您可以参考[Kubernetes文档获取说明](http://bit.ly/2J1dPuN)。

在整本书中，我们将使用此示例来讨论Kubernetes的各个方面，其内部工作以及如何扩展它。

对于章更高级的例子[8](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#ch_custom-api-servers)和[9](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#ch_advanced-topics)，我们将模拟集群与比萨比萨饼店和一流的对象。有关详细信息，请参阅[“示例：比萨餐厅”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#aggregation-example)。

# 扩展模式

Kubernetes是一个功能强大且内在可扩展的系统。通常，有多种方法可以自定义和/或扩展Kubernetes：使用[配置文件和](http://bit.ly/2KteqbA)控制平面组件（如`kubelet`Kubernetes API服务器）的标志，以及通过许多已定义的扩展点：

- 所谓的[云提供商](http://bit.ly/2FpHInw)，传统上是树中的控制器管理器的一部分。从1.11开始，Kubernetes通过提供[与云集成](http://bit.ly/2WWlcxk)的[自定义`cloud-controller-manager`流程](http://bit.ly/2WWlcxk)，使树外开发成为可能。云提供商允许使用特定于云提供商的工具，如负载平衡器或虚拟机（VM）。
- `kubelet`用于[网络](http://bit.ly/2L1tPzm)，[设备](http://bit.ly/2XthLgM)（如GPU），[存储](http://bit.ly/2x7Unaa)和[容器运行时的](http://bit.ly/2Zzh1Eq)二进制插件。
- 二进制`kubectl` [插件](http://bit.ly/2FmH7mu)。
- API服务器中的访问扩展，例如[带有webhooks](http://bit.ly/2DwR2Y3)的[动态准入控制](http://bit.ly/2DwR2Y3)（参见[第9章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch09.html#ch_advanced-topics)）。
- 自定义资源（请参阅[第4章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#ch_crds)）和自定义控制器; 请参阅以下部分。
- 自定义API服务器（请参阅[第8章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch08.html#ch_custom-api-servers)）。
- 调度程序扩展，例如使用[webhook](http://bit.ly/2xcg4FL)来实现您自己的调度决策。
- 使用webhook进行[身份验证](http://bit.ly/2Oh6DPS)。

在本书的上下文中，我们将重点关注自定义资源，控制器，webhook和自定义API服务器，以及Kubernetes [扩展模式](http://bit.ly/2L2SJ1C)。如果您对其他扩展点感兴趣，例如存储或网络插件，请查看[官方文档](http://bit.ly/2Y0L1J9)。

现在您已经对Kubernetes扩展模式和本书的范围有了基本的了解，让我们转到Kubernetes控制平面的核心，看看我们如何扩展它。

# 控制器和操作员

在 在本节中，您将了解Kubernetes中的控制器和操作员以及它们的工作原理。

根据[Kubernetes术语表](http://bit.ly/2IWGlxz)，*控制器*实现一个控制循环，通过API服务器观察集群的共享状态，并进行更改以尝试将当前状态移至所需状态。

在我们深入了解控制器的内部工作之前，让我们来定义我们的术语：

- 控制器可以对核心资源（例如部署或服务）执行操作，这些资源通常是控制平面中[Kubernetes控制器管理器的](http://bit.ly/2WUAEVy)一部分，或者可以监视和操作用户定义的自定义资源。
- 操作员是编码器，它编码一些操作知识，例如应用程序生命周期管理，以及[第4章中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#ch_crds)定义的自定义资源。

当然，鉴于后者的概念是基于前者，我们首先考虑控制器，然后讨论操作员的更专业的情况。

## 控制回路

在 一般来说，控制循环如下所示：

1. 阅读资源状态，最好是事件驱动（使用手表，如[第3章所述](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#ch_client-go)）。有关详细信息，请参阅[“事件”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#controller-events)和[“边缘与电平驱动的触发器”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#edge-vs-level)。
2. 更改群集或群集外部世界中的对象的状态。例如，启动pod，创建网络端点或查询云API。有关详细信息，请参阅[“更改群集对象或外部世界”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#controller-change-world)。
3. 通过API服务器更新步骤1中资源的状态`etcd`。有关详细信息，请参阅[“乐观并发”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#optimistic-concurrency)。
4. 重复循环; 回到第1步。

无论您的控制器有多复杂或简单，这三个步骤 - 读取资源状态˃更改世界˃更新资源状态 - 保持不变。让我们深入探讨一下如何在Kubernetes控制器中实现这些步骤。控制回路[如图1-2所示](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#controller-loop-overview)，其中显示了典型的运动部件，控制器的主回路位于中间。该主循环在控制器进程内持续运行。此过程通常在群集中的pod中运行。

![Kubernetes控制循环](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0102.png)

###### 图1-2。Kubernetes控制循环

从架构的角度来看，控制器通常使用以下数据结构（如[第3章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#ch_client-go)详细讨论）：

- 线人

  线人以可扩展和可持续的方式观察所需的资源状态。它们还实现了重新同步机制（请参阅[“信息和缓存”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#informers)以获取详细信息），这些机制强制执行定期协调，通常用于确保群集状态和缓存在内存中的假定状态不会漂移（例如，由于错误或网络问题） ）。

- 工作队列

  基本上，一项工作queue是一个组件，可由事件处理程序用于处理状态更改的排队并帮助实现重试。在`client-go`这个功能是通过所提供的[*工作队列*包](http://bit.ly/2x7zyeK)（见[“工作队列”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#workqueue)）。在更新世界或写入状态（循环中的步骤2和3）时，如果出现错误，可以重新排队资源，或者仅仅因为我们必须在一段时间后因其他原因重新考虑资源。

对于关于Kubernetes作为声明引擎和状态转换的更正式的讨论，请阅读Andrew Chen和Dominik Tornow [撰写的“Kubernetes的力学”](http://bit.ly/2IV2lcb)。

现在让我们仔细看看控制循环，从Kubernetes事件驱动架构开始。

## 活动

该Kubernetes控制平面大量使用事件和松耦合组件的原理。其他分布式系统使用远程过程调用（RPC）以触发行为。Kubernetes没有。Kubernetes控制器监视API服务器中Kubernetes对象的更改：添加，更新和删除。当发生这样的事件时，控制器执行其业务逻辑。

例如，为了通过部署启动pod，许多控制器和其他控制平面组件一起工作：

1. 部署控制器（内部`kube-controller-manager`）通知（通过部署通知程序）用户创建部署。它在其业务逻辑中创建副本集。
2. 副本集控制器（再次在内部`kube-controller-manager`）通知（通过副本集提交者）新副本集并随后运行其业务逻辑，从而创建pod对象。
3. 调度程序（在`kube-scheduler`二进制文件内） - 它也是一个控制器 - 通过一个空`spec.nodeName`字段通知pod（通过pod informer）。其业务逻辑将pod放入其调度队列中。
4. 与此同时- `kubelet`另一个控制器 - 注意到新的pod（通过其pod informer）。但是新pod的`spec.nodeName`字段为空，因此与`kubelet`节点名称不匹配。它忽略了pod并重新进入睡眠状态（直到下一个事件）。
5. 调度程序将pod从工作队列中取出，并通过更新`spec.nodeName`pod中的字段并将其写入API服务器，将其调度到具有足够可用资源的节点。
6. 将`kubelet`再次由于吊舱更新事件唤醒。它再次将其`spec.nodeName`与自己的节点名称进行比较。名称匹配，因此`kubelet`启动容器的容器并通过将此信息写入容器状态并返回API服务器来报告容器已启动。
7. 副本集控制器注意到已更改的窗格但无关。
8. 最终pod终止。该`kubelet`会注意到这一点，您可以通过API服务器荚对象，并设置在吊舱的状态中的“结束”状态，并把它写回API服务器。
9. 副本集控制器注意到已终止的pod并确定必须替换此pod。它删除API服务器上已终止的pod并创建一个新的pod。
10. 等等。

如您所见，许多独立控制循环仅通过API服务器上的对象更改以及这些更改通过触发器触发的事件进行通信。

这些事件从API服务器发送到控制器内的通知器手表（参见[“手表”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#client-go-watches)） - 也就是手表事件的流媒体连接。所有这些对用户来说几乎是不可见的。甚至API服务器审计机制也不会使这些事件可见; 只有对象更新可见。但是，当控制器对事件做出反应时，控制器通常会使用日志输出。

##### 观看事件与事件对象

看事件和`Event`Kubernetes中的顶级对象有两个不同的东西：

- 监视事件通过API服务器和控制器之间的流HTTP连接发送，以驱动线程。
- 顶级`Event`对象是类似于pod，部署或服务的资源，具有特殊属性，它具有一小时的生存时间，然后自动清除`etcd`。

`Event`对象仅仅是用户可见的日志记录机制。许多控制器创建这些事件，以便将其业务逻辑的各个方面传达给用户。例如，`kubelet`报告pod的生命周期事件（即，当容器启动，重新启动和终止时）。

You可以列出自己使用的集群中发生的第二类事件`kubectl`。通过发出以下命令，您可以看到`kube-system`命名空间中发生了什么：

```
$ kubectl -n kube-system get events
LAST SEEN   FIRST SEEN   COUNT  NAME                                              KIND
3m          3m           1      kube-controller-manager-master.15932b6faba8e5ad   Pod
3m          3m           1      kube-apiserver-master.15932b6fa3f3fbbc            Pod
3m          3m           1      etcd-master.15932b6fa8a9a776                      Pod
…
2m          3m           2      weave-net-7nvnf.15932b73e61f5bc6                  Pod
2m          3m           2      weave-net-7nvnf.15932b73efeec0b3                  Pod
2m          3m           2      weave-net-7nvnf.15932b73e8f7d318                  Pod
```

如果如果您想了解有关活动的更多信息，请阅读Michael Gasch的博客文章[“活动，Kubernetes的DNA”](http://bit.ly/2MZwbl6)，在那里他提供了更多背景和示例。

## 边缘与水平驱动的触发器

让我们 退一步，更加抽象地看看我们如何构建在控制器中实现的业务逻辑，以及为什么Kubernetes选择使用事件（即状态变化）来驱动其逻辑。

有两个原则选项 检测状态变化（事件本身）：

- 边缘驱动的触发器

  在 发生状态更改的时间点，触发处理程序 - 例如，从无pod到pod运行。

- 水平驱动的触发器

  检查状态 定期间隔，如果满足某些条件（例如，pod运行），则触发处理程序。

该后者是一种民意调查。它不能很好地适应对象的数量，并且控制器注意到更改的延迟取决于轮询的间隔以及API服务器可以回答的速度。如[“事件”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#controller-events)所述，涉及许多异步控制器，结果是需要很长时间来实现用户期望的系统。

对于许多对象，前一种选择效率更高。延迟主要取决于控制器处理事件中的工作线程数。因此，Kubernetes基于事件（即边缘驱动的触发器）。

在在Kubernetes控制平面上，许多组件在API服务器上更改对象，每次更改都会导致事件（即边缘）。我们将这些组件称为*事件源*或*事件生成器*。另一方面，在控制器的上下文中，我们对消耗事件感兴趣 - 即何时以及如何对事件做出反应（通过线人）。

在分布式系统中，有许多actor并行运行，并且事件以任何顺序异步进入。当我们有一个错误的控制器逻辑，一些稍微错误的状态机或外部服务失败时，很容易丢失事件，因为我们不完全处理它们。因此，我们必须深入研究如何 应对错误。

在[图1-3中，](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#edge-vs-level-overview)您可以看到不同的工作策略：

1. 仅边缘驱动逻辑的示例，其中可能错过第二状态改变。
2. 边缘触发逻辑的一个示例，它在处理事件时始终获得最新状态（即级别）。换句话说，逻辑是边沿触发但是水平驱动。
3. 具有附加重新同步的边缘触发的电平驱动逻辑的示例。

![触发选项（边缘与级别）](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0103.png)

###### 图1-3。触发选项（边缘驱动与电平驱动）

策略1无法很好地应对错过的事件，无论是因为破坏的网络使其丢失事件，还是因为控制器本身存在错误或某些外部云API已关闭。想象一下，副本集控制器只有在终止时才会替换pod。缺少事件意味着副本集将始终以较少的pod运行，因为它永远不会协调整个状态。

策略2在收到另一个事件时从这些问题中恢复，因为它基于集群中的最新状态实现其逻辑。对于副本集控制器，它始终将指定的副本计数与群集中正在运行的pod进行比较。当它丢失事件时，它将在下次收到pod更新时替换所有丢失的pod。

策略3添加连续重新同步（例如，每五分钟）。如果没有pod事件进入，它将至少每五分钟协调一次，即使应用程序运行非常稳定并且不会导致许多pod事件。

鉴于纯边缘驱动触发器的挑战，Kubernetes控制器通常实施第三种策略。

如果你想了解更多关于触发器的起源以及在Kubernetes中使用和解进行关卡触发的动机，请阅读James Bowes的文章[“Kubernetes中的关卡触发和调和”](http://bit.ly/2FmLLAW)。

最后讨论了检测外部变化并对其作出反应的不同抽象方法。[图1-2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#controller-loop-overview)控制循环的下一步是更改集群对象或按照规范更改外部世界。我们现在来看看。

## 更改群集对象或外部世界

在在这个阶段，控制器改变它正在监督的对象的状态。例如，[控制器管理](http://bit.ly/2WUAEVy)`ReplicaSet`器中的[控制器](http://bit.ly/2WUAEVy)正在监督pod。在每个事件（边缘触发）上，它将观察其pod的当前状态，并将其与所需状态（水平驱动）进行比较。

由于更改资源状态的行为是特定于域或任务的，因此我们几乎无法提供指导。相反，我们将继续关注`ReplicaSet`我们之前介绍的控制器。`ReplicaSet`s用于部署，相应控制器的底线是：维护用户定义数量的相同pod副本。也就是说，如果播客数量少于用户指定的播放次数（例如，因为播放器已经死亡或者副本值已经增加），控制器将启动新播客。但是，如果有太多的pod，它会选择一些用于终止。控制器的整个业务逻辑经由可用[的*replica_set.go*包](http://bit.ly/2L4eKxa)，和与状态改变（编辑为清楚起见）转到代码交易以下摘录：

```
// manageReplicas checks and updates replicas for the given ReplicaSet.
// It does NOT modify <filteredPods>.
// It will requeue the replica set in case of an error while creating/deleting pods.
func (rsc *ReplicaSetController) manageReplicas(
	filteredPods []*v1.Pod, rs *apps.ReplicaSet,
) error {
    diff := len(filteredPods) - int(*(rs.Spec.Replicas))
    rsKey, err := controller.KeyFunc(rs)
    if err != nil {
        utilruntime.HandleError(
        	fmt.Errorf("Couldn't get key for %v %#v: %v", rsc.Kind, rs, err),
        )
        return nil
    }
    if diff < 0 {
        diff *= -1
        if diff > rsc.burstReplicas {
            diff = rsc.burstReplicas
        }
        rsc.expectations.ExpectCreations(rsKey, diff)
        klog.V(2).Infof("Too few replicas for %v %s/%s, need %d, creating %d",
        	rsc.Kind, rs.Namespace, rs.Name, *(rs.Spec.Replicas), diff,
        )
        successfulCreations, err := slowStartBatch(
        	diff,
        	controller.SlowStartInitialBatchSize,
        	func() error {
        		ref := metav1.NewControllerRef(rs, rsc.GroupVersionKind)
                err := rsc.podControl.CreatePodsWithControllerRef(
            	    rs.Namespace, &rs.Spec.Template, rs, ref,
                )
                if err != nil && errors.IsTimeout(err) {
                	return nil
                }
                return err
            },
        )
        if skippedPods := diff - successfulCreations; skippedPods > 0 {
            klog.V(2).Infof("Slow-start failure. Skipping creation of %d pods," +
            	" decrementing expectations for %v %v/%v",
            	skippedPods, rsc.Kind, rs.Namespace, rs.Name,
            )
            for i := 0; i < skippedPods; i++ {
                rsc.expectations.CreationObserved(rsKey)
            }
        }
        return err
    } else if diff > 0 {
        if diff > rsc.burstReplicas {
            diff = rsc.burstReplicas
        }
        klog.V(2).Infof("Too many replicas for %v %s/%s, need %d, deleting %d",
        	rsc.Kind, rs.Namespace, rs.Name, *(rs.Spec.Replicas), diff,
        )

        podsToDelete := getPodsToDelete(filteredPods, diff)
        rsc.expectations.ExpectDeletions(rsKey, getPodKeys(podsToDelete))
        errCh := make(chan error, diff)
        var wg sync.WaitGroup
        wg.Add(diff)
        for _, pod := range podsToDelete {
            go func(targetPod *v1.Pod) {
                defer wg.Done()
                if err := rsc.podControl.DeletePod(
                	rs.Namespace,
                	targetPod.Name,
                	rs,
                ); err != nil {
                    podKey := controller.PodKey(targetPod)
                    klog.V(2).Infof("Failed to delete %v, decrementing " +
                    	"expectations for %v %s/%s",
                    	podKey, rsc.Kind, rs.Namespace, rs.Name,
                    )
                    rsc.expectations.DeletionObserved(rsKey, podKey)
                    errCh <- err
                }
            }(pod)
        }
        wg.Wait()

        select {
        case err := <-errCh:
            if err != nil {
                return err
            }
        default:
        }
    }
    return nil
}
```

您可以看到控制器计算行中规范和当前状态之间的差异`diff` `:= len(filteredPods) - int(*(rs.Spec.Replicas))`，然后根据具体情况实现两种情况：

- `diff` `<` `0`：复制品太少; 必须创建更多的pod。
- `diff` `>` `0`：复制品太多; 必须删除pod。

它还实施了一种策略来选择最不利于删除它们的pod `getPodsToDelete`。

但是，更改资源状态并不一定意味着资源本身必须是Kubernetes集群的一部分。换句话说，控制器可以改变位于Kubernetes之外的资源的状态，例如云存储服务。例如，[AWS Service Operator](http://bit.ly/2ItJcif)允许您管理AWS资源。除此之外，它还允许您管理S3存储桶 - 即，S3控制器正在监控存在于Kubernetes之外的资源（S3存储桶），状态更改反映了其生命周期中的具体阶段：创建了一个S3存储桶，在某些时候删除。

这应该说服您使用自定义控制器，您不仅可以管理核心资源（如pod）和自定义资源（如我们的`cnat`示例），还可以计算或存储Kubernetes之外的资源。这使控制器具有非常灵活和强大的集成机制，提供了跨平台和环境使用资源的统一方法。

## 乐观并发

在[“控制循环”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#controller-loop)，我们 在步骤3中讨论了控制器 - 在根据规则更新集群对象和/或外部世界之后将结果规范写入触发控制器在步骤1中运行的资源的状态。

这和其他任何写入（也在步骤2中）都可能出错。在分布式系统中，此控制器可能只是更新资源的众多控制器之一。由于写冲突，并发写入可能会失败。

为了更好地了解正在发生的事情，让我们退一步看看[图1-4](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#scheduling-archs)。[2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#idm46336867946120)

![调度分布式系统中的体系结构](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0104.png)

###### 图1-4。调度分布式系统中的体系结构

该 source定义了Omega的并行调度器架构，如下所示：

> 我们的解决方案是围绕共享状态构建的新并行调度程序体系结构，使用无锁的乐观并发控制来实现实现可扩展性和性能可伸缩性。这种架构正在谷歌的下一代集群管理系统Omega中使用。

而Kubernetes继承了[Borg](http://bit.ly/2XNSv5p)的许多特性和经验教训，这个特定的事务控制平面功能来自Omega：为了在没有锁的情况下执行并发操作，Kubernetes API服务器使用乐观并发。

简而言之，这意味着如果API服务器检测到并发写入尝试，它将拒绝后两次写入操作。然后由客户端（控制器，调度程序`kubectl`等）来处理冲突并可能重试写操作。

以下演示了Kubernetes中乐观并发的概念：

```
var err error
 forretries：=0 ;retries <10 ;retries ++ {
    foo，err =client.Get ("foo"，metav1.GetOptions {})
    iferr！=零{
        break
    }

    <更新的环球和 -  FOO>

    _，err =client.Update (foo )
    iferr！=没有错误&&.IsConflict 错误！零() {
        continue
    } else if={
        break
    }
}
```

代码显示了一个重试循环，它`foo`在每次迭代中获取最新的对象，然后尝试更新世界和`foo`状态以匹配`foo`规范。在`Update`通话之前完成的更改是乐观的。

`foo`来自`client.Get`调用的返回对象包含一个*资源版本*（嵌入式`ObjectMeta`结构的一部分- 请参阅[“ObjectMeta”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#ObjectMeta)以获取详细信息），它将告诉调用后`etcd`的写操作，同时`client.Update`集群中的另一个actor编写了该`foo`对象。如果是这种情况，我们的重试循环将获得*资源版本冲突错误*。这意味着乐观并发逻辑失败。换句话说，`client.Update`电话也是乐观的。

###### 注意

资源版本实际上是`etcd`键/值版本。每个对象的资源版本是Kubernetes中包含整数的字符串。这个整数直接来自`etcd`。`etcd`维护一个计数器，每次修改一个键（保存对象的序列化）的值时，该计数器都会增加。

在整个API机器代码中，资源版本（或多或少因此）像任意字符串一样处理，但在其上有一些排序。存储整数的事实只是当前`etcd`存储后端的实现细节。

让我们看一个具体的例子。想象一下，您的客户端不是群集中修改pod的唯一角色。那里是另一个演员，即`kubelet`不断修改某些字段，因为容器不断崩溃。现在你的控制器读取pod对象的最新状态，如下所示：

```
kind: Pod
metadata:
  name: foo
  resourceVersion: 57
spec:
  ...
status:
  ...
```

现在假设控制器需要几秒钟来更新世界。七秒钟后，它尝试更新它读取的pod - 例如，它设置了一个注释。同时，`kubelet`已注意到另一个容器重启并更新了pod的状态以反映出来; 也就是说，`resourceVersion`已经增加到58。

控制器在更新请求中发送的对象具有`resourceVersion: 57`。API服务器尝试`etcd`使用该值设置pod 的密钥。`etcd`注意资源版本不匹配，并报告与58冲突58.更新失败。

此示例的底线是，对于您的控制器，您负责实施重试策略并适应乐观操作失败。您永远不知道还有谁可能在操纵状态，无论是其他自定义控制器还是核心控制器（如部署控制器）。

其实质是：*冲突错误在控制器中完全正常。总是期待他们并优雅地处理它们*。

重要的是要指出乐观并发非常适合基于级别的逻辑，因为通过使用基于级别的逻辑，您可以重新运行控制循环（请参阅[“Edge-Versus Level-Driven Triggers”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#edge-vs-level)）。该循环的另一次运行将自动撤消先前失败的乐观尝试的乐观变化，并且它将尝试将世界更新为最新状态。

让我们继续讨论自定义控制器的特定情况（以及自定义资源）：运营商。

## 运营商

运营商作为Kubernetes中的一个概念，CoreOS于2016年推出。在他的开创性博客文章[“介绍运营商：将操作知识融入软件”中](http://bit.ly/2ZC4Rui)，CoreOS首席技术官Brandon Philips将运营商定义如下：

> 一个现场可靠性工程师（SRE）是一个通过编写软件来操作应用程序的人。他们是工程师，开发人员，知道如何专门为特定应用领域开发软件。由此产生的软件具有编程到其中的应用程序的操作领域知识。
>
> […]
>
> 我们将这类新的软件称为操作员。Operator是一个特定于应用程序的控制器，它扩展了Kubernetes API，以代表Kubernetes用户创建，配置和管理复杂有状态应用程序的实例。它建立在基本的Kubernetes资源和控制器概念的基础上，但包括领域或特定于应用程序的知识，以自动执行常见任务。

在本书的上下文中，我们将使用飞利浦描述的运算符，更正式地说，要求满足以下三个条件（参见[图1-5](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#operator-conceptual)）：

- 您希望自动化一些特定于领域的操作知识。
- 这种操作知识的最佳实践是已知的并且可以明确 - 例如，在Cassandra操作员的情况下，何时以及如何重新平衡节点，或者在服务网格的操作员的情况下，如何创建一条路线。
- 在运营商的背景下运送的工件是：
  - 一个一组*自定义资源定义*（CRD）捕获CRD之后的特定于域的模式和自定义资源，在实例级别上，CRD表示感兴趣的域。
  - 自定义控制器，监控自定义资源，可能还有核心资源。例如，自定义控制器可能会启动一个pod。

![运营商的概念](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0105.png)

###### 图1-5。运营商的概念

运营商从2016年的概念性工作和原型设计到Red Hat（在2018年收购CoreOS并继续构建该想法）的[OperatorHub.io](https://operatorhub.io/)的推出已经[走了很长一段路](http://bit.ly/2x5TSNw)。在[图1中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#operatorhub)可以看到[图1-6](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#operatorhub)的截图。 2019年中期的中心，有大约17名运营商，随时可以使用。

![OperatorHub.io屏幕截图](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0106.png)

###### 图1-6。OperatorHub.io截图

# 摘要

在第一章中，我们定义了本书的范围以及我们对您的期望。我们通过编程Kubernetes解释了我们的意思，并在本书的上下文中定义了Kubernetes原生应用程序。作为后续示例的准备，我们还提供了对控制器和操作员的高级介绍。

所以，既然你已经知道了本书的内容以及如何从中获益，那么让我们深入探讨。在下一章中，我们将详细介绍Kubernetes API，API服务器的内部工作方式，以及如何使用命令行工具与API进行交互`curl`。

[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#idm46336877072840-marker)有关此主题的更多信息，请参阅Megan O'Keefe的[ “适用于MacOS的Kubernetes开发人员工作流程”](http://bit.ly/2WXfzu1)，*Medium*，2019年1月24日; 和Alex Ellis的博客文章[ “Be KinD to yourself”](http://bit.ly/2XkK9C1)，2018年12月14日。

[2](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#idm46336867946120-marker)来源：[ “Omega：适用于大型计算集群的灵活，可扩展的调度程序”](http://bit.ly/2PjYZ59)，作者：Malte Schwarzkopf等人，Google AI，2013。