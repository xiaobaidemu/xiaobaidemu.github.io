---
title: 手把手DEBUG(第四篇：被调度到node上的pod怎么启动的)
date: 2023-08-18
tags:
    - kubernetes
categories:
    - 云原生
sticky: 997
description: 本文围绕k8s中kubelet这个组件，展开说说我是怎么来学习和理解k8s中关于kubelet如何管理pod这部分内容的
---
更详细的debug操作可以参考我的[手把手一起DEBUG-Kubernetes-第一篇-建立k8s的DEBUG环境](手把手一起DEBUG-Kubernetes-第一篇-建立k8s的DEBUG环境)

## 1.编译kubelet二进制

```bash
make all WHAT=cmd/kubelet DBG=1
```

## 2.终止默认的kubelet进程

&emsp;&emsp;kubelet不同于kube-apiserver以及kube-scheduler，kubelet 进程是由操作系统的启动流程或者系统初始化过程中的 systemd 工具来启动的，并作为一个系统级别的服务。所以需要使用systemctl停止。

```bash
systemctl stop kubelet
```

&emsp;&emsp;注意需要将所有的nodes节点的kubelet停止，否则如果某一个节点的kubelet没有停止, 则创建的pod后，就会被立刻调度到这个没有停止的节点上，导致无法进入断点。停止节点上的kubelet后，可以查看nodes节点状态均处于NotReady状态。

```bash
 % kubectl get nodes
NAME                STATUS     ROLES           AGE    VERSION
vm-231-177-centos   NotReady   control-plane   160d   v1.25.3
vm-244-254-centos   NotReady   <none>          159d   v1.25.3
```

## 3.创建一个最基本的pod

&emsp;&emsp;在创建pods前，需要注意的是，由于目前k8s节点上的kubelet均因为我们要进行debug，所以都终止了。这个时候上述的节点可能会被k8s打上两个污点Taint 如下

```yaml
Taints:             node.kubernetes.io/unreachable:NoExecute
                    node.kubernetes.io/unreachable:NoSchedule
```

&emsp;&emsp;其中第二个污点的与pod是否可以被调度到该node相关，所以在创建pod前，我们可以在pod的spec中增加tolerations，来表示同意scheduler将该pod调度到此存在"node.kubernetes.io/unreachable:NoSchedule"这个污点的节点上。另外为了保证pod一定被调度到我们启动debug的节点上，可以在对应的节点上增加label，并通过设置nodeSelector保证pod被调度到待debug的节点，这里我们将debug所需的节点增加了一个label

```bash
kubectl label nodes <debug_node> node-restriction.kubernetes.io/resource=gpu
```

&emsp;&emsp;然后我们在pod对应的yaml中的spec中设定nodeSelector以及tolerations, 然后执行 kubectl apply -f 创建pod

```yaml
spec:
  ...
  nodeSelector:
    node-restriction.kubernetes.io/resource: gpu
  tolerations:
    - key: node.kubernetes.io/unreachable
      operator: Exists
      effect: NoSchedule
```

&emsp;&emsp;注意由于我们已经将kubelet对应的进程移除，所以创建的pod的状态始终是如下处于Pending状态，并且根据[手把手DEBUG-第三篇-pod怎么被调度到一个node上](./手把手DEBUG-第三篇-pod怎么被调度到一个node上)可以猜到，其Status.Conditions为已经处于PodScheduled完成状态(pod已经被成功分配到了我们debug进程等待启动的节点上)，如下

```yaml
Conditions:
  Type           Status
  PodScheduled   True 
...
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  9m22s  default-scheduler  Successfully assigned default/myapp-centos to vm-244-254-centos
```

&emsp;&emsp;而正常情况下，如果Pod被kubelet创建的话，其Status.Conditions应该为

```yaml
Conditions:
  Type            Status
  Initialized     True
  Ready           True
  ContainersReady True
  PodScheduled    True 
```

&emsp;&emsp;大概可以猜测，在PodScheduled执行完成后，其Conditions会依次经历PodScheduled/ContainerReady/Ready，同时创建Spec中要求的所有容器，等待容器启动，然后最终变成Initialized，下面我们可以通过debug逐渐佐证我们的猜想。

## 4.通过vscode开始对kubelet进行debug

&emsp;&emsp;kubelet要想处理被调度的pods，我们可以猜想一定是其接收到了某一个对pod的修改信息，然后通过不断地循环loop reconcile调谐维护容器生命周期，例如Pod状态同步（监听容器状态，并根据需要采取相应的措施例如启动停止重启来维护容器健康）来逐步逼近最终的pod状态。
|  |  |  |  |
|---|---|---|---|
| 1 | cmd/kubelet/app/server.go | func RunKubelet(kubeServer *options.KubeletServer, kubeDeps*kubelet.Dependencies, runOnce bool) | 如其名RunKubelet，开始启动kubelet |
| 2 | pkg/kubelet/kubelet.go | func NewMainKubelet(...) | 一个非常大的函数，主要是根据kubelet所需要的依赖实例化kubelet对象，这里有一个非常重要的对象PodConfig，此对象中的updates属性是一个channel，会监听apiserver中与当前Node绑定(spec.nodeName=当前nodehostname）的所有pods变化 |
|  |  |  |  |

当我们运行启动了kubelet后，我们可以将断点打在这个下面函数所在位置，这样我们按照章节3创建完pod后，由于通过ListWatch会监听与当前Node绑定的所有pods，则当有新的pod被调度到当前的Node,则就会收到所有

|  |  |  |  |
|---|---|---|---|
| 3 | pkg/kubelet/config/apiserver.go | func NewSourceApiserver(c clientset.Interface, nodeName types.NodeName, nodeHasSynced func() bool, updates chan<- interface{}) | 此函数就是2中提到的专门监听在当前node上有pod变化的协程，可以查看k8s.io/client-go/tools/cache/reflector.go中ListAndWatch具体梳理是如何感知到有pods变化的（这部分由于比较复杂不在这里参数） |
| 4 | pkg/kubelet/config/apiserver.go | func newSourceApiserverFromLW(lw cache.ListerWatcher, updates chan<- interface{}) | 这里再接收到变化是，会将信息通过channel写入updates对象中，通过channel传递pods的变化，我们可以将断点端在匿名函数此函数内部send函数中，监听写入channel的过程 |
|  |  |  |  |

```golang
send := func(objs []interface{}) {
    var pods []*v1.Pod
    for _, o := range objs {
    pods = append(pods, o.(*v1.Pod))
    }
    # 可以断点在这里，创建的信号从这里开始
    updates <- kubetypes.PodUpdate{Pods: pods, Op: kubetypes.SET, Source: kubetypes.ApiserverSource}
}
```

|  |  |  |  |
|---|---|---|---|
| 5 | pkg/kubelet/config/config.go | func (s *podStorage) merge(source string, change interface{}) (adds, updates, deletes, removes, reconciles*kubetypes.PodUpdate)  | 从上面第4步监听到的所有pods，会和原来内存中记录的pods做对比，来决定接下来是创建，移除还是做调谐动作 |
| 6 | pkg/kubelet/config/config.go | updatePodsFunc := func(newPods []*v1.Pod, oldPods, pods map[types.UID]*v1.Pod)  | 这是一个内嵌的匿名函数，此函数就是用于将内存中oldPods和上面ListAndWatch接收到的所有newPods对比。 <p> 1.  如果newPods中存在新的pod(这个pod不存在于oldPods中)则加入到addPods列表中，等待创建。|
| 7 | pkg/kubelet/config/config.go | func checkAndUpdatePod(existing, ref *v1.Pod) (needUpdate, needReconcile, needGracefulDelete bool)  | 2.  (紧跟上面的6)如果newPods中pod已经存在于oldPods中，则需要对比前后的pod状态，决定对这个pod的下一个动作是什么。本质上就是就是通过reflect.DeepEqual对比新旧pod的Spec/Label/Annotation/字段是否有改变。 <p> a. 如果存在变化则将pod加入到updatePods队列中. <p> b. 如果这些字段没有变化，但是pod的状态发生了变化，则表示kubelet需要对此pod进行reconcile调谐操作, 使老的pod状态趋近到新的期望的状态。同理此pod进入到reconcilePods队列.<p> c. 当然如果还新的pod中DeletionTimestamp非nil，表示此pod需要被删除，放入到deletePods队列。<p> 这些需要后续通过不同的处理逻辑pods，最终都通过podStorage中的channel传递到另一个协程进行处理。（podStorage可以认为是存储着当前这个时刻最真实状态的pod的信息，这些信息用来和最新的接收到的pods进行对比，从而进一步更新存储的pod信息）|
| 8 | pkg/kubelet/kubelet.go | func (kl *Kubelet) syncLoopIteration(configCh <-chan kubetypes.PodUpdate, handler SyncHandler...) | 此函数可以说是kubelet的核心，它从不同的channel获取对pod的操作动作然后执行对应逻辑。对于我们这个刚创建的pod，根据上面7中阐述，由于新增了一个pod，则会执行HandlePodAdditons.此函数会将pod加入到PodManager中进行后续管理，并且会执行dispatchWork异步执行创建逻辑，而真正的创建和后续维护工作，则由9中提到的podWorkers来完成 |
| 9 | pkg/kubelet/pod_workers.go | func (p *podWorkers) UpdatePod(options UpdatePodOptions)  | 对于本次debug流程，最重要的就是通过UpdatePod这个函数执行创建逻辑。对于当前我们debug的位置，我们创建的pod还没有创建，则UpdatePod会将我们的pod加入到podUpdates这个map，**每一个pod都会启动一个managePodLoop这个协程** |
| 10 | pkg/kubelet/pod_workers.go | func (p *podWorkers) managePodLoop(podUpdates <-chan podWork) | **每一个pod都会启动一个协程，通过一个与此pod唯一对应的channel和协程进行通信，指导pod的创建/声明周期维护/终止** |
| 11 | pkg/kubelet/kubelet.go | func (kl *Kubelet) syncPod(ctx context.Context, updateType kubetypes.SyncPodType, pod, mirrorPod*v1.Pod, podStatus *kubecontainer.PodStatus) (isTerminal bool, err error) | 此函数可以认为是创建/监控pod等最为核心的函数，此函数是可重入的，这一定非常重要，如果函数执行错误，其可以在下一轮继续执行，以求趋向到期望的状态. 对于我们当前pod正处于待创建的状态， |
| 12 | pkg/kubelet/kuberuntime/kuberuntim_manager.go | func (m *kubeGenericRuntimeManager) SyncPod(pod*v1.Pod, podStatus *kubecontainer.PodStatus, pullSecrets []v1.Secret, backOff*flowcontrol.Backoff)  | 这个函数囊括了创建pod中所有容器的详细流程，其中包括调用func (m *kubeGenericRuntimeManager) startContainer 拉取镜像，创建容器并启动容器 <p> kubelet直接与CRI(container runtime interface)交互完成镜像拉取，容器创建 ,管理存储网络等一些列工作。 <p> 同时会更新对应的event，可以通过kubelet describe pods查看到镜像拉取情况，容器启动情况 |
| 13 | pkg/kubelet/pod_workers.go | func (p *podWorkers) completeWork(pod*v1.Pod, phaseTransition bool, syncErr error) | 每一次执行完一轮(11)中的syncPod操作，上述10中的managePodLoop都会调用13，将此pod重新放入队列中，并且设置60s的时间间隔，60s后会重新执行syncPod。常看容器的状态，是否启动，如果均启动则记录Pod的Status.Condition的变化，然后将Pod从Pending状态变成Running状态。 |

&emsp;&emsp;至此，pod终于完成了启动，并且Status变成了Running。Pod的生命终点一定是被销毁，所以我们继续探索如果当执行了kubectl delete pods <pod_name>后，kubelet是如何终止pod然后做相关清理。同样我们将断点打在4中的写入updates这个channel的部分。

```golang
updates <- kubetypes.PodUpdate{Pods: pods, Op: kubetypes.SET, Source: kubetypes.ApiserverSource}
```

&emsp;&emsp;当我们执行delete命令后，就会监听到pod中中DeletionTimestamp和DeletetionGracePeriodSeconds非空，标志着pod终止的开始。同样再次经过步骤5，将最新的pods和存在内存中的pods进行对比，在发现了某一个pods中上述字段从nil变成非空后，就会将DELETE的信息写入到步骤10的podUpdates，步骤10的 managePodLoop 在读取到PodUpdates消息后，就会开始进入到终止pod的流程。
|  |  |  |  |
|---|---|---|---|
| 14 | pkg/kubelet/kubelet.go | func (kl *Kubelet) syncTerminatingPod(ctx context.Context, pod*v1.Pod, podStatus *kubecontainer.PodStatus, runningPod*kubecontainer.Pod, gracePeriod *int64, podStatusFn func(*v1.PodStatus)) error | 终止各类liveness检测，终止所有pod中的容器 |
| 15 | pkg/kubelet/kubelet.go | func (kl *Kubelet) killPod(pod*v1.Pod, p kubecontainer.Pod, gracePeriodOverride *int64) error  | 在终止的时候，会等待DeletetionGracePeriodSeconds秒，尽力删除所有的pod中的容器 |
| 16 | pkg/kubelet/kubelet.go | func (kl *Kubelet) syncTerminatedPod(ctx context.Context, pod*v1.Pod, podStatus *kubecontainer.PodStatus) | 卸载volume等，最终删除pod内存中的信息，以及其在etcd的记录 |

&emsp;&emsp;终于一个pod从出生到死亡，所有的逻辑都被我们所debug了一遍。在走读debug kubelet的实现时，由于存在非常多的channel通信和协程，所以要跟踪整个信息的轮转确实需要一些不断的尝试，比如可以使用Expression Debug的方法设置条件断点或者修改源码等方法，例如当pod.ObjectMeta.Name和我们创建 pod的名字相同的时候，可以设置断点
