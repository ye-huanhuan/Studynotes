---
description: yarn的产生背景
---

# Yarn基本原理

### yarn的产生背景

由于MRv1设计存在许多问题，而MRv2的出现解决了这些问题。

* **可扩展性不强**，由于JobTrack的任务繁重，既要管理集群资源\(TaskSchduler\)，还要监控集群上的每个任务。
* **可靠性较弱**，MRv1是master/slave结构，并且master是存在单点故障的，所以一旦master挂掉，将导致整个集群不可用。
* **资源利用率较低**，MRv1中资源的描述单位为槽位（slot），并且map task使用的资源和reduce task使用的资源之间不能共享，这就导致任务刚开始的时候只有map task在运行，而reduce task的资源就只能空闲着。
* **不支持多计算框架**，由于MRv1将资源管理和任务管理融合在一起了，而资源管理就是为mapreduce任务而设计的，所以不能很好的支持多计算框架。

正是由于MRv1的设计缺陷，为了解决这些缺陷出现了MRv2；MRv2将资源管理从中抽离出来了，并且为了支持多计算框架，形成了单独的资源管理系统Yarn；而任务管理交给了各个计算框架去实现自己的AppMaster，在yarn上运行的所有任务的AppMaster将会分散到NodeManger中，这样将大大减轻了RM的压力；在资源利用率方面，通过多种计算框架共享集群资源，并且使用队列的方式进行有效的隔离，大大的提升了资源利用率。

### yarn设计思想预计本架构

MRv1是由三部分组成：编程模型（新旧API组成）、数据处理引擎（Map Task和Reduce Task组成）、运行时环境（JobTracker和TaskTracker）。

在MRv2中，为了能够让用户以前的程序迁移到新版本集群，在编程模型上没有进行大的变动，并且兼容旧版本；数据引擎也没有改变。只是运行时环境改变了，将资源管理和作业监控进行了分离，资源管理形成了通用的管理系统yarn，作业管理交给各个计算框架去实现自己AppMaster。

![](/assets/yarn整体架构.JPG)

* ResourceManager：包含资源调度器和应用程序管理器，调度器管理整个集群的资源，根据每个程序的需求根据一定的算法，将资源封装成container分配给AppMaster；而应用程序管理器负责监控各个AppMaster、接受任务提交、与RM申请资源启动AppMaster。
* NodeManager：向RM进行心跳汇报节点健康状况以及该节点上的container运行情况、接受AM的运行与停止任务的请求。
* AppMaster（各类型）：与RM进行通信申请资源、把申请来的资源进一步分配给各个子任务、与NM通信要求启动/停止任务、监控所有运行的任务，如果失败负责重新启动。
* Container：这是资源的一种抽象，包括CPU和内存两个维度；yarn会为每个任务分配一个container。

### 各组件之间的通信协议

在yarn中各个组件之间的通信使用的是RPC协议，而任何两个需要通信的组件之间仅有一个RPC协议，而对于通信双方来说，有一个client和一个server，总是client向server发送请求，server应答这种模式（pull模式）。yarn中主要的RPC协议有以下五个。

* ApplicationClientProtocol：该协议用于JobClient（作业提交客户端）与RM之间的通信，JobClient可以通过该协议提交作业，查询作业运行状况等。
* ResourceManagerAdministrationProtocol：该协议用于Admin与RM之间的通信，系统管理员可以通过该协议更新系统配置，比如节点黑白名单、用户队列权限等。
* ApplicationMasterProtocol：该协议用于AM与RM之间的通信，AM向RM注册和注销自己，并且为任务向RM申请资源。
* ContainerManagementProtocol：该协议用于AM与NM之间的通信，AM通过该协议要求NM启动或者停止container，获取各个container的运行状况。
* ResourceTracker：该协议用于NM与RM之间的通信，NM向RM注册或者注销自己，并且汇报自己节点上的资源使用情况和Container运行状况。

![](/assets/yarn的RPC协议.JPG)

### yarn的工作流程

当用户向RM提交了一个应用程序后，yarn将分两个阶段运行该应用程序；第一阶段是向RM申请资源，然后在分配的NodeManager上启动AppMaster，之后再有AppMaster为具体的任务申请资源，然后要求NM启动任务，并且监控该任务的运行状况，直到完成。

1. 用户向Yarn提交应用程序，包括AppMaster程序、启动AppMaster的命令、应用程序等。
2. RM为该应用程序申请第一个container，然后和对应的NM通信，要求NM在这个container中启动AppMaster。
3. AM启动之后向RM进行注册自己，这样用户就可以通过RM查看任务的运行状况，然后为各个任务申请资源，并监控各个任务，直到任务结束。重复4-7
4. AM采用轮询的方式向RM申请资源，并领取资源。
5. 一旦AM申请到资源之后，便于NM进行通信，要求NM启动任务。
6. NM为任务设置好运行环境（jar包、环境变量等），将任务启动命令写到一个脚本中，并通过该脚本启动任务。
7. 各个任务通过RPC协议向AM汇报自己的运行状况和进度，以便AM随时掌握自己的运行状况，当出错时重启任务。
8. 运行完成之后AM向RM注销自己。

![](/assets/yarn工作流程.JPG)



