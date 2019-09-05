## vcore的概念

* vcore的出现主要是为了屏蔽cpu处理能力的差异的，比如A cpu和B cpu都是只有一个核心，但是Acpu的处理能力是Bcpu的处理能力的两倍，那么我们就可以把Acpu的vcore数设置成Bcpu的vcore数量的两倍；所以vcore的数量并不受物理核心数的影响。 
* 当没有内存隔离的时候，container运行时会占用空闲的cpu的资源，并不会严格按照分配的cpu资源来运行。
* 使用cgroup来进行cpu隔离，并且使用的是hard模式（yarn.nodemanager.linux-container-executor.cgroups.strict-resource-usage=true），这时就规定了container使用cpu资源的上限，计算公式如下：

   container申请的CPU计算公式 = container申请的vcore / 每个节点设置的cvore X 每台机器的物理核 X 每个节点限制的CPU

* 而如果使用cgroup的soft模式，这时，container就会占用节点限制cpu的cpu资源内的空闲资源。
* 如果没有使用cgroup来进行资源隔离，那么每个container使用的cpu资源将无法预估，因为container在运行过程中会占用其他空闲的cpu资源。



