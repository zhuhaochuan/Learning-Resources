#### 谷歌的Borg系统是一个集群管理器，可以运行数千个不同的应用程序中的数十万个作业，这些作业分布在多个集群中，每个集群最多成千上万的机器。


- 三个优点

Borg provides three main benefits: 

it (1) hides the details of resource management and failure handling so its users can
focus on application development instead;

 (2) operates with
very high reliability and availability, and supports applications that do the same; 

and (3) lets us run workloads across
tens of thousands of machines effectively. 



![1551781849470](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1551781849470.png)

分为：

- 用户视角
- 负载



负载分两种 一种是面向终端用户的 需要长期运行的 低延时的应用  一种是需要一定时间运行处理之后结束的应用  这个对于短时间的表现要求不高

许多上层的应用是基于borg开发的，都是含有controller层 去提交主要任务或者job的。

“production” (prod)   为什么叫pods  是因为 从这里来的吧  ？

Most long-running server jobs are prod; most batch jobs are non-prod. 

单元中的计算机属于单个集群，由高性能数据中心规模的网络结构连接他们。



#### job

A Borg job’s properties include its name, owner, and the number of tasks it has 

几乎所有的Borg的task都会包含一个内置的HTTP服务，用来发布健康信息和几千个性能指标

Borg监控这些健康检查URL，把其中响应超时的和error的task重启。其他数据也被监控工具追踪并在Dashboard上展示，当服务级别对象（SLO）出问题时就会报警。

这个就是我之前的操作吧 就是需要监控task的状态 但是如何获取呢 



#### 架构

A Borg cell consists of a set of machines, a logically centralized controller called the Borgmaster, and an agent process called the Borglet that runs on each machine in a cell (see Figure 1). All components of Borg are written in C++ 

就是说 有一个Borgmaster 以及每一个cell上都会有一个Borglet  



#### Borgmaster 

含有两个进程：

主进程 处理 RPC 获取状态 或者提供 获取只读数据的方法 ，同时管理所有系统当中的对象的状态机，与Borglets 的通信 提供前端的UI

实际上使用了多个进程副本 每个副本在内存当中保存一份所有的cell的状态 同时使用一个高可用分布式的使用Paxos一致性算法的副本的本地存储存下。



job被提交到borg后，被存放到Paxos stroe中，同时将对应的task添加到pending队列中。scheduler异步的扫描pending队列，为task挑选出合适的机器。scheulder优先处理高优先级的task，对优先级相同的task，按照它们所属的job，使用Round Robin策略。调度，即挑选node的过程，分为两步：选出符合条件的node（feasibility checking），对node进行评分（scoring）。



两种极端策略，一种是将task尽可能的分配到所有的node上，一种是将一个node尽可能装满。 第一种方式，会导致剩余资源碎片化，第二种方式中task聚合的太紧密，负载突增时影响较大，并且优先级较低非生产任务不能充分利用闲置的资源。 Borg的评分细节没有公开，只知道是以上两种极端策略的折衷。



在Borg中，Job是用来对task进行分组的唯一方式，这种方式相当不灵活，导致用户自己发展出很多方式来实现更灵活对分组。 Kuberntes吸取了Borg的这个教训，引入了Labels，使Pod可以被灵活地分组。



在Borg中，一个node上的所有task共享node的ip，这直接导致端口也成为一种资源，在调度时候需要被考虑。Kubernetes为每个Pod分配独立的IP。





### k8s

- Kubernetes 项目带来的体验是确定的：现在我有了应用的容器镜像，请帮我在一个给定的集群上把这个应用运行起来
-  Kubernetes 能给我提供路由网关、水平扩展、监控、备份、灾难恢复等一系列运维能力。



master结点：如何编排、管理、调度用户提交的作业

**从一开始，Kubernetes 项目就没有像同时期的各种“容器云”项目那样，把 Docker 作为整个架构的核心，而仅仅把它作为最底层的一个容器运行时实现**

运行在大规模集群中的各种任务之间，实际上存在着各种各样的关系。这些关系的处理，才是作业编排和管理系统最困难的地方。



但容器技术出现以后，你就不难发现，在“功能单位”的划分上，容器有着独一无二的“细粒度”优势：毕竟容器的本质，只是一个进程而已

一旦要追求项目的普适性，那就一定要从顶层开始做好设计

**Kubernetes 项目最主要的设计思想是，从更宏观的角度，以统一的方式来定义任务之间的各种关系，并且为将来支持更多种类的关系留有余地。**

这个留有余地是很关键的 设计上为以后的变化留有余地。

#### secret

Kubernetes 项目提供了一种叫作 Secret 的对象，它其实是一个保存在 Etcd 里的键值对数据。这样，你把 Credential 信息以 Secret 的方式存在 Etcd 里，Kubernetes 就会在你指定的 Pod（比如，Web 应用的 Pod）启动时，自动把 Secret 里的数据以 Volume 的方式挂载到容器里。这样，这个 Web 应用就可以访问数据库了



#### **应用运行的形态**

为此，Kubernetes 定义了新的、基于 Pod 改进后的对象。比如 Job，用来描述一次性运行的 Pod（比如，大数据任务）；再比如 DaemonSet，用来描述每个宿主机上必须且只能运行一个副本的守护进程服务；又比如 CronJob，则用于描述定时任务等等。



#### 声明式API

Kubernetes 最核心的设计理念

 通过一个“编排对象”，比如 Pod、Job、CronJob 等，来描述你试图管理的应用

再为它定义一些“服务对象”，比如 Service、Secret、Horizontal Pod Autoscaler（自动水平扩展器）等。这些对象，会负责具体的平台级功能



###### Kubernetes 项目所擅长的，是按照用户的意愿和整个系统的规则，完全自动化地处理好容器之间的各种关系。**这种功能，就是我们经常听到的一个概念：编排。**



**容器技术的核心功能，就是通过约束和修改进程的动态表现，从而为其创造出一个“边界”**

**Cgroups 技术**是用来制造约束的主要手段，而**Namespace 技术**则是用来修改进程视图的主要方法

对被隔离应用的进程空间做了手脚，使得这些进程只能看到重新计算过的进程编号

 Namespace 的使用方式也非常有意思：它其实只是 Linux 创建新进程的一个可选参数。我们知道，在 Linux 系统中创建线程的系统调用是 clone()

**PID Namespace，Linux 操作系统还提供了 Mount、UTS、IPC、Network 和 User 这些 Namespace，用来对各种不同的进程上下文进行“障眼法”操作**

1. UTS: 主机名（本文介绍）
2. IPC: 进程间通信 （之后的文章会讲到）
3. PID: "chroot"进程树（之后的文章会讲到）
4. NS: 挂载点，首次登陆Linux（之后的文章会讲到）
5. NET: 网络访问，包括接口（之后的文章会讲到）
6. USER: 将本地的虚拟user-id映射到真实的user-id（之后的文章会讲到）

使用 Namespace 作为隔离手段的容器并不需要单独的 Guest OS

**“敏捷”和“高性能”是容器相较于虚拟机最大的优势，也是它能够在 PaaS 这种更细粒度的资源管理平台上大行其道的重要原因。**

**Linux Cgroups 就是 Linux 内核中用来为进程设置资源限制的一个重要功能。**

**它最主要的作用，就是限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等**

Cgroups 给用户暴露出来的操作接口是文件系统，即它以文件和目录的方式组织在操作系统的 /sys/fs/cgroup 路径下



**容器里的进程看到的文件系统又是什么样子的呢**

即使开启了 Mount Namespace，容器进程看到的文件系统也跟宿主机完全一样

**它对容器进程视图的改变，一定是伴随着挂载操作（mount）才能生效。**

在容器进程启动之前重新挂载它的整个根目录“/”。而由于 Mount Namespace 的存在，这个挂载对宿主机不可见，所以容器进程就可以在里面随便折腾了

一个名为 chroot 的命令可以帮助你在 shell 中方便地完成这个工作。顾名思义，它的作用就是帮你“change root file system”，即改变进程的根目录到你指定的位置

**Mount Namespace 正是基于对 chroot 的不断改良才被发明出来的，它也是 Linux 操作系统里的第一个 Namespace**

会在这个容器的根目录下挂载一个完整操作系统的文件系统，比如 Ubuntu16.04 的 ISO。这样，在容器启动之后，我们在容器里通过执行 "ls /" 查看根目录下的内容，就是 Ubuntu 16.04 的所有目录和文件。

**而这个挂载在容器根目录上、用来为容器进程提供隔离后执行环境的文件系统，就是所谓的“容器镜像”。它还有一个更为专业的名字，叫作：rootfs（根文件系统）。**

进入容器之后执行的 /bin/bash，就是 /bin 目录下的可执行文件，与宿主机的 /bin/bash 完全不同



docker 三步骤

为待创建的用户进程：

1. 启用 Linux Namespace 配置；
2. 设置指定的 Cgroups 参数；
3. 切换进程的根目录（Change Root）。



**rootfs 只是一个操作系统所包含的文件、配置和目录，并不包括操作系统内核。在 Linux 操作系统中，这两部分是分开存放的，操作系统只有在开机启动时才会加载指定版本的内核镜像。**

**由于 rootfs 里打包的不只是应用，而是整个操作系统的文件和目录，也就意味着，应用以及它运行所需要的所有依赖，都被封装在了一起。**

是啊！！！！！  环境都在这些文件当中  这是核心！！！

Docker 在镜像的设计中，引入了层（layer）的概念。也就是说，用户制作镜像的每一步操作，都会生成一个层，也就是一个增量 rootfs。联合文件系统（Union File System）的能力。

联合挂载！！！！

分为：

- 只读层
- init层  也是只读的但是 有一些变量需要用户以环境变量的方式 注入  但是不希望被一起提交 所以就是单独挂载 用户commit的时候 只会提交可读写层 是不包含init层的。
- 可读可写层

whiteout  就是对应删除文件，不是真正的删除这个文件而是 遮挡这个文件不让用户看见，whiteout 形象地翻译为：“白障”。

读写层可以被push到镜像仓库上，同时保持原始的只读层不变。

最上面这个可读写层的作用，就是专门用来存放你修改 rootfs 后产生的增量，无论是增、删、改，都发生在这里。而当我们使用完了这个被修改过的容器之后，还可以使用 docker commit 和 push 指令，保存这个被修改过的可读写层，并上传到 Docker Hub 上，供其他人使用；而与此同时，原先的只读层里的内容则不会有任何变化。这，就是增量 rootfs 的好处。



