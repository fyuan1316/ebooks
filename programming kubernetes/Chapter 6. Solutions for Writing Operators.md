# 第6章编写运算符的解决方案

所以到目前为止，我们已经在[“控制器和操作员”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#ch_controllers-operators)的概念层面上看过自定义控制器和运算符，在[第5章中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#ch_autocodegen)，我们将讨论如何使用Kubernetes代码生成器 - 一种处理该主题的低级方法。在本章中，我们将介绍三个解决方案 详细编写自定义控制器和操作员，并讨论更多替代方案。

使用本章中讨论的解决方案之一应该可以帮助您避免编写大量重复代码，并使您能够专注于业务逻辑，而不是样板代码。它应该让您更快地开始并提高您的工作效率。

###### 注意

一般而言，运营商以及我们在本章中具体讨论的工具在2019年中期仍在迅速发展。虽然我们尽力而为，但您在此处看到的某些命令和/或输出可能会发生变化。考虑到这一点，并确保始终使用相应工具的最新版本，密切关注相应的问题跟踪器，邮件列表和Slack通道。

虽然有在线资源可以[比较](http://bit.ly/2ZC5fZT)我们在此讨论的解决方案，但我们不会向您推荐具体的解决方案。但是，我们鼓励您自己评估和比较它们，并选择最适合您的组织和环境的那个。

# 制备

我们将使用`cnat`（`at`我们在[“动机示例”中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch01.html#mot-example)介绍的cloud-native ）作为本章中不同解决方案的运行示例。如果你想跟随，请注意我们假设你：

1. 安装了Go版本1.12或更高版本并正确设置。
2. 有权访问Kubernetes集群中的1.12或以上，或者通过在当地如版本`kind`或`k3d`，或者远程通过您最喜欢的云服务提供商和`kubectl`配置进行访问。
3. `git clone`我们的[GitHub存储库](http://bit.ly/2N3R6U4)。此处提供了完整，有效的源代码和以下部分中显示的必要命令。请注意，我们在这里展示的是如何从头开始工作。如果您想查看结果而不是自己执行这些步骤，也欢迎您克隆存储库并仅运行命令来安装CRD，安装CR并启动自定义控制器。

将这些内务管理项目排除在外，让我们跳到编写操作员：我们将`sample-controller`在本章中介绍，Kubebuilder和Operator SDK。

准备？让我们来吧！

# 以下样本控制器

让我们从`cnat`基于[*k8s.io/sample-controller*](http://bit.ly/2UppsTN)的实现开始，它[直接](http://bit.ly/2Yas9HK)使用`client-go`库。在使用[*k8s.io/code-generator*](http://bit.ly/2Kw8I8U)生成一个类型的客户端，告密者，一线明星，和深拷贝功能。每当自定义控制器中的API类型发生变化时 - 例如，在自定义资源中添加新字段 - 您必须使用*update-codegen.sh*脚本（另请参阅GitHub中的[源代码](http://bit.ly/2Fq3Td1)）来重新生成上述源文件。`sample-controller`

###### 警告

您可能已经注意到*k8s.io*被用作整本书中的基本URL。我们在[第3章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#ch_client-go)介绍了它的用法; 作为提醒，它实际上是*kubernetes.io*的别名，在Go包管理的上下文中，它解析为*github.com/kubernetes*。请注意，*k8s.io*没有自动重定向。所以，例如，*k8s.io / sample-control*真的意味着你应该看看[*github.com/kubernetes/sample-controller*](http://bit.ly/2UppsTN)，等等。

好的，让我们按照以下方式[`cnat`](http://bit.ly/2RpHhON)使用我们的运算符。（请参阅[我们的仓库中](http://bit.ly/2N3R6U4)的[相应目录](http://bit.ly/2N3R6U4)。）`client-go``sample-controller`

## 引导

至开始，做一个**go get k8s.io/sample-controller**将源和依赖项放到你的系统上，它应该在*$ GOPATH / src / k8s.io / sample- \ controller中*。

如果从头开始，将*sample-controller*目录的内容复制到您选择的目录中（例如，我们在repo中使用*cnat-client-go*），您可以运行以下命令序列来构建和运行基本控制器（使用默认实现，而不是`cnat`业务逻辑）：

```
# build custom controller binary:
$ 去构建-o cnat-controller。

# launch custom controller locally:
$ ./cnat-controller -kubeconfig =$HOME/.kube/config
```

此命令将启动自定义控制器并等待您注册CRD并创建自定义资源。我们现在就做，看看会发生什么。在第二个终端会话，输入：

```
$ kubectl apply -f artifacts / examples / crd.yaml
```

使 确保CRD正确注册并可以这样使用：

```
$ kubectl得到crds
姓名创建于
foos.samplecontroller.k8s.io 2019-05-29T12：16：57Z
```

请注意，您可能会在此处看到其他CRD，具体取决于您使用的Kubernetes发行版; 但是，*至少*应该列出*foos.samplecontroller.k8s.io*。

接下来，我们创建示例自定义资源*foo.samplecontroller.k8s.io/example-foo*并检查控制器是否完成其工作：

```
$ kubectl apply -f artifacts / examples / example-foo.yaml
创建了foo.samplecontroller.k8s.io/example-foo

$ kubectl得到po，rs，deploy，foo
NAME READY STATUS RESTARTS AGE
pod / example-foo-5b8c9679d8-xjhdf 1/1运行    0          67s

NAME希望当前的准备好年龄
replicaset.extensions / example-foo-5b8c9679d8    1         1         1     67s

名称准备好最新可用年龄
deployment.extensions / example-foo 1/1      1            1           67s

姓名年龄
foo.samplecontroller.k8s.io/example-foo 67s
```

是的，它按预期工作！我们现在可以继续实现实际的`cnat`特定业务逻辑。

## 商业逻辑

至开始实现业务逻辑，我们首先将现有目录*pkg / apis / samplecontroller*重命名为*pkg / apis / cnat*，然后创建我们自己的CRD和自定义资源，如下所示：

```
$ cat artifacts / examples / cnat-crd.yaml
apiVersion：apiextensions.k8s.io/v1beta1
kind：CustomResourceDefinition
元数据：
  name: ats.cnat.programming-kubernetes.info
规格：
  group：cnat.programming-kubernetes.info
  版本：v1alpha1
  名称：
    亲切的：在
    复数：ats
  范围：Namespaced

$ cat artifacts / examples / cnat-example.yaml
apiVersion：cnat.programming-kubernetes.info/v1alpha1
亲切的：在
元数据：
  标签：
    controller-tools.k8s.io: "1.0"
  name：example-at
规格：
  时间表"2019-04-12T10:12:00Z"
  command::"echo YAY"
```

请注意，每当API类型发生更改时（例如，当您向`At`CRD 添加新字段时），您必须执行*update-codegen.sh*脚本，如下所示：

```
$ ./hack/update-codegen.sh
```

这将自动生成以下内容：

- *包装/的API / CNAT / v1alpha1 / zz_generated.deepcopy.go*
- *PKG /生成/ **

在业务逻辑方面，我们在运营商中实现了两个部分：

- 在[*types.go中，*](http://bit.ly/31QosJw)我们修改`AtSpec`结构以包含相应的字段，例如`schedule`和`command`。请注意，`update-codegen.sh`每当您在此处更改某些内容时都必须运行，以便重新生成相关文件。
- 在[*controller.go中，*](http://bit.ly/31MM4OS)我们更改`NewController()`和`syncHandler()`函数以及添加辅助函数，包括创建窗格和检查调度时间。

在*types.go中*，请注意代表`At`资源三个阶段的三个常量：直到预定的时间`PENDING`，然后`RUNNING`到完成，最后在`DONE`状态：

```
// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

const (
    PhasePending = "PENDING"
    PhaseRunning = "RUNNING"
    PhaseDone    = "DONE"
)

// AtSpec defines the desired state of At
type AtSpec struct {
    // Schedule is the desired time the command is supposed to be executed.
    // Note: the format used here is UTC time https://www.utctime.net
    Schedule string `json:"schedule,omitempty"`
    // Command is the desired command (executed in a Bash shell) to be
    // executed.
    Command string `json:"command,omitempty"`
}

// AtStatus defines the observed state of At
type AtStatus struct {
    // Phase represents the state of the schedule: until the command is
    // executed it is PENDING, afterwards it is DONE.
    Phase string `json:"phase,omitempty"`
}
```

请注意构建标记的显式用法`+k8s:deepcopy-gen:interfaces`（请参阅[第5章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#ch_autocodegen)），以便自动生成相应的源。

我们现在可以实现自定义控制器的业务逻辑。也就是说，我们-从阶段实现三者之间的状态过渡`PhasePending`到`PhaseRunning`以`PhaseDone`-in [controller.go](http://bit.ly/31MM4OS)。

在[“工作队列”中，](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch03.html#workqueue)我们介绍并解释了`client-go`提供的工作队列。现在，我们可以把这些知识来工作：在`processNextWorkItem()`中*controller.go* -to更准确地说，在[线路176至186](http://bit.ly/2WYDbyi) -你可以找到以下（生成）的代码：

```
if when, err := c.syncHandler(key); err != nil {
    c.workqueue.AddRateLimited(key)
    return fmt.Errorf("error syncing '%s': %s, requeuing", key, err.Error())
} else if when != time.Duration(0) {
    c.workqueue.AddAfter(key, when)
} else {
    // Finally, if no error occurs we Forget this item so it does not
    // get queued again until another change happens.
    c.workqueue.Forget(obj)
}
```

这个片段展示了如何`syncHandler()`调用我们的（尚未编写的）自定义函数（稍后解释）并涵盖以下三种情况：

1. 第一个`if`分支通过`AddRateLimited()`函数调用重新排列项目，处理瞬态错误。
2. 第二个分支，`else if`通过`AddAfter()`函数调用重新排列项目以避免热循环。
3. 最后一种情况`else`是，该项已成功处理并通过`Forget()`函数调用被丢弃。

现在我们已经对通用处理有了充分的了解，让我们继续讨论特定于业务逻辑的功能。关键是上述`syncHandler()`功能，我们正在实现自定义控制器的业务逻辑。它有以下签名：

```
// syncHandler compares the actual state with the desired state and attempts
// to converge the two. It then updates the Status block of the At resource
// with the current status of the resource. It returns how long to wait
// until the schedule is due.
func (c *Controller) syncHandler(key string) (time.Duration, error) {
    ...
}
```

此`syncHandler()`函数实现以下状态转换：[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch06.html#idm46336858995208)

```
...
// If no phase set, default to pending (the initial phase):
if instance.Status.Phase == "" {
    instance.Status.Phase = cnatv1alpha1.PhasePending
}

// Now let's make the main case distinction: implementing
// the state diagram PENDING -> RUNNING -> DONE
switch instance.Status.Phase {
case cnatv1alpha1.PhasePending:
    klog.Infof("instance %s: phase=PENDING", key)
    // As long as we haven't executed the command yet, we need
    // to check if it's time already to act:
    klog.Infof("instance %s: checking schedule %q", key, instance.Spec.Schedule)
    // Check if it's already time to execute the command with a
    // tolerance of 2 seconds:
    d, err := timeUntilSchedule(instance.Spec.Schedule)
    if err != nil {
        utilruntime.HandleError(fmt.Errorf("schedule parsing failed: %v", err))
        // Error reading the schedule - requeue the request:
        return time.Duration(0), err
    }
    klog.Infof("instance %s: schedule parsing done: diff=%v", key, d)
    if d > 0 {
        // Not yet time to execute the command, wait until the
        // scheduled time
        return d, nil
    }

    klog.Infof(
       "instance %s: it's time! Ready to execute: %s", key,
       instance.Spec.Command,
    )
    instance.Status.Phase = cnatv1alpha1.PhaseRunning
case cnatv1alpha1.PhaseRunning:
    klog.Infof("instance %s: Phase: RUNNING", key)

    pod := newPodForCR(instance)

    // Set At instance as the owner and controller
    owner := metav1.NewControllerRef(
        instance, cnatv1alpha1.SchemeGroupVersion.
        WithKind("At"),
    )
    pod.ObjectMeta.OwnerReferences = append(pod.ObjectMeta.OwnerReferences, *owner)

    // Try to see if the pod already exists and if not
    // (which we expect) then create a one-shot pod as per spec:
    found, err := c.kubeClientset.CoreV1().Pods(pod.Namespace).
        Get(pod.Name, metav1.GetOptions{})
    if err != nil && errors.IsNotFound(err) {
        found, err = c.kubeClientset.CoreV1().Pods(pod.Namespace).Create(pod)
        if err != nil {
            return time.Duration(0), err
        }
        klog.Infof("instance %s: pod launched: name=%s", key, pod.Name)
    } else if err != nil {
        // requeue with error
        return time.Duration(0), err
    } else if found.Status.Phase == corev1.PodFailed ||
        found.Status.Phase == corev1.PodSucceeded {
        klog.Infof(
            "instance %s: container terminated: reason=%q message=%q",
            key, found.Status.Reason, found.Status.Message,
        )
        instance.Status.Phase = cnatv1alpha1.PhaseDone
    } else {
        // Don't requeue because it will happen automatically
        // when the pod status changes.
        return time.Duration(0), nil
    }
case cnatv1alpha1.PhaseDone:
    klog.Infof("instance %s: phase: DONE", key)
    return time.Duration(0), nil
default:
    klog.Infof("instance %s: NOP")
    return time.Duration(0), nil
}

// Update the At instance, setting the status to the respective phase:
_, err = c.cnatClientset.CnatV1alpha1().Ats(instance.Namespace).
    UpdateStatus(instance)
if err != nil {
    return time.Duration(0), err
}

// Don't requeue. We should be reconcile because either the pod or
// the CR changes.
return time.Duration(0), nil
```

此外，为了设置线人和控制器，我们在以下方面实施以下内容`NewController()`：

```
// NewController returns a new cnat controller
func NewController(
    kubeClientset kubernetes.Interface,
    cnatClientset clientset.Interface,
    atInformer informers.AtInformer,
    podInformer corev1informer.PodInformer) *Controller {

    // Create event broadcaster
    // Add cnat-controller types to the default Kubernetes Scheme so Events
    // can be logged for cnat-controller types.
    utilruntime.Must(cnatscheme.AddToScheme(scheme.Scheme))
    klog.V(4).Info("Creating event broadcaster")
    eventBroadcaster := record.NewBroadcaster()
    eventBroadcaster.StartLogging(klog.Infof)
    eventBroadcaster.StartRecordingToSink(&typedcorev1.EventSinkImpl{
        Interface: kubeClientset.CoreV1().Events(""),
    })
    source := corev1.EventSource{Component: controllerAgentName}
    recorder := eventBroadcaster.NewRecorder(scheme.Scheme, source)

    rateLimiter := workqueue.DefaultControllerRateLimiter()
    controller := &Controller{
        kubeClientset: kubeClientset,
        cnatClientset: cnatClientset,
        atLister:      atInformer.Lister(),
        atsSynced:     atInformer.Informer().HasSynced,
        podLister:     podInformer.Lister(),
        podsSynced:    podInformer.Informer().HasSynced,
        workqueue:     workqueue.NewNamedRateLimitingQueue(rateLimiter, "Ats"),
        recorder:      recorder,
    }

    klog.Info("Setting up event handlers")
    // Set up an event handler for when At resources change
    atInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: controller.enqueueAt,
        UpdateFunc: func(old, new interface{}) {
            controller.enqueueAt(new)
        },
    })
    // Set up an event handler for when Pod resources change
    podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: controller.enqueuePod,
        UpdateFunc: func(old, new interface{}) {
            controller.enqueuePod(new)
        },
    })
    return controller
}
```

为了使它工作，我们还需要两个辅助函数：一个计算直到计划的时间，如下所示：

```
func timeUntilSchedule(schedule string) (time.Duration, error) {
    now := time.Now().UTC()
    layout := "2006-01-02T15:04:05Z"
    s, err := time.Parse(layout, schedule)
    if err != nil {
        return time.Duration(0), err
    }
    return s.Sub(now), nil
}
```

另一个使用`busybox`容器图像创建一个带有要执行的命令的pod ：

```
func newPodForCR(cr *cnatv1alpha1.At) *corev1.Pod {
    labels := map[string]string{
        "app": cr.Name,
    }
    return &corev1.Pod{
        ObjectMeta: metav1.ObjectMeta{
            Name:      cr.Name + "-pod",
            Namespace: cr.Namespace,
            Labels:    labels,
        },
        Spec: corev1.PodSpec{
            Containers: []corev1.Container{
                {
                    Name:    "busybox",
                    Image:   "busybox",
                    Command: strings.Split(cr.Spec.Command, " "),
                },
            },
            RestartPolicy: corev1.RestartPolicyOnFailure,
        },
    }
}
```

我们将`syncHandler()`在本章后面的函数中重用这两个辅助函数和业务逻辑的基本流程，因此请确保您熟悉它们的详细信息。

请注意，从`At`资源的角度来看，pod是辅助资源，控制器必须确保清理这些pod或以其他方式冒孤立的pod。

现在，它`sample-controller`是学习如何制作香肠的好工具，但通常您希望专注于创建业务逻辑而不是处理样板代码。为此，您可以选择两个相关项目：Kubebuilder和Operator SDK。让我们看看每个以及如何`cnat`实现它们。

# Kubebuilder

[Kubebuilder](http://bit.ly/2I8w9mz)，拥有由Kubernetes特殊兴趣小组（SIG）API Machinery维护，是一个工具和一套库，使您能够以简单有效的方式构建操作员。Kubebuilder深度潜水的最佳资源是在线[Kubebuilder书籍](https://book.kubebuilder.io/)，它将向您介绍其组件和用法。但是，我们将重点关注[`cnat`](http://bit.ly/2RpHhON)使用Kubebuilder 实现我们的运算符（请参阅[我们的Git存储库中的相应目录](http://bit.ly/2Iv6pAS)）。

首先，让我们确保安装所有依赖项 - 即[dep](http://bit.ly/2x9Yrqq)，[kustomize](http://bit.ly/2Y3JeCV)（参见[“Kustomize”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch07.html#kustomize)）和[Kubebuilder本身](http://bit.ly/32pQmfu)：

```
$ dep版本
DEP：
 版本：v0.5.1
 建造日期：2019-03-11
 去hash    ：faa6189
 go version：go1.12
 go编译器：gc
 平台：darwin / amd64
 特征 ： ImportDuringSolve=false

$ kustomize版本
版本：{KustomizeVersion：v2.0.3 GitCommit：a6f65144121d1955266b0cd836ce954c04122dc8
          BuildDate：2019-03-18T22：15：21 + 00:00 GoOs：darwin GoArch：amd64}

$ Kubebuilder版本
版本：version.Version {
  KubeBuilder 版本："1.0.8"，
  KubernetesVendor : "1.13.1",
  GitCommit："1adf50ed107f5042d7472ba5ab50d5e1d357169d";
  BuildDate："2019-01-25T23:14:29Z"，GoOs："unknown"，GoArch："unknown"
}
```

我们将引导您完成`cnat`从头开始编写操作符的步骤。首先，创建一个您选择的目录（我们在我们*的仓库*中使用*cnat-kubebuilder*），您将用作所有其他命令的基础。

###### 警告

在撰写本文时，Kubebuilder正在转向新版本（v2）。由于它还不稳定，我们会显示（稳定）[版本v1](https://book-v1.book.kubebuilder.io/)的命令和设置。

## 引导

至引导`cnat`操作符，我们使用这样的`init`命令（请注意，这可能需要几分钟，具体取决于您的环境）：

```
$ kubebuilder init \
              --domain programming-kubernetes.info \
              --license apache2 \
              --owner "Programming Kubernetes authors"
运行`dep确保`获取依赖项(推荐的) [y / n ]？
和
保证
跑步......
使
go generate ./pkg / .... / cmd / ...
go fmt ./pkg / .... / cmd / ...
去兽医./pkg / .... / cmd / ...
去运行vendor / sigs.k8s.io / controller-tools / cmd / controller-gen / main.go all
CRD舱单下产生'config/crds'
RBAC舱单下产生'config/rbac'
去test./pkg / ... ./cmd / ... -coverprofile cover.out
？github.com/mhausenblas/cnat-kubebuilder/pkg/apis         [没有test 文件]
？github.com/mhausenblas/cnat-kubebuilder/pkg/controller   [没有test 文件]
？github.com/mhausenblas/cnat-kubebuilder/pkg/webhook      [没有test 文件]
？github.com/mhausenblas/cnat-kubebuilder/cmd/manager      [没有test 文件]
构建-o bin / manager github.com/mhausenblas/cnat-kubebuilder/cmd/manager
```

完成此命令后，Kubebuilder已经为操作员搭建了支架，有效地生成了一堆文件，从自定义控制器到样本CRD。您的现在，基本目录应该类似于以下内容（为清晰起见，不包括巨大的*供应商*目录）：

```
$ 树 - 我的供应商
.
├──Dockerfile
├──Gopkg.lock
├──Gopkg.toml
├──Makefile
├──项目
├──宾
│└──经理
├──cmd
│└──经理
│└──main.go
├──配置
│├──cds
│├──默认
├──│├──kustomization.yaml
││├──manage_auth_proxy_patch.yaml
││├──manage_image_patch.yaml
││└──manage_prometheus_metrics_patch.yaml
│├──经理
└──│└──manage.yaml
│└──rbac
│├──auth_proxy_role.yaml
│├──auth_proxy_role_binding.yaml
│├──auth_proxy_service.yaml
│├──rbac_role.yaml
│└──rbac_role_binding.yaml
├──cover.out
├──黑客
│└──boarplate.go.txt
└──pkg
    ├──apis
    │└──apis.go
    ├──控制器
    │└──controller.go
    何webhook
        └──webhook.go

13目录，22文件
```

接下来，我们创建一个API，即一个自定义控制器 - 使用该`create api`命令（这应该比上一个命令更快，但仍然需要一点时间）：

```
$ kubebuilder创建api \
              --group cnat \
              --version v1alpha1\
              - 来吧
在pkg / apis [y / n 下创建资源]？
和
在pkg / controller [y / n 下创建控制器]？
和
写脚手架for你要编辑......
包装/的API / CNAT / v1alpha1 / at_types.go
包装/的API / CNAT / v1alpha1 / at_types_test.go
PKG /控制器/ AT / at_controller.go
PKG /控制器/ AT / at_controller_test.go
跑步......
go generate ./pkg / .... / cmd / ...
go fmt ./pkg / .... / cmd / ...
去兽医./pkg / .... / cmd / ...
去运行vendor / sigs.k8s.io / controller-tools / cmd / controller-gen / main.go all
CRD舱单下产生'config/crds'
RBAC舱单下产生'config/rbac'
去test./pkg / ... ./cmd / ... -coverprofile cover.out
？github.com/mhausenblas/cnat-kubebuilder/pkg/apis         [没有test 文件]
？github.com/mhausenblas/cnat-kubebuilder/pkg/apis/cnat    [没有test 文件]
ok github.com/mhausenblas/cnat-kubebuilder/pkg/apis/cnat/v1alpha1 9.011s
？github.com/mhausenblas/cnat-kubebuilder/pkg/controller   [没有test 文件]
ok github.com/mhausenblas/cnat-kubebuilder/pkg/controller/at 8.740s
？github.com/mhausenblas/cnat-kubebuilder/pkg/webhook      [没有test 文件]
？github.com/mhausenblas/cnat-kubebuilder/cmd/manager      [没有test 文件]
构建-o bin / manager github.com/mhausenblas/cnat-kubebuilder/cmd/manager
```

让我们看看发生了什么变化，重点关注已经收到更新和补充的两个目录：

```
$ 树配置/ pkg /
配置/
├──cds
Nat└──cnat_v1alpha1_at.yaml
├──默认
├──├──kustomization.yaml
│├──manage_auth_proxy_patch.yaml
│├──manage_image_patch.yaml
│└──manage_prometheus_metrics_patch.yaml
├──经理
└──└──manage.yaml
├──rbac
│├──auth_proxy_role.yaml
│├──auth_proxy_role_binding.yaml
│├──auth_proxy_service.yaml
│├──rbac_role.yaml
│└──rbac_role_binding.yaml
└──样品
    └──cnat_v1alpha1_at.yaml
包装/
├──apis
├──addtoscheme_cnat_v1alpha1.go
│├──apis.go
.└──猫
│├──group.go
│└──v1alpha1
│├──at_types.go
│├──at_types_test.go
│├──doc.go
│├──register.go
│├──v1alpha1_suite_test.go
│└──zz_generated.deepcopy.go
├──控制器
│├──add_at.go
│├──在
││├──at_controller.go
││├──at_controller_suite_test.go
││└──at_controller_test.go
│└──controller.go
何webhook
    └──webhook.go

11目录，27文件
```

请注意在*config / crds /中*添加了*cnat_v1alpha1_at.yaml*，它是CRD，以及*config / samples /中的**cnat_v1alpha1_at.yaml*（是，同名），表示CRD的自定义资源示例实例。此外，在*pkg /中*我们看到了许多新文件，最重要的是*apis / cnat / v1alpha1 / at_types.go*和*controller / at / at_controller.go*，我们将在下面修改这两个文件。

接下来，我们`cnat`在Kubernetes中创建一个专用的命名空间并将其用作默认值，将上下文设置如下（作为一种好的做法，总是使用专用的命名空间，而不是`default`一个）：

```
$ kubectl创建ns cnat && \
  kubectl config set-context $(kubectl config current-context )--namespace =cnat
```

我们 安装CRD：

```
$ make install
去运行vendor / sigs.k8s.io / controller-tools / cmd / controller-gen / main.go all
根据'config/crds'
RBAC生成的CRD清单生成'config/rbac'
kubectl apply -f config / crds
customresourcedefinition.apiextensions.k8s.io/ats.cnat.programming-kubernetes.info created
```

和 现在我们可以在本地启动运营商：

```
$ 跑步
go generate ./pkg / .... / cmd / ...
go fmt ./pkg / .... / cmd / ...
去兽医./pkg / .... / cmd / ...
去运行./cmd/manager/main.go
{"level"："info"，"ts"：1559152740.0550249， ："logger"，"entrypoint"：
   "msg"：，"setting up client for manager"}
{"level" ：1559152740.057556， ：，：
   ：， ：1559152740.1396701， ：，：
   ：， ：1559152740.1397， ：，：
   ：， ：1559152740.139773， ：，：
   ：， ：1559152740.139831， ：，：
   ，： ，
   ：：，：1559152740.139929， ：，：
   ，：，：
  "info""ts""logger""entrypoint""msg""setting up manager"}
{"level""info""ts""logger""entrypoint""msg""Registering Components."}
{"level""info""ts""logger""entrypoint""msg""setting up scheme"}
{"level""info""ts""logger""entrypoint""msg""Setting up controller"}
{"level""info""ts""logger""kubebuilder.controller""msg""Starting EventSource""controller""at-controller""source""kind source: /, Kind="}
{"level""info""ts""logger""kubebuilder.controller""msg""Starting EventSource""controller""at-controller""source""kind source: /, Kind="}
{"level":"info","ts":1559152740.139971,"logger":"entrypoint",
  "msg":"setting up webhooks"}
{"level":"info","ts":1559152740.13998,"logger":"entrypoint",
  "msg":"Starting the Cmd."}
{"level":"info","ts":1559152740.244628,"logger":"kubebuilder.controller",
  "msg":"Starting Controller","controller":"at-controller"}
{"level":"info","ts":1559152740.344791,"logger":"kubebuilder.controller",
  "msg":"Starting workers","controller":"at-controller","worker count":1}
```

离开 终端会话正在运行，并在新会话中安装CRD，验证它，并创建示例自定义资源，如下所示：

```
$ kubectl apply -f config / crds / cnat_v1alpha1_at.yaml
customresourcedefinition.apiextensions.k8s.io/ats.cnat.programming-kubernetes.info
配置

$ kubectl得到crds
姓名创建于
ats.cnat.programming-kubernetes.info 2019-05-29T17：54：51Z

$ kubectl apply -f config / samples / cnat_v1alpha1_at.yaml
at.cnat.programming-kubernetes.info/at-sample created
```

如果现在查看`make run`运行的会话的输出，您应该注意以下输出：

```
...
 {"level"："info"，"ts"：1559153311.659829， ："logger"，"controller"：
   "msg"，"Creating Deployment"："namespace"，"cnat"："name"：，"at-sample-deployment"}
{"level" ：1559153311.678407， ：，：
   ，：，：：， ：1559153311.6839428， ：，：
   ，：，：：， ：1559153311.693443， ：，：
   ，：， ：：，：1559153311.7023401， ：，：
   ，：，：：，"info""ts""logger""controller""msg""Updating Deployment""namespace""cnat""name""at-sample-deployment"}
{"level""info""ts""logger""controller""msg""Updating Deployment""namespace""cnat""name""at-sample-deployment"}
{"level""info""ts""logger""controller""msg""Updating Deployment""namespace""cnat""name""at-sample-deployment"}
{"level""info""ts""logger""controller""msg""Updating Deployment""namespace""cnat""name""at-sample-deployment"}
{"level""info""ts":1559153332.986961,"logger":"controller",#
  "msg":"Updating Deployment","namespace":"cnat","name":"at-sample-deployment"}
```

这告诉我们整体设置成功！现在我们已经完成了脚手架并成功启动了`cnat`运营商，我们可以继续实际的核心任务：`cnat`使用Kubebuilder 实现业务逻辑。

## 商业逻辑

对于首先，我们将[*config / crds / cnat_v1alpha1_at.yaml*](http://bit.ly/2N1jQNb)和[*config / samples / cnat_v1alpha1_at.yaml更改*](http://bit.ly/2Xs1F7c)为我们自己的`cnat`CRD定义和自定义资源值，重新使用与[“Follow sample-controller”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch06.html#cnat-client-go)相同的结构。

在业务逻辑方面，我们在运营商中实现了两个部分：

- 在[*pkg / apis / cnat / v1alpha1 / at_types.go中，*](http://bit.ly/31KNLfO)我们修改`AtSpec`结构以包含相应的字段，例如`schedule`和`command`。请注意，`make`每当您在此处更改某些内容时都必须运行，以便重新生成相关文件。Kubebuilder使用Kubernetes生成器（在[第5章中](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch05.html#ch_autocodegen)描述）并发布它自己的一组生成器（例如，生成CRD清单）。
- 在[*pkg / controller / at / at_controller.go中，*](http://bit.ly/2Iwormg)我们修改了`Reconcile(request reconcile.Request)`在定义的时间创建pod 的方法`Spec.Schedule`。

在*at_types.go*：

```
const (
    PhasePending = "PENDING"
    PhaseRunning = "RUNNING"
    PhaseDone    = "DONE"
)

// AtSpec defines the desired state of At
type AtSpec struct {
    // Schedule is the desired time the command is supposed to be executed.
    // Note: the format used here is UTC time https://www.utctime.net
    Schedule string `json:"schedule,omitempty"`
    // Command is the desired command (executed in a Bash shell) to be executed.
    Command string `json:"command,omitempty"`
}

// AtStatus defines the observed state of At
type AtStatus struct {
    // Phase represents the state of the schedule: until the command is executed
    // it is PENDING, afterwards it is DONE.
    Phase string `json:"phase,omitempty"`
}
```

在*at_controller.go*我们落实三个阶段之间的状态转变，`PENDING`到`RUNNING`到`DONE`：

```
func (r *ReconcileAt) Reconcile(req reconcile.Request) (reconcile.Result, error) {
    reqLogger := log.WithValues("namespace", req.Namespace, "at", req.Name)
    reqLogger.Info("=== Reconciling At")
    // Fetch the At instance
    instance := &cnatv1alpha1.At{}
    err := r.Get(context.TODO(), req.NamespacedName, instance)
    if err != nil {
        if errors.IsNotFound(err) {
            // Request object not found, could have been deleted after
            // reconcile request—return and don't requeue:
            return reconcile.Result{}, nil
        }
            // Error reading the object—requeue the request:
        return reconcile.Result{}, err
    }

    // If no phase set, default to pending (the initial phase):
    if instance.Status.Phase == "" {
        instance.Status.Phase = cnatv1alpha1.PhasePending
    }

    // Now let's make the main case distinction: implementing
    // the state diagram PENDING -> RUNNING -> DONE
    switch instance.Status.Phase {
    case cnatv1alpha1.PhasePending:
        reqLogger.Info("Phase: PENDING")
        // As long as we haven't executed the command yet, we need to check if
        // it's already time to act:
        reqLogger.Info("Checking schedule", "Target", instance.Spec.Schedule)
        // Check if it's already time to execute the command with a tolerance
        // of 2 seconds:
        d, err := timeUntilSchedule(instance.Spec.Schedule)
        if err != nil {
            reqLogger.Error(err, "Schedule parsing failure")
            // Error reading the schedule. Wait until it is fixed.
            return reconcile.Result{}, err
        }
        reqLogger.Info("Schedule parsing done", "Result", "diff",
            fmt.Sprintf("%v", d))
        if d > 0 {
            // Not yet time to execute the command, wait until the scheduled time
            return reconcile.Result{RequeueAfter: d}, nil
        }
        reqLogger.Info("It's time!", "Ready to execute", instance.Spec.Command)
        instance.Status.Phase = cnatv1alpha1.PhaseRunning
    case cnatv1alpha1.PhaseRunning:
        reqLogger.Info("Phase: RUNNING")
        pod := newPodForCR(instance)
        // Set At instance as the owner and controller
        err := controllerutil.SetControllerReference(instance, pod, r.scheme)
        if err != nil {
            // requeue with error
            return reconcile.Result{}, err
        }
        found := &corev1.Pod{}
        nsName := types.NamespacedName{Name: pod.Name, Namespace: pod.Namespace}
        err = r.Get(context.TODO(), nsName, found)
        // Try to see if the pod already exists and if not
        // (which we expect) then create a one-shot pod as per spec:
        if err != nil && errors.IsNotFound(err) {
            err = r.Create(context.TODO(), pod)
            if err != nil {
            // requeue with error
                return reconcile.Result{}, err
            }
            reqLogger.Info("Pod launched", "name", pod.Name)
        } else if err != nil {
            // requeue with error
            return reconcile.Result{}, err
        } else if found.Status.Phase == corev1.PodFailed ||
                  found.Status.Phase == corev1.PodSucceeded {
            reqLogger.Info("Container terminated", "reason",
                found.Status.Reason, "message", found.Status.Message)
            instance.Status.Phase = cnatv1alpha1.PhaseDone
        } else {
            // Don't requeue because it will happen automatically when the
            // pod status changes.
            return reconcile.Result{}, nil
        }
    case cnatv1alpha1.PhaseDone:
        reqLogger.Info("Phase: DONE")
        return reconcile.Result{}, nil
    default:
        reqLogger.Info("NOP")
        return reconcile.Result{}, nil
    }

    // Update the At instance, setting the status to the respective phase:
    err = r.Status().Update(context.TODO(), instance)
    if err != nil {
        return reconcile.Result{}, err
    }

    // Don't requeue. We should be reconcile because either the pod
    // or the CR changes.
    return reconcile.Result{}, nil
}
```

请注意，最后的`Update`调用操作在*/ status*子资源上（参见[“Status subresource”](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#status-subresource)）而不是整个CR。因此，在这里我们遵循规范状态分割的最佳实践。

现在，一旦`example-at`创建了CR ，我们就会看到本地执行的运算符的以下输出：

```
$ 跑步
...
{"level"："info"，"ts"：1555063897.488535， ："logger"，"controller"：
   "msg"，"=== Reconciling At"："namespace"，"cnat"："at"：，"example-at"}
{"level" ：1555063897.488621， ：，：
   ，：，：：， ：1555063897.4886441， ：，：
   ，：，：，：
   ：， ：1555063897.488703， ：，：
   ，：，： ，
   ：：，：1555063907.489264， ：，：
   ，：，"info""ts""logger""controller""msg""Phase: PENDING""namespace""cnat""at""example-at"}
{"level""info""ts""logger""controller""msg""Checking schedule""namespace""cnat""at""example-at""Target""2019-04-12T10:12:00Z"}
{"level""info""ts""logger""controller""msg""Schedule parsing done""namespace""cnat""at""example-at""Result""2019-04-12 10:12:00 +0000 UTC with a diff of 22.511336s"}
{"level""info""ts""logger""controller""msg""=== Reconciling At""namespace""cnat""at"："example-at"}
{"level"："info"，"ts"：1555063907.489402， ："logger"，"controller"：
   "msg"，"Phase: PENDING"："namespace"，"cnat"："at"：，"example-at"}
{"level" ：1555063907.489428， ：，：
   ，：，：，：
   ：， ：1555063907.489486， ：，：
   ，：，：，：
   ：， ：1555063917.490178， ：，：
   ， ：，：：，：1555063917.4902349， ：，：
   ，："info""ts""logger""controller""msg""Checking schedule""namespace""cnat""at""example-at""Target""2019-04-12T10:12:00Z"}
{"level""info""ts""logger""controller""msg""Schedule parsing done""namespace""cnat""at""example-at""Result""2019-04-12 10:12:00 +0000 UTC with a diff of 12.510551s"}
{"level""info""ts""logger""controller""msg""=== Reconciling At""namespace""cnat""at""example-at"}
{"level""info""ts""logger""controller""msg""Phase: PENDING""namespace""cnat"，"at"："example-at"}
{"level"："info"，"ts"：1555063917.490247， ："logger"，"controller"：
   "msg"，"Checking schedule"："namespace"，"cnat"："at"，"example-at"：
   "Target"：，"2019-04-12T10:12:00Z"}
{"level" ：1555063917.490278， ：，：
   ，：，：，：
   ：， ：1555063927.492718， ：，：
   ，：，：：， ：1555063927.49283， ：，：
   ，：，：：，：1555063927.492857， ：，：
   ，"info""ts""logger""controller""msg""Schedule parsing done""namespace""cnat""at""example-at""Result""2019-04-12 10:12:00 +0000 UTC with a diff of 2.509743s"}
{"level""info""ts""logger""controller""msg""=== Reconciling At""namespace""cnat""at""example-at"}
{"level""info""ts""logger""controller""msg""Phase: PENDING""namespace""cnat""at""example-at"}
{"level""info""ts""logger""controller""msg""Checking schedule""namespace"："cnat"，"at"："example-at"，
   "Target"："2019-04-12T10:12:00Z"}
{"level"："info"，"ts"：1555063927.492915， ："logger"，"controller"：
   "msg"，"Schedule parsing done"："namespace"，"cnat"："at"，"example-at"：
   "Result"：，"2019-04-12 10:12:00 +0000 UTC with a diff of -7.492877s"}
{"level" ：1555063927.4929411， ：，：
   ，：，：，
   ：：， ：1555063927.626236， ：，：
   ，：，：：， ：1555063927.626303，：，
   ：，：，：：，：1555063928.07445， ："info""ts""logger""controller""msg""It's time!""namespace""cnat""at""example-at""Ready to execute""echo YAY"}
{"level""info""ts""logger""controller""msg""=== Reconciling At""namespace""cnat""at""example-at"}
{"level""info""ts""logger""controller""msg""Phase: RUNNING""namespace""cnat""at""example-at"}
{"level""info""ts""logger""controller",
  "msg":"Pod launched","namespace":"cnat","at":"example-at",
  "name":"example-at-pod"}
{"level":"info","ts":1555063928.199562,"logger":"controller",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063928.199645,"logger":"controller",
  "msg":"Phase: DONE","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063937.631733,"logger":"controller",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555063937.631783,"logger":"controller",
  "msg":"Phase: DONE","namespace":"cnat","at":"example-at"}
...
```

至 验证我们的自定义控制器是否已完成其工作，执行：

```
$ kubectl到达，豆荚
姓名年龄
at.cnat.programming-kubernetes.info/example-at 11m

NAME READY STATUS RESTARTS AGE
pod / example-at-pod 0/1完成      0          38s
```

大！该`example-at-pod`有 已创建，现在是时候看到操作的结果：

```
$ kubectl记录example-at-pod
SUMMER
```

完成自定义控制器的开发后，使用此处所示的本地模式，您可能希望从中构建容器图像。随后可以使用该自定义控制器容器映像，例如，在Kubernetes部署中。您可以使用以下命令生成容器图像并将其推送到repo *quay.io/pk/cnat*：

```
$ export IMG=quay.io/pk/cnat:v1

$ 使docker-build

$ 使docker-push
```

有了这个，我们转向运营商SDK，它共享一些Kubebuilder的代码库和API。

# 运营商SDK

至为了更容易构建Kubernetes应用程序，CoreOS / Red Hat将运营商框架整合在一起。其中一部分是[Operator SDK](http://bit.ly/2KtpK7D)，它使开发人员无需深入了解Kubernetes API即可构建运算符。

Operator SDK提供了构建，测试和打包运算符的工具。虽然SDK中有更多可用的功能，特别是在测试时，我们专注于[`cnat`](http://bit.ly/2RpHhON)使用SDK 实现我们的运算符（请参阅[我们的Git存储库中的相应目录](http://bit.ly/2FpCtE9)）。

首先要做的事情是：确保[安装Operator SDK](http://bit.ly/2ZBQlCT)并检查所有依赖项是否可用：

```
$ dep版本
DEP：
 版本：v0.5.1
 建造日期：2019-03-11
 去hash    ：faa6189
 go version：go1.12
 go编译器：gc
 平台：darwin / amd64
 特征 ： ImportDuringSolve=false

 $ operator-sdk --version
operator-sdk version v0.6.0
```

## 引导

现在是时候`cnat`按如下方式引导操作员了：

```
$ operator-sdk new cnat-operator && cd cnat-operator
```

接下来，和Kubebuilder非常相似，我们添加一个API - 或简单地说：初始化自定义控制器，如下所示：

```
$ operator-sdk add api \
               --api-version =cnat.programming-kubernetes.info/v1alpha1 \
               --kind =At

$ operator-sdk add controller \
               --api-version =cnat.programming-kubernetes.info/v1alpha1 \
               --kind =At
```

这些命令产生必要的样板代码以及一些辅助功能，如深复印功能`DeepCopy()`，`DeepCopyInto()`和`DeepCopyObject()`。

现在 我们可以将自动生成的CRD应用于Kubernetes集群：

```
$ kubectl apply -f deploy / crds / cnat_v1alpha1_at_crd.yaml

$ kubectl得到crds
姓名创建于
ats.cnat.programming-kubernetes.info 2019-04-01T14：03：33Z
```

让我们`cnat`在本地启动我们的自定义控制器。有了它，它可以开始处理请求：

```
$ OPERATOR_NAME=cnatop operator-sdk up local--namespace "cnat"
INFO [0000 ]在本地运行运算符。
INFO [0000 ]使用命名空间cnat。
{"level"："info"，"ts"：1555041531.871706， ："logger"，"cmd"：
   "msg"：，"Go Version: go1.12.1"}
{"level" ：1555041531.871785， ：，：
   ：， ：1555041531.8718028， ：，：
   ：， ：1555041531.8739321， ：，：
   ：， ：1555041531.8743382， ：，：
   ：， ：1555041536.1611362， ：，：
   ：， ：1555041536.1622112，：，
   ：，：，
  "info""ts""logger""cmd""msg""Go OS/Arch: darwin/amd64"}
{"level""info""ts""logger""cmd""msg""Version of operator-sdk: v0.6.0"}
{"level""info""ts""logger""leader""msg""Trying to become the leader."}
{"level""info""ts""logger""leader""msg""Skipping leader election; not running in a cluster."}
{"level""info""ts""logger""cmd""msg""Registering Components."}
{"level""info""ts""logger""kubebuilder.controller""msg""Starting EventSource""controller""at-controller""source":"kind source: /, Kind="}
{"level":"info","ts":1555041536.162519,"logger":"kubebuilder.controller",
  "msg":"Starting EventSource","controller":"at-controller",
  "source":"kind source: /, Kind="}
{"level":"info","ts":1555041539.978822,"logger":"metrics",
  "msg":"Skipping metrics Service creation; not running in a cluster."}
{"level":"info","ts":1555041539.978875,"logger":"cmd",
  "msg":"Starting the Cmd."}
{"level":"info","ts":1555041540.179469,"logger":"kubebuilder.controller",
  "msg":"Starting Controller","controller":"at-controller"}
{"level":"info","ts":1555041540.280784,"logger":"kubebuilder.controller",
  "msg":"Starting workers","controller":"at-controller","worker count":1}
```

我们的自定义控制器将保持此状态，直到我们创建CR，*ats.cnat.programming-kubernetes.info*。所以我们这样做：

```
$ cat deploy / crds / cnat_v1alpha1_at_cr.yaml
apiVersion：cnat.programming-kubernetes.info/v1alpha1
亲切的：在
元数据：
  name：example-at
规格：
  时间表"2019-04-11T14:56:30Z"
  command::"echo YAY"

$ kubectl apply -f deploy / crds / cnat_v1alpha1_at_cr.yaml

$ kubectl得到
姓名年龄
at.cnat.programming-kubernetes.info/example-at 54s
```

## 商业逻辑

在 在业务逻辑方面，我们在运营商中实现了两个部分：

- 在[*pkg / apis / cnat / v1alpha1 / at_types.go中，*](http://bit.ly/31Ip2sF)我们修改`AtSpec`struct以包含相应的字段，例如`schedule`和`command`，并用于`operator-sdk generate k8s`重新生成代码，以及使用`operator-sdk generate openapi`OpenAPI位的命令。
- 在[*pkg / controller / at / at_controller.go中，*](http://bit.ly/2Fpo5Mi)我们修改了`Reconcile(request reconcile.Request)`在定义的时间创建pod 的方法`Spec.Schedule`。

更详细地应用于自举代码的更改如下（关注相关位）。在*at_types.go*：

```
// AtSpec defines the desired state of At
// +k8s:openapi-gen=true
type AtSpec struct {
    // Schedule is the desired time the command is supposed to be executed.
    // Note: the format used here is UTC time https://www.utctime.net
    Schedule string `json:"schedule,omitempty"`
    // Command is the desired command (executed in a Bash shell) to be executed.
    Command string `json:"command,omitempty"`
}

// AtStatus defines the observed state of At
// +k8s:openapi-gen=true
type AtStatus struct {
    // Phase represents the state of the schedule: until the command is executed
    // it is PENDING, afterwards it is DONE.
    Phase string `json:"phase,omitempty"`
}
```

在*at_controller.go*我们实施了三个阶段的状态图，`PENDING`来`RUNNING`来`DONE`。

###### 注意

这[`controller-runtime`](http://bit.ly/2ZFtDKd)是另一个SIG API Machinery拥有的项目，旨在为Go包形式的构建控制器提供一套通用的低级功能。有关详细信息，请参阅[第4章](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch04.html#ch_crds)。

由于Kubebuilder和Operator SDK共享控制器运行时，该`Reconcile()`函数实际上是相同的：

```
func (r *ReconcileAt) Reconcile(request reconcile.Request) (reconcile.Result, error) {
    the-same-as-for-kubebuilder
}
```

`example-at`创建CR后，我们会看到本地执行的运算符的以下输出：

```
$ OPERATOR_NAME=cnatop operator-sdk up local--namespace "cnat"
INFO [0000 ]在本地运行运算符。
INFO [0000 ]使用命名空间cnat。
...
{"level"："info"，"ts"：1555044934.023597， ："logger"，"controller_at"：
   "msg"，"=== Reconciling At"："namespace"，"cnat"："at"：，"example-at"}
{"level" ：1555044934.023713， ：，：
   ，：，：：， ：1555044934.0237482， ：，：
   ，：，：，
   ：：， ：1555044934.02382， ：，：
   ，：，： ，
   ：：，：1555044934.148148， ：，：
   ，：，"info""ts""logger""controller_at""msg""Phase: PENDING""namespace""cnat""at""example-at"}
{"level""info""ts""logger""controller_at""msg""Checking schedule""namespace""cnat""at""example-at""Target""2019-04-12T04:56:00Z"}
{"level""info""ts""logger""controller_at""msg""Schedule parsing done""namespace""cnat""at""example-at""Result""2019-04-12 04:56:00 +0000 UTC with a diff of 25.976236s"}
{"level""info""ts""logger""controller_at""msg""=== Reconciling At""namespace""cnat""at"："example-at"}
{"level"："info"，"ts"：1555044934.148224， ："logger"，"controller_at"：
   "msg"，"Phase: PENDING"："namespace"，"cnat"："at"：，"example-at"}
{"level" ：1555044934.148243， ：，：
   ，：，：，：
   ：， ：1555044934.1482902， ：，：
   ，：，：，：
   ：， ：1555044944.1504588， ：，：
   ， ：，：：，：1555044944.150568， ：，：
   ，："info""ts""logger""controller_at""msg""Checking schedule""namespace""cnat""at""example-at""Target""2019-04-12T04:56:00Z"}
{"level""info""ts""logger""controller_at""msg""Schedule parsing done""namespace""cnat""at""example-at""Result""2019-04-12 04:56:00 +0000 UTC with a diff of 25.85174s"}
{"level""info""ts""logger""controller_at""msg""=== Reconciling At""namespace""cnat""at""example-at"}
{"level""info""ts""logger""controller_at""msg""Phase: PENDING""namespace""cnat"，"at"："example-at"}
{"level"："info"，"ts"：1555044944.150599， ："logger"，"controller_at"：
   "msg"，"Checking schedule"："namespace"，"cnat"："at"，"example-at"：
   "Target"：，"2019-04-12T04:56:00Z"}
{"level" ：1555044944.150663， ：，：
   ，：，：，：
   ：， ：1555044954.385175， ：，：
   ，：，：：， ：1555044954.3852649， ：，：
   ，：，：：，：1555044954.385288， ：，：
   ，"info""ts""logger""controller_at""msg""Schedule parsing done""namespace""cnat""at""example-at""Result""2019-04-12 04:56:00 +0000 UTC with a diff of 15.84938s"}
{"level""info""ts""logger""controller_at""msg""=== Reconciling At""namespace""cnat""at""example-at"}
{"level""info""ts""logger""controller_at""msg""Phase: PENDING""namespace""cnat""at""example-at"}
{"level""info""ts""logger""controller_at""msg""Checking schedule""namespace"："cnat"，"at"："example-at"，
   "Target"："2019-04-12T04:56:00Z"}
{"level"："info"，"ts"：1555044954.38534， ："logger"，"controller_at"：
   "msg"，"Schedule parsing done"："namespace"，"cnat"："at"，"example-at"：
   "Result"：，"2019-04-12 04:56:00 +0000 UTC with a diff of 5.614691s"}
{"level" ：1555044964.518383， ：，：
   ，：，：：， ：1555044964.5184839， ：，：
   ，：，：：， ：1555044964.518566， ：，
   ：，：，：，
   ：：，：1555044964.5186381， ："info""ts""logger""controller_at""msg""=== Reconciling At""namespace""cnat""at""example-at"}
{"level""info""ts""logger""controller_at""msg""Phase: PENDING""namespace""cnat""at""example-at"}
{"level""info""ts""logger""controller_at""msg""Checking schedule""namespace""cnat""at""example-at""Target""2019-04-12T04:56:00Z"}
{"level""info""ts""logger""controller_at"，
   "msg"："Schedule parsing done"，"namespace"："cnat"，"at"："example-at"，
   "Result"："2019-04-12 04:56:00 +0000 UTC with a diff of -4.518596s"}
{"level"："info"，"ts"：1555044964.5186849， ："logger"，"controller_at"：
   "msg"，"It's time!"："namespace"，"cnat"："at"，"example-at"：
   "Ready to execute"：，"echo YAY"}
{"level" ：1555044964.642559， ：，：
   ，：，：：， ：1555044964.642622， ：，：
   ，：，：：， ：1555044964.911037 ，：，
   ：，：，：：，：1555044964.9111192，"info""ts""logger""controller_at""msg""=== Reconciling At""namespace""cnat""at""example-at"}
{"level""info""ts""logger""controller_at""msg""Phase: RUNNING""namespace""cnat""at""example-at"}
{"level""info""ts""logger""controller_at""msg""=== Reconciling At""namespace""cnat""at""example-at"}
{"level""info""ts""logger":"controller_at",
  "msg":"Phase: RUNNING","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044966.038684,"logger":"controller_at",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044966.038771,"logger":"controller_at",
  "msg":"Phase: DONE","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044966.708663,"logger":"controller_at",
  "msg":"=== Reconciling At","namespace":"cnat","at":"example-at"}
{"level":"info","ts":1555044966.708749,"logger":"controller_at",
  "msg":"Phase: DONE","namespace":"cnat","at":"example-at"}
...
```

在这里你可以看到我们的运营商的三个阶段：`PENDING`直到时间戳`1555044964.518566`，然后`RUNNING`，然后`DONE`。

要验证自定义控制器的功能并检查操作结果，请输入：

```
$ kubectl到达，豆荚
姓名年龄
at.cnat.programming-kubernetes.info/example-at 23m

NAME READY STATUS RESTARTS AGE
pod / example-at-pod 0/1已完成      0          46秒

$ kubectl记录example-at-pod
SUMMER
```

完成自定义控制器的开发后，使用此处所示的本地模式，您可能希望从中构建容器图像。随后可以使用该自定义控制器容器映像，例如，在Kubernetes部署中。您可以使用以下命令生成容器图像：

```
$ operator-sdk build $REGISTRY/ PROJECT / IMAGE
```

这里 是了解更多有关Operator SDK及其示例的更多资源：

- Toader Sebastian在BanzaiCloud上发表的[“Kubernetes Operator SDK完整指南”](http://bit.ly/2RqkGSf)
- Rob Szumski的博客文章[“为Prometheus和Thanos建立一个Kubernetes运营商”](http://bit.ly/2KvgHmu)
- 来自Cloudark on ITNEXT的[“改善可用性的Kubernetes运营商开发指南”](http://bit.ly/31P7rPC)

为了总结本章，我们来看一些编写自定义控制器和运算符的替代方法。

# 其他方法

在 除了我们讨论过的方法之外，或者可能与之相结合，您可能需要查看以下项目，库和工具：

- [Metacontroller](https://metacontroller.app/)

  该Metacontroller的基本思想是为您提供状态和变化的声明性规范，与JSON接口，基于级别触发的协调循环。也就是说，您将收到描述观察状态的JSON并返回描述所需状态的JSON。这对于在Python或JavaScript等动态脚本语言中快速开发自动化特别有用。除了简单的控制器，Metacontroller还允许您将API组合成更高级别的抽象 - 例如，[BlueGreenDeployment](http://bit.ly/31KNTfi)。

- [EVERYWHERE](https://kudo.dev/)

  类似对于Metacontroller，KUDO提供了一种声明性方法来构建Kubernetes运算符，涵盖整个应用程序生命周期。简而言之，它是Mesosphere从Apache Mesos框架体验到Kubernetes的经验。KUDO高度自以为是，但也易于使用，几乎不需要编码; 实质上，您必须指定的是Kubernetes清单的集合，其中包含用于定义何时执行的内置逻辑。

- [Rook操作工具包](http://bit.ly/2J34faw)

  这个是一个实现运营商的通用库。它起源于Rook运营商，但已被分拆成一个独立的独立项目。

- [ericchiang / K8S](http://bit.ly/2ZHc5h0)

  这个是使用Kubernetes协议缓冲支持生成的Eric Chiang精简的Go客户端。它的行为类似于官方的Kubernetes `client-go`，但只导入两个外部依赖项。虽然它有一些限制 - 例如，在[集群访问配置方面](http://bit.ly/2ZBQIxh) - 它是一个简单易用的Go包。

- [`kutil`](http://bit.ly/2Fq3ojh)

  AppsCode通过提供Kubernetes `client-go`附加组件`kutil`。

- 基于CLI客户端的方法

  一个客户端方法，主要用于实验和测试，是以`kubectl`编程方式利用（例如，[kubecuddler](http://bit.ly/2L3CDoi)库）。

###### 注意

虽然我们专注于使用本书中的Go编程语言编写运算符，但您可以使用其他语言编写运算符。二值得注意的例子是Flant的[Shell-operator](http://bit.ly/2ZxkZ0m)，它使您能够在良好的旧shell脚本中编写运算符，以及Zalando的[Kopf（Kubernetes运算符框架）](http://bit.ly/2WRXU6Q)，Python框架和库。

正如本章开头所提到的，运营商领域正在迅速发展，越来越多的从业者以代码和最佳实践的形式分享他们的知识，因此请关注这里的新工具。请一定要看看网上资源和论坛，比如`#kubernetes-operators`，`#kubebuilder`和`#client-go-docs`渠道上Kubernetes松弛，学习新的方法和/或讨论问题，当你被卡住得到帮助。

# 吸收和未来方向

评委们仍然认为编写运算符的方法将是最受欢迎和广泛使用的。在Kubernetes项目的背景下，在CR和控制器方面，有几个SIG的活动。主要利益相关者是SIG [API Machinery](http://bit.ly/2RuTPEp)，它拥有CR和控制器，负责[Kubebuilder](http://bit.ly/2I8w9mz)项目。运营商SDK已经加大了与Kubebuilder API的协调力度，因此存在很多重叠。

# 摘要

在本章中，我们了解了不同的工具，使您可以更有效地编写自定义控制器和运算符。传统上，跟随它`sample-controller`是唯一的选择，但是使用Kubebuilder和Operator SDK，您现在有两个选项可以让您专注于自定义控制器的业务逻辑而不是处理样板。幸运的是，这两个工具共享了很多API和代码，因此从一个工具转移到另一个工具应该不会太困难。

现在，让我们看看如何提供我们的劳动成果 - 即如何打包和运送我们一直在编写的控制器。

[1](https://learning.oreilly.com/library/view/programming-kubernetes/9781492047094/ch06.html#idm46336858995208-marker)我们只在这里展示相关部分; 函数本身有很多其他的样板代码，我们并不关心它们。
