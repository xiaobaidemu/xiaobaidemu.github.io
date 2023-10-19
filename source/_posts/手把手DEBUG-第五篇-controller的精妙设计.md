---
title: 手把手DEBUG-第五篇-controller的精妙设计
date: 2023-08-19
tags:
    - kubernetes
categories:
    - 云原生
description: 本文围绕k8s中controller这个抽象的组件，来聊聊我是怎么来学习和理解这个非常关键的，并且由多个组件共同构成的控制k8s核心API逻辑的。
---
# 从informer/workqueue到controller

informer直译为告密者，所以正如字面意思理解，它会监听k8s中各种API对象的变化，然后将监听到的信息，告诉后面的处理逻辑执行相应的处理。

informer中一个很重要的抽象叫做Reflector，它是informer架构中的一个关键组件。它是负责从 API 服务器监听资源对象的变更事件（如添加、更新和删除），并将这些变化被同步到本地的存储（如 DeltaFIFO 队列和 Indexer，后面会提到）并且支持通过回调的方式在执行用户自定义的事件处理函数。

informer这个架构，可以在k8s.io/client-go/informers中看到他的源码，由于client-go中的examples中已经提供了很多示例，我这里提供一个方便debug的非常简单的代码（主要是为了方便通过断点调试进行走读）

```golang
package main

import (
    "flag"
    "fmt"
    "path/filepath"
    "time"

    appsv1 "k8s.io/api/apps/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/util/runtime"
    "k8s.io/client-go/informers"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/cache"
    "k8s.io/client-go/tools/clientcmd"
    "k8s.io/client-go/util/homedir"
)

func main() {
    var kubeconfig *string
    if home := homedir.HomeDir(); home != "" {
        kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file")
    } else {
        kubeconfig = flag.String("kubeconfig", "", "absolute path to the kubeconfig file")
    }
    flag.Parse()

    config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
    if err != nil {
        panic(err)
    }
    clientset, err := kubernetes.NewForConfig(config)
    if err != nil {
        panic(err)
    }
    // deploymentsClient := clientset.AppsV1().Deployments(apiv1.NamespaceDefault)
    // 创建一个 SharedInformerFactory，以便于创建 informer。
    watchNamespace := "default"
    // 一个自定义的本地索引，下面indexer章节会叙述
    indexFunc := func(obj interface{}) ([]string, error) {
        deployment := obj.(*appsv1.Deployment)
        app := deployment.Labels["app"]
        return []string{app}, nil
    }
    // 这里我们设置一个informerFactory，表示仅监听指定命名空间和指定label的对象，通过这个工厂，可以创建对具体某一个API对象的监控，通过factory来创建各类API对象的informer可以节约很多计算网络资源，有效减少本地和远程apiserver的压力
    // 在debug时候，可以设置informer的同步周期为非常大，避免频繁同步造成debug在频繁在多个goroutine中切换影响debug过程中代码走读

    factory := informers.NewSharedInformerFactoryWithOptions(clientset, time.Hour*10, informers.WithNamespace(watchNamespace),
        informers.WithTweakListOptions(func(options *metav1.ListOptions) {
            options.LabelSelector = "app=label-test"
        }))
    // 创建一个 Deployment informer，用于监视集群中 Deployment 资源的变化。
    deploymentInformer := factory.Apps().V1().Deployments().Informer()
    deploymentInformer.AddIndexers(cache.Indexers{"app": indexFunc})
    // 为 Deployment informer 添加事件处理器，处理资源添加、更新和删除事件。
    deploymentInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            deployment := obj.(*appsv1.Deployment)
            fmt.Printf("Deployment added: %s, %s, %+v\n", deployment.GetName(), deployment.GetResourceVersion(), deployment.Status)
        },
        UpdateFunc: func(oldObj, newObj interface{}) {
            indexer := deploymentInformer.GetIndexer()
            deployment1, _ := indexer.ByIndex("namespace", "default")
            obj1 := deployment1[0].(*appsv1.Deployment)
            fmt.Printf("before !!!!!!!!!!!!!!!!!!!!! %+v\n", obj1.DeletionTimestamp)
        },
        DeleteFunc: func(obj interface{}) {
            deployment := obj.(*appsv1.Deployment)
            fmt.Printf("Deployment deleted: %s, %s, %+v\n", deployment.GetName(), deployment.GetResourceVersion(), deployment.Status)
        },
    })
    // 启动 informer，开始监视集群中 Deployment 资源的变化。
    stopCh := make(chan struct{})
    defer close(stopCh)
    defer runtime.HandleCrash()

    factory.Start(stopCh) // 直接阻塞，不使用协程方便debug
    // 无限循环，直到程序被中断。
    <-stopCh
}
```

代码默认会读取本地的$HOME/.kube/config配置，用于在集群外连接apiserver。vscode launch配置如下

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "client-go informer-demo debug",
            "type": "go",
            "request": "launch",
            "mode": "debug",
            "program": "${workspaceFolder}/staging/src/k8s.io/client-go/examples/informer-demo",
            "args": [],
            "env": {},
            "showLog": true,
            "trace": "verbose"
        }
    ]
}
```

上述代码中，并没有显示的声明一个Reflector,这是因为它更多是informer的内部实现的一部分，在实际应用中，使用SharedInformer 或 InformerFactory，以便更方便地管理多个资源类型和事件处理器。这些组件在内部使用 Reflector 来同步数据，但提供了更高级的功能，如自动管理 Reflector 实例、共享缓存和多路复用 Watch 连接等。通过使用 SharedInformer 或 InformerFactory，以便轻松地处理多个 API 资源，而无需手动创建和管理 Reflector 实例。

这里我们通过创建一个label为app=label-test的deployment。然后再启动debug程序，来对代码走读。下面是一些关键的debug点，可以通过这些debug理解，informer在监听过程中内部的具体实现。

|  |  |  |  |
|:----|:----|:----|:----|
| 1 | staging/src/k8s.io/client-go/tools/cache/reflector.go | func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error  | 首先拉取所有Reflector中指定的需要监听的k8s资源对象，同时开启两个协程：1. 其中一个协程根据用户设定的周期时间，定期触发拉取全量的k8s监听资源 2. 另一个协程主要是创建一个stream监听流（ 如果是HTTP/2 在 则利用<font color=red>Server-Push、Multiplexing 上的高性能 Stream 特性</font>，在实现 RESTful Watch ）此流就是通过下面2中面熟的newStreamWatcher创建的 |
| 2 | staging/src/k8s.io/client-go/rest/request.go | func (r *Request) newStreamWatcher(resp*http.Response) (watch.Interface, error)  | 创建出来的流是一个high level的抽象，其底层实现了Decoder:从网络的io.Reader解析二进制字节流，并将其组装成json.同时通过channel方式将此对象由channel传递到informer DeltaFIFO中（DeltaFIFO下文来解释）<p> (关于具体如何接收二进制并且拼装数据，可以断点staging/src/k8s.io/apimachinery/pkg/util/framer/framer.go->Read函数) |
| 3 | staging/src/k8s.io/client-go/tools/cache/reflector.go | func (r *Reflector) list(stopCh <-chan struct{}) error | 从apiserver中获取所有informer中指定所有对象，由于上述代码中informer是针对development，所以会在启动informer前获取所有的development对象信息 |

<font color=#0099ff>当然Reflector 中的 Watch 能力的根源来自于 etcd 支持监听变化。</font>Kubernetes 使用 etcd 作为其主要的数据存储，etcd 提供了一种高效的机制来监听存储中数据的变化。etcd 中的 watch API 允许客户端订阅一个或多个键（key）的变化。当这些键的值发生变化（例如，添加、修改或删除）时，etcd 会将相应的事件发送给客户端。这使得客户端可以实时了解存储中数据的变化，而无需频繁地轮询 etcd。

至此启动informer后首轮拉取全量监听的资源，以及创建监听资源变化的watch stream流已经完成。之后我们可以直接执行，kubectl delete demployment <deployment_name>操作，可以进一步观察informer收到watch流中的数据，并更新indexer，以及触发用户自定义的操作。

删除操作触发后，deployment最先感知到的是Updated，这里会涉及到感知到deployment例如status等变化，然后全量更新cache中对应资源的信息，<font color=red>以及相应的索引信息</font>。

## 最简单的方法理解informer中的索引indexer

索引本质上很简单，可以理解为是以下3个map的组合。
|map对象名|key|value|备注|用golang结构理解|
|:----|:----|:----|:----|:----|
|indexers|自定义的字符串任意字符串，代表这个<font color=red>索引类名</font>|函数：type IndexFunc func(obj interface{}) ([]string, error) <p> 根据任意对象(可以是k8s中的对象)，生成多个索引值|一个indexers中可以包含多个kv，也就是说可以包含多种索引的生成方式。不同的kv代表不同的索引生成方式。默认的每个informer中都会包含一个key为namespace的索引类名，其value就是可以根据k8s对象返回命名空间值的函数IndexFunc。|map[string]indexFunc|
|indices|同indexers<p> 自定义的字符串任意字符串，代表这个<font color=red>索引类名 </font>|value也是一个map，此map的key就包含了相同索引类名中对应上述indexers中的indexFunc生成的所有索引值|indexers和indices是一一对应的，也就是存在于indexer的key中的索引类名，一定存在于indices的key中|map[string]Index <p>  type Index map[string]sets.String <p> type String map[string]struct{} <p> 其中Index中的key就是indexers中indexFunc生成的索引值，而后面的sets.String就是这个对应的索引值中包含的所有对象键(见下面的items)|
|items|k8s资源对象生成的代表某一个对象的<font color=red>对象键<font>|对象键对应interface{},例如deployment就代表的是k8s.io/api/apps/v1.Deployment|每个存在于上面Index的sets.String中的对象键，都可以在items中定位到。对象键名默认是<namespace>/<api对象名>组成|map[string]interface{}|

我们也可以自己定义indexer，下面的代码是一个自定义的Indexers，支持根据deployment对象中的label中key为"app"的value值作为索引键，并将这个构建索引的方式命名为"app"这个索引类名。

```golang
indexFunc := func(obj interface{}) ([]string, error) {
       deployment := obj.(*appsv1.Deployment)
    app := deployment.Labels["app"]
       return []string{app}, nil
}
factory := informers.NewSharedInformerFactoryWithOptions(clientset, time.Hour*10, informers.WithNamespace(watchNamespace),
       informers.WithTweakListOptions(func(options *metav1.ListOptions) {
        // 这里我们简化逻辑只监听符合app=label-test这个标签的api对象
              options.LabelSelector = "app=label-test"
       }))
// 创建一个 Deployment informer，用于监视集群中 Deployment 资源的变化。
deploymentInformer := factory.Apps().V1().Deployments().Informer()
deploymentInformer.AddIndexers(cache.Indexers{"app": indexFunc})
// 设置EventHandeler/启动informer开始监听......
indexer := deploymentInformer.GetIndexer()
// 从索引类名为app索引中，找到索引值为label-test的所有对象
deployments, _ := indexer.ByIndex("app", "label-test")
```

结合上面的代码，假如我们创建了一个名为demo-deploymen的Deployment对象，且label为"app=label-test"的对象，我们用一个表格来展示这个代码运行时，三个索引对象中的内容

|  |  |
|:---|:---|
|indexers|{"app":indexFunc; "namespace": MetaNamespaceIndexFunc} // 其中后面的namespace是默认包含的索引方式|
|indices|{<p>app": {"label-test": set(["default/demo-deployment",".<其他对象键>"])} <p> "namespace": {"default": set(["default/demo-deployment","<其他对象键>"])}<p>}|
|items |{"default/demo-deployment": <api对象>，"<其他对象键>": <api对象>}|

## 处理DeltaFIFO中的增删改消息

<font color=RoyalBlue>DeltaFIFO 是 Informer 架构的一个关键组件</font>，主要作用是确保事件处理器（event handlers）按照正确的顺序处理资源对象的添加、修改和删除事件。当 Reflector 从 API 服务器接收到资源对象的变化时，它会将这些对象添加到 DeltaFIFO 中。DeltaFIFO 会为每个对象维护一个队列，其中包含该对象的一系列变化（称为 Delta）。

DeltaFIFO 提供了一种名为 Pop 的方法，用于获取并处理下一个待处理的资源对象。Pop 方法会按照资源对象在队列中的顺序处理它们的变化，并将这些变化传递给事件处理器。这确保了事件处理器按照正确的顺序处理资源对象的添加、修改和删除事件。

Delta中定义了5种对象变更类型（Sync/Replaced/Added/Updated/Delete）但是本质上在处理过程中只有下表列出的三种处理过程。

另外需要注意的是<font color=RoyalBlue>Kubernetes API 服务器会将整个对象的新状态发送给 watch 客户端。因此，Informer 接收到的是整个对象的变化，而不是增量变化</font>。然而，在处理这些变化时，事件处理函数会比较新旧对象，以确定发生了哪些具体变化。例如OneUpdate

|  |  |
|:---|:---|
|OnUpdate(oldObj, newObj interface{})|当监听到变化的对象存在于内存缓存(Store)中，获取old状态，然后直接更新Store中改对象的信息。|
|OnAdd(obj interface{})|当监听到变化的对象不存在于内存缓存(Store)中，则在Store中新增对象。|
|Deleted(obj interface{})|从Store中删除对应的对象|

需要注意的是用户常常使用的是SharedFactory创建的informer,而不会直接对Delta进行操作，如果使用sharedInformer监听事件，那上面的三个函数<font color=red>并非在监听到事件后直接执行用户在informer中注册的增删改行为</font>，而是将这个通知动作通过channel 1 写入一个无界环形缓冲区，然后在无限读取这个缓冲区的通知信息，写入到channel2. 同时另一个协程持续性的监听并读取channel2的消息，每当读取到一个消息，然后执行用户注册的函数，可以发现<font color=red>执行用户注册的EventHandler的过程都是串行执行的</font>。那么当需要监听的api对象非常多的时候，势必会很多性能问题，甚至导致检测到的变化延迟过大。这个时候就需要下面要说的的<font color=red>workqueue</font>来缓解这种情况。

上面的设计和实现是一个非常巧妙的实现，通过两个协程+两个channel+一个无界环形缓冲区，完成了一个简单的消息的缓冲处理，这种设计可以用来很多的队列优化上.

```golang
// processorListener 通过两个协程+两个channel+一个无界环形缓冲区，完成了一个简单的消息的缓冲处理。
// k8s.io/client-go/tools/cache/shared_informer.go
type processorListener struct {
    nextCh chan interface{}
    addCh  chan interface{}
    handler ResourceEventHandler
        pendingNotifications buffer.RingGrowing
        .....
}
// 协程1 持续从addCh中获取消息，并写入到缓冲区中，然后再依次从缓冲区中提取一个消息写入nextCh
func (p *processorListener) pop() {...}
// 协程2 持续从nextCh获取消息，处理用户注册的eventhandler
func (p *processorListener) run() {
```

## 从workqueue来看controller的设计

informer本身不包含workqueue的设计，本质上在接收到ADD/Update/Delete动作后，执行AddEventHandler中注册的行为即可，但是注册进addHandler中的函数本身是串行执行的，所以如果不使用一些队列模型很难做到高并发的处理。workqueue的这也是可以在各类controller控制器中用途非常多的一个组件。

workqueue包括多种类型，从通用类型，到支持重试的队列类型，可以用在很多其他系统中，用来优化整个系统。

### 通用队列

k8s.io/client-go/util/workqueue/rate_limiting_queue/queue.go

一个通用的FIFO队列，但是区别于最简单的由纯数组构成的队列，这个队列添加两个set对象（<font color=RoyalBlue>正在处理的对象集合processing和所有需要被处理的对象集合dirty</font>）。
因为队列主要与informer和controller搭配使用，所以一个k8s对象会频繁的监听到变化，因为会常常有这种情况，controller正在处理某一个对象时，有可能监听到多个Update进入到队列。通过用两个set，可以有效的减少对同一个对象短时间频繁执行reconcile操作（下面的"数组"表示用于存放先入先出的数据的结构）

Add: 入队的时候： 1. 存在于dirty则不再写入数组 2: 不存在于dirty则要写入到dirty集合中，表示再次入队，而如果同时又存在在processing中，则不用在写入到数组中（原因可以看下面的Done的分析）

Get：从队列数组中弹出第一个元素，并从插入到processing中，表示开始处理，同时从dirty中删除。

Done：表示任务处理完成，需要从processing集合中删除。同时删除后判断是否该对象仍然出现在dirty集合中，如果存在说明在处理过程中又一次通过ADD加入，需要再次被放入到数组中。可以想想如果不用这两个额外的set，<font color=RoyalBlue>假如在某次处理过程中，监听到该对象变化了非常多次，那么该对象会入队多次，后面又会执行非常多次，而如果有了这两个set，即使监听到了多次变化，后续也只会再执行一次reconcile操作</font>

### 延时队列

k8s.io/client-go/util/workqueue/delaying_queue.go

延时队列是在通用队列的基础上增加一个能力，可以通过<font color=Red>AddAfter</font>方法，指定多长时间再入队列，这种方法可以指定某一个处理失败的项目在指定时间后再进行重试，有效避免短时间多次重试。

其本质是在通用队列的基础上，<font color=Red>增加了一个最小堆，堆顶是一个距离当前时间最近的需要需要入队的item</font>，通过在一个协程waitingLoop()持续循环，监测最小堆的堆顶元素的入队时间是否已经到达，如果已经达到入队时间，则从最小堆弹出堆顶元素，加入到通用队列中。

### 限速队列

k8s.io/client-go/util/workqueue/rate_limiting_queue.go

限速队列又是在延迟队列的基础上增加了一个限速器。限速器可以指导延迟队列，告诉延迟队列执行<font color=Red>AddAfter</font>操作的时候，具体应该等待多久(也就是设置的时间参数是多大)。

常见的限速器：

1. ItemExponentialFailureRateLimiter（指数退避限速：记录每一个入队元素的错误次数，进而根据失败次数，逐步延长重试的间隔）

2. BucketRateLimiter （基于golang原生的rate实现，相当于内嵌了一个恒定速率的限速器）

## 官方controller sample浅析

在k8s.io/sample-controller中有一个官方的controller说明，非常清晰的说明了controller的开发。controller本质上是informer和workqueue的组合，通过informer高效的监听所需要的API资源状态以及变化而不至于对apiserver造成过大负载，同时通过workqueue以及不同的队列模式，稳步高效处理api对象。下面这个经典的被很多博客所描述的图就是来自于<https://github.com/kubernetes/sample-controller/blob/master/docs/images/client-go-controller-interaction.jpeg>
![controller](手把手DEBUG-第五篇-controller的精妙设计/client-go-controller-interaction.jpg)

这个sample-controller主要是完成了对名为<font color=red>Foo的CRD资源</font>的监听和控制。下面阐述一下要完成一个这个controller的实现要具体完成什么事情。下面的阐述以[GitHub - kubernetes/sample-controller: Repository for sample controller. Complements sample-apiserver](https://github.com/kubernetes/sample-controller/tree/master)中代码作实例。

1. 创建一个名为Foo的CRD资源（CRD可以用于管理数据库的实例，或者由k8s原语组成的更高抽象的资源）

```bash
# 参考https://github.com/kubernetes/sample-controller/blob/master/artifacts/examples/crd-status-subresource.yaml
kubectl create -f artifacts/examples/crd-status-subresource.yaml
```

2. 根据定义的CRD，完成相关types.go和doc.go的编写，types.go就包含了如下。Foo结构体必须含有基础的TypeMeta和ObjectMeta对象，Spec和Status为用户定义的信息。其中// +tag_name=tag_value 主要是服务于后面自动代码生成

```golang
package v1alpha1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// Foo is a specification for a Foo resource
type Foo struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   FooSpec   `json:"spec"`
    Status FooStatus `json:"status"`
}

// FooSpec is the spec for a Foo resource
type FooSpec struct {
    DeploymentName string `json:"deploymentName"`
    Replicas       *int32 `json:"replicas"`
}

// FooStatus is the status for a Foo resource
type FooStatus struct {
    AvailableReplicas int32 `json:"availableReplicas"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// FooList is a list of Foo resources
type FooList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata"`

    Items []Foo `json:"items"`
}
```

同时必须要有一个doc.go（如果没有doc.go，后续的自动代码生成工具无法生成zz_generated.deepcopy.go相关文件），用于告知自动生成脚本生成的对象基本信息（例如groupName等）

```golang
// +k8s:deepcopy-gen=package,register
// +groupName=samplecontroller.k8s.io

// Package v1alpha1 is the v1alpha1 version of the API.
package v1alpha1 // import "k8s.io/sample-controller/pkg/apis/samplecontroller/v1alpha1"
```

文件的所在的层次目录如下

```text
- pkg
    - apis/samplecontroller // samplecontroller就是我们定义的这个CRD
       - v1alpha1 // groupname的version，根据预定一个对象group可以包含不同的版本
           - doc.go
           - types.go
           - zz_generated.deepcopy.go // 待生成的文件
       - register.go // 包含samplecontroller这个CRD在k8s中的groupname常量名称
    - generated // 这个目录下面的目录均是通过代码生成脚本生成的
        - clientset
        - informers
        - listers
- hack
    - update-codegen.sh 
```

然后我们可以执行hack/update-codegen.sh脚本生成 pkg/apis/samplecontroller/v1alpha1/zz_generated.deepcopy.go和pkg/generated代码。后者就是上文中informer方式监听CRD对象时用到的组件。

3. 撰写controller。官方的例子非常简单，其定义了一个Kind为Foo这个CRD对象（可以通过kubectl get Foo 查看所有创建的Foo实例），一个Foo对象就是一个deployment，当controller监听到一个Foo对象的创建/修改, 则对应的着创建/修改deployment对象。

```golang
type Controller struct {
    // kubeclientset is a standard kubernetes clientset
    kubeclientset kubernetes.Interface
    // sampleclientset is a clientset for our own API group
    sampleclientset clientset.Interface

    deploymentsLister appslisters.DeploymentLister
    deploymentsSynced cache.InformerSynced
    foosLister        listers.FooLister
    foosSynced        cache.InformerSynced

    // workqueue is a rate limited work queue. This is used to queue work to be
    // processed instead of performing it as soon as a change happens. This
    // means we can ensure we only process a fixed amount of resources at a
    // time, and makes it easy to ensure we are never processing the same item
    // simultaneously in two different workers.
    workqueue workqueue.RateLimitingInterface
    // recorder is an event recorder for recording Event resources to the
    // Kubernetes API.
    recorder record.EventRecorder
}

```

上面就是用于Foo的Controller包含的信息，因为同时要监听Foo实例以及要根据Foo实例的变化创建Deployment，也就是要同时操作两类API，一个是我们自己定义的API对象Foo，另一个就是要操作k8s原生的API对象Deployment，所以sampleclientset和foosLister用于操作我们自定义的API对象，而kubeclientset/deploymentsLister用于操作标准的k8s对象(deployment)。

另外workqueue存在的目的就是有效限制同时处理多个资源的速度，避免对apiserver或者系统其他组件造成负载。

根据上面的informer和workqueue解析，一个controller最基本的两个操作就是设置Informer的EventHandler以及设置工作协程持续读取队列内容处理。下面对controller中最重要的三个函数进行一些说明

### AddEventHandler

fooInformer.Informer().AddEventHandler  主要用于在监听到Foo的CR(Custom Resource)对象的增/改事件（为什么例子中没有删除，后文会提）将<font color=red>最新的Foo实例对象对应的key放入限速队列</font>）

### processNextWorkItem

被持续循环执行的函数，这个函数就做了三个事情

1. 从workqueue中取出一个Foo对象
2. 执行syncHandler，syncHandler是一个reconcile过程，对比当前状态和期望的状态
      如果执行失败，则需要重试，主动执行c.workqueue.AddRateLimited(key)重新写入延迟队列
3. 无论2是否成功，执行 c.workqueue.Done(obj)表示本次的处理完成

### syncHandler

在syncHandler函数中，主要有以下几种情况

1. 查询对应的Foo对象对应的deployment是否存在，如果不存在则根据Foo中Spec中指定的replicas创建对应的Deployment
2. 如果当前Foo所创建的Deployment中deployment.Spec.Replicas!=foo.Spec.Replicas，则表示用户可能修改了Foo的定义，需要同时修改deployment的Spec.Replicas
执行c.kubeclientset.AppsV1().Deployments(foo.Namespace).Update(context.TODO(), newDeployment(foo), metav1.UpdateOptions{})
3. 最后更新Foo.Status中的AvailableReplicas为当前Foo对应的Deployment中<font color=red>deployment.Status.AvailableReplicas</font>

如果任何一环出现错误，则直接进行return err，processNextWorkItem会重新将Foo对象入队，等待一段时间自动触发执行，这里最重要的就是要保证syncHandler是一个幂等的函数，无论执行多少次都不应该有副作用

这里不知道你有没有一个疑惑，<font color=RoyalBlue>为什么没有设置对Foo这个CRD的DeleteFunc</font>，在fooInformer只设置了AddFunc和UpdateFunc，并没有DeleteFunc。这个是因为由于Foo和Deployment使一一对应的，并且每一个创建的Deployment都设置了OwnerReferences为Foo。

```golang
&appsv1.Deployment{
    ObjectMeta: metav1.ObjectMeta{
        Name:      foo.Spec.DeploymentName,
        Namespace: foo.Namespace,
        OwnerReferences: []metav1.OwnerReference{
            *metav1.NewControllerRef(foo, samplev1alpha1.SchemeGroupVersion.WithKind("Foo")),
        },
    },
......
```

在执行kubectl delete Foo <foo_name>时，kube-controller会执行cascading delete级联删除（概念参考[Garbage Collection | Kubernetes](https://kubernetes.io/docs/concepts/architecture/garbage-collection/) ），也就是在删除Foo实例对象的时候，也会将对应的deployment删除（因为创建deployment时设置了ownerRefernces为foo）.默认执行kubectl delete Foo <foo_name> 是后台删除与其关联的deployment，所以执行完会立刻返回成功。如果使用 kubectl delete Foo example-foo --cascade=foreground 前台删除模式，则执行命令后，会等到所有deployment的资源删除完后，命令才返回。

## 附录

1. [深入源码分析 kubernetes client-go list-watch 和 informer 机制的实现原理](https://xiaorui.cc/archives/7361)
2. [深入源码分析 kubernetes client-go sharedIndexInformer 和 SharedInformerFactory 的实现原理](https://xiaorui.cc/archives/7359)
3. <https://github.com/kubernetes/sample-controller/blob/master/docs/controller-client-go.md>
4. [workqueue - K8S训练营](https://www.qikqiak.com/k8strain/k8s-code/client-go/workqueue/#_1)
5. [Indexer - K8S训练营](https://www.qikqiak.com/k8strain/k8s-code/client-go/indexer/)
6. [Garbage Collection | Kubernetes](https://kubernetes.io/docs/concepts/architecture/garbage-collection/))
7. [Kubernetes Deep Dive: Code Generation for CustomResources](https://cloud.redhat.com/blog/kubernetes-deep-dive-code-generation-customresources)
