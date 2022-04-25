#### （一）cgroup
**简述**
cgroups（control groups）是Linux下控制一个（或一组）进程的资源限制机制，可以对cpu、内存等资源做精细化控制，比如Docker在Linux下就是基于cgroups提供的资源限制机制来实现资源控制的；除此之外，也可以指直接基于cgroups来进行进程资源控制，比如8核的机器上部署了一个web服务和一个计算服务，可以让web服务仅可使用其中6个核，把剩下的两个核留给计算服务。

cgroups实现了一个通用的进程分组框架，不同资源的具体管理工作由其子系统来实现，当需要多个限制策略比如同时针对cpu和内存进行限制，则同时关联多个cgroup子系统。

**cgroups子系统：**
* cpu 子系统，主要限制进程的 cpu 使用率。
* cpuacct 子系统，可以统计 cgroups 中的进程的 cpu 使用报告。
* cpuset 子系统，可以为 cgroups 中的进程分配单独的 cpu 节点或者内存节点。
* memory 子系统，可以限制进程的 memory 使用量。
* blkio 子系统，可以限制进程的块设备 io。
* devices 子系统，可以控制进程能够访问某些设备。
* net_cls 子系统，可以标记 cgroups 中进程的网络数据包，然后可以使用 tc 模块（traffic control）对数据包进行控制。
* freezer 子系统，可以挂起或者恢复 cgroups 中的进程。
* ns 子系统，可以使不同 cgroups 下面的进程使用不同的 namespace。

CGroup 提供了一个 CGroup 虚拟文件系统(VFS)，作为进行分组管理和各子系统设置的用户接口。要使用 CGroup，必须挂载 CGroup 文件系统。这时通过挂载选项指定使用哪个子系统。

Cgroups提供了以下功能：
1.限制进程组可以使用的资源数量（Resource limiting ）。比如：memory子系统可以为进程组设定一个memory使用上限，一旦进程组使用的内存达到限额再申请内存，就会出发OOM（out of memory）。
2.进程组的优先级控制（Prioritization ）。比如：可以使用cpu子系统为某个进程组分配特定cpu share。
3.记录进程组使用的资源数量（Accounting ）。比如：可以使用cpuacct子系统记录某个进程组使用的cpu时间
4.进程组隔离（Isolation）。比如：使用ns子系统可以使不同的进程组使用不同的namespace，以达到隔离的目的，不同的进程组有各自的进程、网络、文件系统挂载空间。
5.进程组控制（Control）。比如：使用freezer子系统可以将进程组挂起和恢复。
![images](https://img-blog.csdnimg.cn/20181102144301402.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RlZmluZV91cw==,size_16,color_FFFFFF,t_70)

`cgroup提供了一种控制Linux资源的机制和方法。（但是不能保证资源的隔离）`

**应用**
1：Docker
2：LXC


#### （二）LXC
LXC是Linux Container的缩写，LinuxContainer容器可以提供轻量级的虚拟化，以便隔离进程和资源，而且不需要提供指令解释机制以及全虚拟化的其他复杂性。它本质上是用来隔离进程的。那么LXC与上面讲的cgroup有何关系？LXC建立在CGroup基础上，LXC内部包含了cgroup，同时还有namespace、chroot等组件。

1）LXC的优势与虚拟化相比，它的优势在于：
a)不需要指令级模拟；
b)不需要即时(Just-in-time)编译；
c)容器可以在CPU核心的本地运行指令，而不需要任何专门的解释机制；
d)避免了准虚拟化和系统调用替换中的复杂性。
总结来说，就是LXC更加轻量级，具有更小的性能开销、更快的相应时间。

我们需要知道，在Linux中的container在内核层面有两个机制保证：资源的隔离性保障(namespace)；资源的控制保障(cgroup)。
说到namespace，Linux上的namespace有6种：pid，uts，ipc，net和user。


Docker、LXC、Cgroup的结构关系：
![81b46468c5fccbf58f81a84c6c367bd7.png](en-resource://database/1022:1)
