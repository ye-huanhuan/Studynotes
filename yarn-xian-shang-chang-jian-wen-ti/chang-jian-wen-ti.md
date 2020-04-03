# 常见问题

## 内存超限问题

### 问题描述：

spark on yarn任务出现内存超出限制错误，如图所示

![](../.gitbook/assets/chang-jian-cuo-wu-nei-cun-chao-xian.jpg)

### 问题原理：

* 客户端提交时指定 --executor-memory 8g，预计分配给executor的内存=executorMemory\(堆内内存\)+executorMemoryOverhead\(堆外内存\)
* 堆外内存在不指定时，默认为：Max{MEMORY\_OVERHEAD\_FACTOR\(默认0.1\) \* executorMemory，MEMORY\_OVERHEAD\_MIN\(默认384M\)}
* 然而，因为yarn对于内存的管理粒度问题；这里还涉及到规整因子\(yarn.scheduler.increment-allocation-mb配置，洛阳集群配置的是512M\)
* 所以，最终分配给executor的内存=ceil\(\(堆内内存+堆外内存\)/规整因子\)\*规整因子，而不管是executor内存超限还是虚拟内存超限都会出现该问题。

### 解决方式：

知道了--executor-memory配置的内存与最终执行时能够使用的内存量的关系，这样根据自己的程序适当的增加--executor-memory指定的内存量

## GC超限问题

### 问题描述：

Reduce Task出现GC超限，报出OOM，如图所示

![](../.gitbook/assets/chang-jian-cuo-wu-gc-chao-xian.JPG)

### 问题原理：

1. 集群默认配置的jvm heap内存为200m（mapred.child.java.opts指定）。
2. jvm为了防止堆过小，应用一直长时间运行但是进度很小或者根本没有进度这种情况，启动时默认开启了推测特性（-XX:+UseGCOverheadLimit指定），该特性会检测GC时间和GC回收的空间，如果出现收回时间超过98%并且回收空间少于堆空间的2%，那么将会抛出OOM异常。
3. 并且大部分OOM出现在reduce端，因为业务数据分布不均导致，某个task的数据量明显大于其他task。

### 解决方案：

1. 调整jvm heap的大小。（不能超过每个任务分配的内存，集群默认是16G；并且该方式可能导致集群负载过高）
2. 业务方调整程序逻辑，防止数据严重倾斜。

## 用户程序错误

### 问题描述：

由于用户提交的程序存在错误，任务在执行过程中抛出异常，导致任务终止。

![](../.gitbook/assets/chang-jian-cuo-wu-yong-hu-cheng-xu-cuo-wu.JPG)

### 问题原理：

该实例中由于用户的Python程序存在list index out of range异常，所以导致map任务抛出异常，以至于整个任务失败。

### 解决方案：

用户修改自己程序代码重新提交任务。

## 权限认证问题

### 问题描述：

### 问题原理：

### 解决方案：

## NodeManager进程被杀掉

### 问题描述：

NodeManager突然掉线，并且日志中没有出现错误，知识ResourceManager日志中出现重连NM次数达到上限，如图所示。![](../.gitbook/assets/jin-cheng-tu-ran-bei-sha.jpg)NM日志![](../.gitbook/assets/rm-lian-jie-bu-shang-nm.jpg)RM日志

### 问题原因：

由于NM日志上显示container都正常运行，突然没有了任何日志记录，而RM尝试重连NM失败，说明很有可能是NM进程突然掉了；可能被人为kill -9 了，也有可能是系统把该进程kill了。

### 解决方案：

重新启动nodemanager进程，命令：yarn-daemon.sh start nodemanager。

注意：已经在集群上面运行的sparkstreaming任务，将不会分配到这个新加入的NM的资源。

