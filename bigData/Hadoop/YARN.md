#### 1. 与Hadoop1.0的对比
主要从两个方面讲一下：
* 体系架构。Hadoop1.0采用jobTracker完成资源管理和作业调度两大任务，当集群规模大任务多时，由于jobTracker只能是单节点的，所以会成为单点故障源，严重制约了拓展性与可靠性。而在YARN中，任务调度与资源管理是分离的。RM负责资源的分配管理，具体的任务是由启动在NM上AM去监控管理的，解决了单点的限制。
* 运算框架。Hadoop1.0对计算模式支持比较单一，只有MR。而在YARN中开放出统一的接口，只要实现接口，都可以运行在YARN上。丰富了生态圈，使得spark/storm/flink/Tez等都可以支持。

#### 2. 体系架构
YARN作为资源管理、任务调度框架，主要包含3大模块：ResourceManager(RM)、NodeManager(NM)、ApplicationMaster(AM)。其中，RM负责全局的资源监控、分配和管理；AM常驻NodeManger中，每提交一个应用便产生一个实例，负责程序的调度、协调、监控，它会与RM协商沟通获取资源，同时与NM通信来执行和监控task；NM负责每一个节点的维护。
![images](https://github.com/LadyTao/study-notes/blob/main/picture/yarn-structure.png)

`RM`  全局的资源管理系统。NM通过心跳定时向RM汇报本机的资源使用情况（CPU和内存）。当有应用程序提交任务请求时，RM会启动AM，按照一定的策略分配资源给应用。
RM会通过其内部的Scheduler组件分配资源，Scheduler一旦完成分配资源后，就不再负责job的监控、追踪、状态反馈以及失败重启等工作。

`NM` YARN集群每一个节点都运行一个NM实例，充当这台机器的代理，负责本地程序的运行，并对资源进行管理和监控。定时向RM汇报本节点的资源使用情况以及Container运行状态。NM有一个组件NodeHealthCheckServer，可以监控自身状况，当其处于健康、亚健康等状态时，会影响RM是否继续为其分配任务。

`AM`  与RM协商资源，并启动Task运行，监控task状态，失败重启等操作。

#### 3. 任务提交
![images](https://github.com/LadyTao/study-notes/blob/main/picture/yarn.png)

（1）作业提交。client向集群提交作业，RM会生成一个作业id，返回给客户端这个id以及资源的提交路径，client接收到这些信息后将jar包，切片信息和配置文件提交到执行的资源路径(NM)。然后向RM申请运行application master。


（2）作业初始化。RM接收请求，将该job添加到schduler调度器中。调度器根据具体的策略找到具体的NM，启动第一个container，下载client提交的资源到本地，并运行application master。


（3）任务分配。如果作业很小，AM自己就处理了，如果不够继续向RM申请多个container。
（4）任务运行。当RM通过调度器分配好container以后，AM通过联系NM启动container，运行前需要下载运行所需的资源，配置/jar/缓存等。
（5）进度和状态更新。AM将监控任务的运行状态，客户端每秒向AM请求进度更新做展示。
（6）作业完成。作业执行完后，AM和其对应的container会清理工作状态，并销毁？作业的执行信息会保留一段时间。


#### 4. 资源调度器
YARN中有3中资源调度器：
* FIFO 先进先出。最简单的调度方式，先来的任务先执行。只有一个队列，大任务会影响其他任务的执行。
* Capacity Schduler。Hadoop2的默认方式，支持分配多个队列，每个队列可以嵌套子队列。每个队列都有严格的ACL控制，确保一个用户只能提交到属于自己的队列上，且无法修改、查看其他用户提交的任务。运行时可以动态的添加或修改队列(但无法删除)。但是一个空闲的队列资源是无法被其他的队列使用的，可能造成资源的浪费。每个队列内部可以采用FIFO/DRF/Fair等方式调度。
* Fair Capacity。以资源池的形式来组织作业。它也可以配置多个队列，多个队列共享资源池。可以给队列配置最小份额资源和最大份资源，当队列没有任务时，只需要保留最小份额的资源即可，以满足有小任务进来而不受影响，剩余的资源可以被其他队列占有使用；当该队列有大任务而它的资源被其他队列占有时，不会立即回收到资源，需要等待一段时间后才能强制kill掉那些container。每个队列内部可以采用FIFO/DRF/Fair等方式调度。

`关于三种调度方式：`
* fifo：这个没啥好说的，来了就处理，先来先得资源先启动；
* fair：公平调度方式。一般只能针对单一的资源（cpu或memory）做公平化处理。说到fair通常是指min-max算法实现。它有加权和不加权两种：
>`不加权`：假设4个用户，现在分别请求资源：2, 2,6, 4, 5，现有资源为10，根据min-max算法，第一轮用总的剩余资源/任务数，即每个任务都得到10/4=2.5的资源，发现第一个用户已经满足需求且多出0.5的资源，任务1启动；接下来这剩下的0.5再分配给其余的3个任务，0.5/3=0.16。这时，任务2得到的资源是2.5+0.16=2.66，也满足了他的要求，任务2启动，且剩余了0.06；接下来将这份资源平分给任务3和任务4，即0.06/2=0.03。此时，他们都得到2.5+0.16+0.03=2.7。他们都没有得到最够的资源，需要等待其他的任务释放资源。
> `加权`：有4个任务，分别需要4,2,10,4的资源，资源总量是16，权重分别为5,8,1,2。这样可以将资源划分为16份，第一轮按照权重分配16的资源总量，4个任务每个分到5,8,1,2的资源，发现任务1和任务2已经满足需求，正常启动，同时剩下资源：（5-4）+ （8-2）= 7。剩下的7份资源按照任务3和任务4的权重，分别得到 7 * 1/3 = 2.3 和 7 * 2/3 = 4.67。此时，任务3得到资源2.3+1=3.3；任务4得到资源2+4.67=6.67。最后任务4启动，且剩下2.67的资源；最后一轮，将剩下的字眼都给任务3，它最终得到3.3+2.67=5.97，最终需要等待。

* DRF：上面说的fair方式只能针对翻译的资源做公平算法，无法处理多种资源。DRF就是处理这种情况的。DRF的实现相对比较复杂，是一个数学上的最优解问题。
考虑一个有9个cpu和18GB的系统，有两个用户：用户A的每个任务都请求（1CPU，4GB）资源；用户B的每个任务都请求（3CPU，1GB）资源。
![images](https://github.com/LadyTao/study-notes/blob/main/picture/drf.png)
我们会发现在资源充足的情况下，实际分配给每个任务的资源比它申请的额度偏多一点。而当资源不够分配的情况下，就会与其实际的要求有一定的**差额**。在同一个队列中，job的差额越大，越先获得资源执行。那么就会出现任务越大，申请资源越多的任务会越容易执行，导致其他任务可能长时间等待，为了解决这个问题，可以通过**带权重的DRF**算法，它可以给每个job分配不同的权重，这样可以人为干预任务优先级。
我们可以有多种方式控制优先级：
①静态调整。不同队列配置不同优先级
```xml
<property>
    <name>yarn.scheduler.capacity.root.hncscwc.default-application-priority</name>
    <value>80</value>
</property>
```
②静态调整。Hive语句运行时配置优先级
> -- LOW VERY_LOW NORMAL（默认） HIGH VERY_HIGH
> SET mapreduce.job.priority=HIGH;
> 亲测无效

③动态调整(任务已提交)。修改优先级
> yarn application -appId application_1478676388082_963529 -updatePriority 200
> 亲测无效（据说只能用于FIFO）

④动态调整(任务已提交)。修改优先级(更改到优先级更高的队列)
> --movetoqueue 已过期，建议使用changeQuene
> yarn application -movetoqueue application_1639385734502_0008 -queue default 

所以对于公平调度模式，该如何更改其优先级呢？
目前没有找到在任务启动时或运行时的修改优先级的方法，想到的就是多个队列时，每个队列配置不同的优先级，我们可以通过将一些任务移动到优先级高的队列中实现这个目标。对于公平调度，可以修改`fair-scheduler.xml`文件：
```xml
<allocations>
  <queue name="sls_queue_1">
    <minResources>1024 mb, 1 vcores</minResources>
    <schedulingMode>fair</schedulingMode>
    <weight>0.25</weight>
  </queue>
  <queue name="sls_queue_2">
    <minResources>1024 mb, 1 vcores</minResources>
    <schedulingMode>fair</schedulingMode>
    <weight>0.25</weight>
  </queue>
  <queue name="sls_queue_3">
    <minResources>1024 mb, 1 vcores</minResources>
    <weight>0.5</weight>
    <schedulingMode>fair</schedulingMode>
  </queue>
</allocations>
```
