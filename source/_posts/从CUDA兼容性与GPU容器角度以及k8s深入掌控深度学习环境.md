---
title: 从CUDA兼容性与GPU容器角度以及k8s深入掌控深度学习环境
date: 2022-03-17
tags:
    - kubernetes
categories:
    - 云原生
sticky: 10000
description: 太长不想看全文，看这里->：1. CUDA runtime之于driver向后兼容原则，任何指定的CUDA版本都有一个与之对应的最小GPU driver版本。2.GPU的容器来说，容器中的CUDA版本受宿主机的GPU driver限制
---
> 太长不想看全文，看这里->：1. CUDA runtime之于driver向后兼容原则，任何指定的CUDA版本都有一个与之对应的最小GPU driver版本。2.GPU的容器来说，容器中的CUDA版本受宿主机的GPU driver限制

&emsp;&emsp;虽然我并不研究深度学习的算法，但是作为相关训练平台的使用者和搭建者，很多同学都会问我这些问题。“我的训练镜像是用的pytorch是cuda11的，那我的任务在如何才能分配到一台支持cuda11的GPU容器中”，"我使用训练平台1和训练平台2训练任务（因为公司内部有两个完全训练平台，这里分别用1和2表示），可是同样的镜像和代码，在平台2可以正常运行，但是平台1有的时候运行出错"。说真的每次碰到这种问题，我头很大，我不知道应该从什么地方说起，本文就是从一个新手的视角出发，并从兼容性原则和容器方向说说，这些运行时环境相关的问题是怎么决定我们的代码运行的。

# 先问自己两个问题

1. nvidia-smi命令中返回的CUDA Version和Driver Version的关系，以及和我运行机器/镜像中的安装的cuda库版本之间的关系？

![图1](从CUDA兼容性与GPU容器角度以及k8s深入掌控深度学习环境/cuda_driver.jpg)

2. 在容器(例如使用docker创建的容器)中通过GPU训练，容器镜像到底需要安装什么？镜像中需要安装GPU驱动吗？

&emsp;&emsp;我认为很多的由于训练环境导致的训练异常，都可以从这两个问题的回答中找到原因，进而更快的定位异常点。下面我围绕这两个问题，详细的做一个回答。

# 基础概念

&emsp;&emsp;CUDA本质上就是NVIDIA专为通用高性能并行计算设计的一套计算平台和编程模型，换句话使用GPU并行编程的规范方法，所以CUDA在软件层面包含了众多库， 那这里我们用一张图来简单阐述CUDA的各类运行时以及库的关系。

![图2](从CUDA兼容性与GPU容器角度以及k8s深入掌控深度学习环境/cuda_runtime_structure.jpg)

&emsp;&emsp;从最底层开始CUDA Driver（也就是常说的GPU驱动）：可以认为是最底层的操作GPU的接口，作为直接与GPU设备打交道，其编程难度很大，但是性能更好。而CUDA Runtime（也就是常说的CUDA库）：更多是面向CUDA应用开发人员，其API更加简化，可编程性更高，而基于CUDA Runtime接口再向上封装了更多的面向专用计算场景的库，例如专用于深度学习的cuDNN库等。最后，应用层可以使用CUDA Library或者直接使用CUDA Runtime API实现其功能。

&emsp;&emsp;我们都知道想要使用GPU训练程序，那么必须要从nividia官方选择安装对应GPU机型的驱动文件。而官方提供的是一个叫做CUDA toolkit打包的东西，这个本质上是CUDA相关库和工具的集合，例如你如果选择[runfile方式安装](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html#runfile)，从官方下载下来的run文件（eg：cuda_11.0.3_450.51.06_linux.run），本身其中包括了CUDA Runtime（CUDA库），CUDA Driver （GPU驱动），还有样例代码，用户可以通过命令选择，需要安装CUDA库还是GPU驱动，还是说两者都安装。

&emsp;&emsp;另外nvidia-smi本质上是直接使用CUDA driver库，所以说和系统中安装的CUDA Runtime（即CUDA版本）无关.

# 回答上方的问题

## 1. 归根还是CUDA Version/Driver Version/兼容原则

&emsp;&emsp;本质上第一个问题，回答的就是上述驱动和CUDA运行时库之间的兼容性的问题。

&emsp;&emsp;nvidia-smi中显示的CUDA Version本质上是DRIVER API COMPATIBILITY VERSION，换句话理解就是<font color=red>根据机器上当前GPU的Driver驱动版本，CUDA Version显示的是与驱动匹配的最高兼容的CUDA Runtime版本</font>（下文都我们简称CUDA Runtime为CUDA，简单理解就是你可以在机器中安装的cuda动态/静态链接库的最高版本，CUDA driver简称为driver或驱动）。下面从源码/二进制/cubin三个角度具体说说兼容性，这有助于更好的排查“为什么我的训练代码在这台机器上跑不起来”的问题。

### 兼容性原则一：源码级别不兼容性

&emsp;&emsp;所谓源码不兼容很好理解，例如用户的代码是基于cuda 10.1这一特定版本对应的API库构建的，那么如果用户升级到cuda11.0的对应API库，则可能无法正常运行。需要用户根据cuda11.0对应的API文档修改代码再进行编译构建。所以我们可以看到pytorch，针对不同的cuda版本，都有对应不同的编译后的库，例如下面两个就是分别基于cuda11.1（[torch-1.10.1%2Bcu113-cp37-cp37m-linux_x86_64.whl](https://download.pytorch.org/whl/cu113/torch-1.10.1%2Bcu113-cp37-cp37m-linux_x86_64.whl)）和cuda11.3（[torch-1.10.0%2Bcu111-cp36-cp36m-linux_x86_64.whl](https://download.pytorch.org/whl/cu111/torch-1.10.0%2Bcu111-cp36-cp36m-linux_x86_64.whl)）不同的cuda版本构建的。

### 兼容原则二：后向兼容

&emsp;&emsp;后向兼容的意思是：如果一个程序使用的CUDA版本可以在某一Driver版本下运行，那么在升级了Driver后，此程序在保持原CUDA版本的情况下，仍然可以在新的更高版本的驱动下运行。换句话说，某一具体的cuda版本存在与之对应的最小驱动版本。而对于cuda11和cuda10这两个主版本下，兼容的情况也有细微的却别。兼容性对照表可以查看。

&emsp;&emsp;<font color=red>1. 对于cuda11主版本 </font>（cuda版本是X.Y.X三段式，其中X为主版本号，Y为次版本号），那么对于以11开头的所有CUDA版本来说，只要driver版本>=450.80.02*, 则即可满足所有的CUDA11.0,11.1,11.2等以11.x开头的CUDA运行时版本。这种兼容模式称作为为次版本兼容（Minor Version Compatibility）。当然这种兼容是“limited feature-set”，换句话说满足在保持驱动不变下，升级cuda版本后，运行不出错，但是对于一些高版本的cuda的特性，如果要更好的使用或者性能，也需要升级driver驱动。比如对于cuda11.2，官方的cuda toolkit包中推荐安装的driver是>=460.00。

&emsp;&emsp;回到上面的问题1，我们用的云开发GPU是一个vGPU实例，把Tesla T4 从一个GPU虚拟化出两个vGPU分配给两台虚拟机，nvidia-smi显示Driver Version为450.102.04，而CUDA Version显示的是11.0，通过上文的说明，可以发现此虚拟机支持包括11.x在内的所有cuda11版本，而CUDA Version显示的可以认为是最高兼容的CUDA“主版本”。这里验证的方法也很简单，可以在云开发GPU机器中安装任意的cuda10.x/cuda11.x，通过编译cuda sampler示例中的deviceQuery程序验证。

| cuda10.1    | cuda11.0 | cuda11.2 |
| -------- | ------- | ------- |
| ![cuda10.1](从CUDA兼容性与GPU容器角度以及k8s深入掌控深度学习环境/cuda10.1.jpg)    | ![cuda11.0](从CUDA兼容性与GPU容器角度以及k8s深入掌控深度学习环境/cuda11_0.jpg) | ![cuda11.2](从CUDA兼容性与GPU容器角度以及k8s深入掌控深度学习环境/cuda11_2.jpg)|

&emsp;&emsp;<font color=red>2. 对于cuda10这主版本</font>，每一个cuda10.x的版本都有与之对应的最小驱动版本号，例如下图是截取自[CUDA Compatibility :: NVIDIA Data Center GPU Driver Documentation](https://docs.nvidia.com/deploy/cuda-compatibility/)。可以看到cuda10.0/10.1/10.2对应的最小满足的版本号均不一样，不同于cuda11.x，可以在驱动不变的情况下升级cuda，cuda10.x，想要升级从10.1升级到10.2，那么驱动版本必须要大于等于440.33。
![cuda_table](从CUDA兼容性与GPU容器角度以及k8s深入掌控深度学习环境/cuda_table.jpg)

&emsp;&emsp;当我们公司的训练平台1创建一个"所谓的"cuda_version版本为10.2的任务，那么分配给我们的机器对应的driver版本号为440.64.00，那么假如我们安装cuda11.2，则显而易见编译后运行上文的deviceQuery程序会返回错误。

![device_query](从CUDA兼容性与GPU容器角度以及k8s深入掌控深度学习环境/devicequery.jpg)

&emsp;&emsp;而对于GPU来说，虚拟机中的驱动都是450.102.04，因此可以支持任意cuda11及以下的CUDA Runtime。

### 兼容原则三：有限的前向兼容

&emsp;&emsp;后向兼容是需要在cuda升级后，驱动也需要根据要求进行升级（或者不变）。而前向兼容的意思就是，在cuda升级后，driver不需要对内核态相关包进行升级，而只需要变更相关用户态文件即可。目的就是可以在老旧驱动上基于新的cuda版本编译程序，从而获取到最新的cuda特性。 而为什么说是有限的兼容，主要表现在两点限制：1. 限制了GPU卡的类型，只有NVIDIA Data Center/Tesla 系列（和小部分特殊的RTX）的GPU卡. 2. 前向兼容的能力理论上只有在需要跨cuda主版本的时使用，例如本来最高只支持cuda版本10.1的Driver418，可以通过安装正确的Compat Package,使其在不更新内核态驱动的情况，支持cuda10.1~cuda11.6。具体可以参考官方文档的前向兼容矩阵，来下载安装对应的兼容包。[CUDA Compatibility :: NVIDIA Data Center GPU Driver Documentation](https://docs.nvidia.com/deploy/cuda-compatibility/)

### 兼容性原则四：cuda应用程序编译产物与不同GPU架构间的兼容

&emsp;&emsp;这部分的兼容性原则理解起来，需要涉及到cuda应用程序编译的相关知识。一个写好的cuda程序，通过nvcc编译后的产物可以包含两种形式，一个是二进制的cubin对象，另一个是PTX（Parallel Thread Execution）汇编代码。

&emsp;&emsp;cubin是特定于指定的GPU架构的，cubin二进制对象对于GPU架构的计算能力（计算能力只是代表一个GPU的能力特性与性能高低无关）是一个向后兼容的，并且对GPU计算能力也是类似Minor Version Compatibility，换句话说，为计算能力为X.y的GPU生成的cubin对象，只能在计算能力为X.z且z>=y的GPU上运行。举个例子：为7.0计算能力生成的cubin，可以在7计算能力为7.5的GPU上执行，但是无法在计算能力为8.0的GPU上执行。

&emsp;&emsp;那对于编译成PTX形式的产物，在cuda应用程序运行加载时，会先由设备驱动程序进一步把PTX通过JIT技术（即时编译）编译成对应GPU架构或者计算能力的cubin，这也就意味着此PTX可以在计算能力高于当前生成的此PTX计算能力的GPU上运行。关于更多JIT的内容可以参考：[Programming Guide :: CUDA Toolkit Documentation](https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#just-in-time-compilation)

&emsp;&emsp;因此，如果一个cuda应用程序在编译时选择包含PTX相关产物，“理论上”可以更好的保证在GPU架构升级后，代码仍然可以兼容运行，换句话说，"理论"上一个原先使用cuda10.x编译且可以在Volta架构V100上运行的应用，选择选择生成PTX二进制代码，那么可以在Ampere架构的A100上运行。

&emsp;&emsp;但是回到一个具体的案例，事实上对于pytorch，由于受制于使用的cuDNN与GPU架构升级的兼容的原因（cuDNN7与Ampere架构不兼容），以及pytorch使用pip wheel安装或者conda安装（pytorch在编译过程根据不同的安装方式会选择不同的编译模式，例如conda安装会选择使用包含PTX的二进制版本，而pip wheel安装可能不会包含），想要使用A100机器训练，必须升级到cuda11且cuDNN8以上版本的pytorch来可以使用。

&emsp;&emsp;换句话说，GPU的架构在一定程度上限制了cuda的版本（注：计算能力只是代表一个GPU的能力特性与性能高低无关）关于更多关于编译链接的内容，可以参考官网文档：[NVCC :: CUDA Toolkit Documentation](https://docs.nvidia.com/cuda/cuda-compiler-driver-nvcc/index.html)

## 2. 归根还是容器中"挂载"宿主机的"文件"

&emsp;&emsp;要回答第二个问题，其实就是理解我们训练时候容器中CUDA 运行时所用的GPU驱动（或者说是CUDA driver API）到底是宿主机的驱动文件还是镜像中的驱动文件。这里可以用云开发GPU机器上的docker来作为观察对象。

&emsp;&emsp;我使用的云开发GPU机器中已经安装了的docker，事实上是把原来底层用来通过操作系统调用创建运行容器的“runc”组件替换为nvidia-container-runtime组件（关于runC的一些概念，可以参考之前的[从kubernetes中容器生态窥探设计模式的哲学](./从kubernetes中容器生态窥探设计模式的哲学.html)），当然nvidia-contianer-runtime本质上是一个做了修改后的runc组件，区别是它增加了一个自定义的prestart hook，目的是在创建容器后，在启动容器前，调用这个hook，而这个hook本身做的就是一些类似将宿主机的device/driver文件等挂载进容器中。下图为NVIDIA官网介绍NVIDIA Container的大致架构组件图。
![gpu_container](从CUDA兼容性与GPU容器角度以及k8s深入掌控深度学习环境/gpu_container.jpg)

&emsp;&emsp;那到底具体将宿主机的哪些设备文件挂载进了容器呢。我们可以打开nvidia-container-runtime的debug功能，详细在其日志中查看所有文件设备挂载列表，具体为修改/etc/nvidia-container-runtime/config.toml文件

```xml
[nvidia-container-cli]
environment = []
debug = "/var/log/nvidia-container-toolkit.log"
load-kmods = true
ldconfig = "@/sbin/ldconfig"
[nvidia-container-runtime]
debug = "/var/log/nvidia-container-runtime.log"
```

&emsp;&emsp;打开debug功能后，我们重新通过docker 启动一个容器

```
docker run  --rm --gpus '"device=0"' --net host  -it mirrors.tencent.com/shadow_test_xiaobaihe/test_for_light:torch_ptx /bin/bash
```

&emsp;&emsp;启动成功后，我们发现可以使用nvidia-smi命令查看挂载进容器的GPU情况。明明我的镜像中没有nvidia-smi这个二进制程序，为什么启动后文件就可以直接使用呢？那么秘密事实上就在nvidia-container-toolkit这个prehook内帮我们完成了。打开上方的/var/log/nvidia-container-toolkit.log文件，可以详细的查询到整个hook过程。

&emsp;&emsp;其中我们发现，hook过程中向容器中注入了包括宿主机的二进制工具，例如nivida-smi/nvida-debugdump等，宿主机的上的库，例如很重要的CUDA Driver API库libcuda.so。另外还有很重要的是在宿主机中通过mknod创建所需的nvidia相关的设备文件，并将宿主机的文件设备文件注入到容器中。

```
# 注入宿主机的二进制程序

I0311 03:09:13.228302 19802 nvc_mount.c:112] mounting /usr/bin/nvidia-smi at /data/dockerimages/overlay2/05f25c9dde0a3cad98c5ec03e78fbd25ce10eb4ac52aeccac393d6645220770f/merged/usr/bin/nvidia-smi
I0311 03:09:13.228326 19802 nvc_mount.c:112] mounting /usr/bin/nvidia-debugdump at /data/dockerimages/overlay2/05f25c9dde0a3cad98c5ec03e78fbd25ce10eb4ac52aeccac393d6645220770f/merged/usr/bin/nvidia-debugdump

# 注入宿主机的CUDA Driver库

I0311 03:09:13.228463 19802 nvc_mount.c:112] mounting /usr/lib64/libcuda.so.450.102.04 at /data/dockerimages/overlay2/05f25c9dde0a3cad98c5ec03e78fbd25ce10eb4ac52aeccac393d6645220770f/merged/usr/lib64/libcuda.so.450.102.04
I0311 03:09:13.228484 19802 nvc_mount.c:112] mounting /usr/lib64/libnvidia-opencl.so.450.102.04 at /data/dockerimages/overlay2/05f25c9dde0a3cad98c5ec03e78fbd25ce10eb4ac52aeccac393d6645220770f/merged/usr/lib64/libnvidia-opencl.so.450.102.04

# 创建设备文件，并将宿主机设备文件注入到容器中

I0311 03:09:13.207136 19807 nvc.c:282] running mknod for /dev/nvidia0
I0311 03:09:13.228019 19802 nvc_info.c:705] listing device /dev/nvidia0 (GPU-40143293-c4ff-11eb-ba91-04c440212a27 at 000000    00:00:09.0)
I0311 03:09:13.280933 19802 nvc_mount.c:208] mounting /dev/nvidia0 at /data/dockerimages/overlay2/05f25c9dde0a3cad98c5ec03e    78fbd25ce10eb4ac52aeccac393d6645220770f/merged/dev/nvidia0
```

&emsp;&emsp;由此可以看到在使用nvidia-contiainer-runtime这种容器使用GPU的解决方案方案下，容器中使用CUDA Driver还有nvidia-smi都是来自于宿主机的，不需要在镜像中安装CUDA Driver。而如果在镜像中包含了CUDA driver库，可能会导致容器在hook过程中，在建立libcuda.so软链时，使用镜像中的driver库，从而可能触发上文说的"前向兼容"流程（即有可能镜像中使用的用户态的driver驱动高于宿主机的内核态的启动，从而使得GPU认为应该用前向兼容），而往往前向兼容是比较有限的，受制于GPU机型，还有驱动版本等，从而导致报错，例如可能出现forwoard compatibilty报错。

# 回到训练平台

&emsp;&emsp;根据第二个问题的答案，这就是为什么我们在各种平台训练的时候，镜像中都只需要打包CUDA Runtime和CUDA Libraries即可，而不需要存在驱动的相关库。因为训练容器使用的本身是宿主机上的驱动库。

&emsp;&emsp;而至于为什么同样的镜像和代码，在平台2可以正常运行，但是在平台1有的时候运行出错。这主要是因为不同平台所处集群中的机器驱动并不都是最新的，有一些机器驱动还是440只能到最高支持cuda10.2，而有的机器的驱动已经是450，可以支持到cuda11.x，而根据上文说的兼容性原则，如果镜像中使用的CUDA runtime库是cuda11，而实际上平台分配到的机器驱动只有440，那么根据上述的向后兼容原则，肯定是不行的，所以需要在训练平台1中强制指定cuda_version参数为11.0。平台2总是支持的原因，也在于其GPU宿主机的机器驱动比较新，都是支持cuda11.x的。

## 在训练平台中训练任务

&emsp;&emsp;现在往往大多数公司的训练平台都是基于kubernetes平台所搭建的，甚至在一个公司内部下的GPU集群是由多个k8s集群所组成的，在其之上，不同部门都会搭建自己不同的训练平台。例如本例中的平台1和平台2，其算力都来自于最底层的k8s集群。不同的训练平台，可以看做是不同的k8s租户，通过不同的的方案，来实现一个训练任务的资源创建。

&emsp;&emsp;在一年前，大模型还没有像如今这么繁荣的情况下，主要使用的k8s中的[Operator](https://www.redhat.com/zh/topics/containers/what-is-a-kubernetes-operator)来定制我们训练任务的多机多卡Pod以及网络的等组合方式，使用kubeflow/mpi-operator方式，来创建满足all-reduce方式的通用任务。通过mpi-operator并配合k8s的CRD（CRD--custom resource definition）来引用MPIJob这个新的对象类型，换句话说训练平台在创建一个任务后通过Operator创建对应CR(custom resource)进行调度。这种方法更接近与云原生的kubeflow，同事可以通过Operator方式，扩展使用Pytorch-Operator/Tensorflow-Operator等（目前的mpi-operator服务于类horovod并行训练）。
![k8s_operator](从CUDA兼容性与GPU容器角度以及k8s深入掌控深度学习环境/k8s_operator_structure.png)

# 附录

1. [CUDA Compatibility :: NVIDIA Data Center GPU Driver Documentation](https://docs.nvidia.com/deploy/cuda-compatibility/)
2. [https://forums.developer.nvidia.com/t/how-to-use-cuda-compatibility-package-to-use-a-newer-driver-on-an-older-kernel-module/77533/4](https://forums.developer.nvidia.com/t/how-to-use-cuda-compatibility-package-to-use-a-newer-driver-on-an-older-kernel-module/77533/4)
3. [Best Practices Guide :: CUDA Toolkit Documentation](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html#cuda-compatibility-and-upgrades)
4. [Architecture Overview &mdash; NVIDIA Cloud Native Technologies documentation](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/arch-overview.html#arch-overview)
