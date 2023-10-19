---
title: 手把手DEBUG(第三篇:pod怎么被调度到一个node上)
date: 2023-05-23 16:42:51
tags:
    - kubernetes
categories:
    - 云原生
sticky: 998
description: 本文围绕k8s中kube-scheduler这个组件，详细说明我是怎么来学习和理解k8s中关于pod的调度逻辑的。
---
更详细的debug操作可以参考我的[手把手一起DEBUG-Kubernetes-第一篇-建立k8s的DEBUG环境](手把手一起DEBUG-Kubernetes-第一篇-建立k8s的DEBUG环境)

# 1.编译kube-scheduler二进制

```
make all WHAT=cmd/kube-schedule DBG=1
```

# 2.终止默认的kube-scheduler对应 的pods

kube-scheduler进程是static pods，是直接由kubelet管理，不经过apiserver, kubelet会监控/etc/kubernetes/manifests/下通过yaml定义的pods，所以可以直接将 kube-scheduler.yaml移除当前目录,即可停止原来的kube-scheduler

```
mv /etc/kubernetes/manifests/kube-scheduler.yaml <其他目录>
kubectl get pods -n kube-system
```

这时候在执行查询命令已经看不到kube-scheduler对应的pods

# 3.创建一个最基本的pod

由于我们已经将kube-scheduler对应的pod移除，所以创建的pod的状态始终是pending状态，这里我用了一个非常简单的pod

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-centos
  labels:
    name: myapp-centos
    app: myapp-centos
spec:
  containers:
    - name: myapp
      image: centos
      resources:
        limits:
          memory: "1024Mi"
          cpu: "500m" # 等同于0.5cpu
      imagePullPolicy: Always
      command: ["sh", "-c", "sleep infinity"]
```

# 4.通过vscode开始对kube-scheuler进行debug

配置如下，注意一定要禁用leader-election, 这样可以防止debug时中断比较久导致leader失效，造成程序自动终止的情况。

```
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Launch local executable kube-scheduler",
            "type": "go",
            "request": "launch",
            "debugAdapter": "dlv-dap",
            "mode": "exec",
            "program": "${workspaceFolder}/_output/bin/kube-scheduler",
            "substitutePath": [
                {
                    "from": "${workspaceFolder}",
                    "to": "${workspaceFolder}/_output/local/go/src/k8s.io/kubernetes",
                },
            ],
            "args": [
                "--authentication-kubeconfig=/etc/kubernetes/scheduler.conf",
                "--authorization-kubeconfig=/etc/kubernetes/scheduler.conf",
                "--bind-address=127.0.0.1",
                "--kubeconfig=/etc/kubernetes/scheduler.conf",
                "--leader-elect=false"
            ]
        }
    ]
}
```

# 5.设置断点代码调试与走读

&emsp;&emsp; 在kube-scheduler pod终止后，我们在上面章节创建的pods的Status会始终处于Pending状态，这个时候就可以启动vscode进行debug。
kube-scheduler从cmd/kube-scheduler/server.go入口的main函数启动后，可以将更多断点加载pkg/scheduler/sceduler_one.go中。

&emsp;&emsp;scheduler进程简单的理解就是一个大的不停机的for循环，<font color=red>watch所有处于未调度状态的pods</font>(Pending状态的pods是其中最简单的一种)，每次循环都从未调度的队列中取出一个pod执行调度逻辑，而每次循环的结束，就是以调用/api/v1/namespaces/default/pods/{pod_name}/binding restful APIw为结束，将此pod和某一个满足要求的评分最高的Node进行绑定，并设置**Pod.Conditions**的PodScheduled为true，至于剩下Pods的启动逻辑就交给绑定的Node上的kubelet来处理了，kubelet会依次将启动Pods中的容器，然后进一步的修改Pod的Conditions信息。下面的表格梳理了需要走读的关键调用链对应的函数（这里代码走读不要陷入类似informer/indexer的细节中，把握主要逻辑点，后续如果想了解更多informer等细节的处理，可以参考附录中的文章。）

| |文件|关键函数|相关解释|
|:----|:----|:----|:----|
|1|pkg/scheduler/sceduler.go|func (sched *Scheduler) Run(ctx context.Context) |这个函数本质是持续不断地执行sched.scheduleOne 函数，如果没有pod需要调度，就会阻塞住|
|2|pkg/scheduler/sceduler_one.go|func (sched *Scheduler) scheduleOne(ctx context.Context)|整个函数做的事情就是取一个未调度的Pod，然后决定将其放在哪个Node上|
|3|pkg/scheduler/internal/queue/scheduling_queue.go|func MakeNextPodFunc(queue SchedulingQueue) func() *framework.QueuedPodInfo|从待调度的队列中pop出待调度的pod|
|4|pkg/scheduler/sceduler_one.go|func (sched *Scheduler) schedulePod(ctx context.Context, fwk framework.Framework, state*framework.CycleState, pod *v1.Pod) (result ScheduleResult, err error)|包含了预选和优选的逻辑。预选就是先对所有Nodes进行过滤只留下符合用户定义的Node.优选就是在剩下的Node中根据负载等当前节点的资源使用情况选择最好的节点。|
|5.1|pkg/scheduler/sceduler_one.go|func (sched *Scheduler) findNodesThatFitPod(ctx context.Context, fwk framework.Framework, state*framework.CycleState, pod *v1.Pod) ([]*v1.Node, framework.Diagnosis, error) |预选过程：依次对nodes检查是否满足pod亲和性，是否达到了资源最小要求(pod中所有容器的cpu等资源相加)，node上挂载的pv是否满足pvc要求|
|5.2|- |func (sched *Scheduler) findNodesThatPassFilters(...)| - |
|6|pkg/scheduler/sceduler_one.go|func prioritizeNodes(...) (framework.NodeScoreList, error)|优选过程：通过对每个过滤后的节点执行多个打分插件，将多个插件针对一个节点的分数相加，然后和其他节点的分数进行比较，选择最高分的节点|
|7|pkg/scheduler/sceduler_one.go|func (sched *Scheduler) bind(ctx context.Context, fwk framework.Framework, assumed*v1.Pod, targetNode string, state *framework.CycleState) (err error) |执行restful API POST-->https://<api-server>/api/v1/namespaces/default/pods/<pod_name>/binding|
|8|pkg/scheduler/framework/plugins/defaultbinder/default_binder.go|func (b DefaultBinder) Bind(ctx context.Context, state *framework.CycleState, p*v1.Pod, nodeName string) *framework.Status| pods/binding事实上是pods这个资源的subresource，而post请求事实上就是将更新Pod的Nodes的调度信息以及Pod的Conditions信息，写入etcd中，之后创建Pod和其容器的职责就转交给对应Node的kubelet了|

&emsp;&emsp;这里面有非常多的细节，比如待调度的pods的队列如何设计，如何利用informer缓存和监听未调度的pods，还有预选和优选的更多的细节，例如如何通过扩展更多"插件"，来自定义预选优选等逻辑，定制调度。

&emsp;&emsp;调度的模块简化为三个步骤：1. 预选：过滤掉无法满足pods需求的节点 2. 优选：在预选阶段剩下的节点进行排序 3. 选择：选择最高分的node进行node和pods的绑定。主要关注pod状态，或者Pods中Status信息的变化，以及选择到的Node与Pod的绑定逻辑，选择Node算法或者调度算法细节可以涉及到比较多细节不做展开。

&emsp;&emsp;可以看到整个scheduler的逻辑是k8s组件中最为单一，也是最为好理解的，也是留给各个云厂商灵活性很大的一个组件，它支持更多调度的扩展，使用户根据集群中应用的场景，设计符合自身的调度算法。

# 附录

更加细节的kube-scheduler的设计思路可以查阅下面两篇官方的文档，详细阐述了设计思想

1. [Scheduling Framework | Kubernetes](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)（主要介绍了从v1.19版本之后，pod调度的设计逻辑，网上有很多源码阅读都是v1.19之前版本的所以会和近期的版本有比较大的出入）
2. [从代码角度详细描述了如何整合各种队列，cache，插件等的方式组装一个可以自定义的调度器](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-scheduling/scheduling_code_hierarchy_overview.md)（其中的代码会和最新版本略有出入但是整体逻辑相同）
3. [详细描述Scheduler中队列的设计包括通过多个队列来处理调度和调度失败情况下pod的重试问题](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-scheduling/scheduler_queues.md)
