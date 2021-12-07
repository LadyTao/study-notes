#### 1. 架构
![4f4f9335ce67a5064c19eef314b51084.jpeg](en-resource://database/889:1)

HDFS是一个主/从（master/slave）体系架构，主要由三部分组成：NameNode,DataNode以及SeondaryNamenode。
* NameNode：管理整个文件系统的元数据，每个文件对应的block信息。
* DataNode：管理用户的文件块，每个数据块可以在多个DN上存在(副本)。
* SecondaryNamenode：监控HDFS的后台辅助程序，定时地获取NN元数据快照。

HDFS 1.X时代只能有唯一的NN管理命名空间以及数据块，存在以下缺点：
①NN节点需要在内存在保存整个文件系统的元数据，所以NN的内存大小约定了整个系统的大小；
②所有的读写流程都要有NN的参与协调，所以吞吐量也会受到影响；
③单点故障隐患；
④数据无法有效隔离，耦合性高；

HDFS 2.X提出了联邦机制(Federation)。一个集群中可以出现多个NN（或NameSpace），这些NN相互独立，各自管理自己的命名空间。集群中的DataNode提供数据块共享存储功能。每个DN会向所有的NN注册且上报信息，并执行NN的指令。
![2cf90dbb21a846089c57a7a49115122a.png](en-resource://database/1024:1)
这么做的好处在于：
①NN可以水平扩展；
②为应用程序和用户提供命名空间卷级别的隔离（见DataNode章节），耦合性降低
③解决单点故障


#### 2. HDFS的特性
1. master/slave架构。NN是集群主节点，为管理类型的角色；DN为从节点，负责存储数据。
2. 分块存储。真实的文件被切分为block的形式存储，每个block默认128M。
3. 名字空间。与传统的层次型文件组织架构类似，提供目录的形式管理文件。
4. NN元数据管理。HDFS中的元数据特指目录结构以及文件的block位置信息。NN维护整个系统的目录结构，以及每个block的块信息（块id，块所在的DN节点）
5. DataNode数据存储。DN定时向NN汇报自己持有的block信息
6. 副本机制。所有block都有副本，每个文件的block大小和副本都是可控的。副本数可以在文件创建后再次修改。
7. 一次写入，多次读取。HDFS文件不支持对文件数据的修改，不适用于数据频繁更新的场景。


#### 3. DataNode
Datanode以数据块的形式(Block)的形式保存数据；响应客户端的读写操作；周期性地向NN上报心跳信息、持有的数据块信息、缓存的数据信息；响应NN的指令，如创建、删除或复制数据块等。
**block：**
block唯一标识NN中的数据块，内部有3个字段：①blockID唯一标识这个对象②numBytes是这个数据块的大小③generationStamp是这个数据块的时间戳。

HDFS联邦机制下，引入了两个概念：`块池(BlockPool)`  和 `命名空间卷`：

![a3312174b8f1bed12297ed460764c8a3.png](en-resource://database/1026:1)
**块池：** 一个块池是一个NN对应的命名空间内所有的数据块组成，这个块池内的数据块可以存储在集群中所有的DN上，每个DN又可以存储集群所有块池的所有数据。每个块池之间相互独立，当一个NN故障不会影响其他NN。
**命名空间卷：** 一个NN的命名空间以及它管理的块池称为命名空间卷，当一个NN/NameSpace被删除后，它管理的块池会从DN删除。
************
**DN的逻辑结构：**
为了支持HDFS2引入的联邦机制，DataNode的逻辑结构可以切分为如下图所示的样子：
![4a3d9eba477056d1b8b58f2896d14aa6.png](en-resource://database/1028:1)
从下往上分为3层，分别是数据层，逻辑层和服务层。
* 数据层。负责在本地磁盘存储数据块的部分，包含两个模块：
DataStorage: 管理和组织DN的磁盘存储空间，以及维护存储空间的生命周期。在Federation架构中，一个DN可以存多个块池的数据块，通过BlockPoolSliceStorage类管理单个块池。
FsDataset: 文件系统数据集。抽象所有DN管理数据块的操作：创建块、维护块、校验等。FSVolumeImpl管理单个存储目录的数据块，FSVolumeList维护所有的FSVolumeImpl引用。
* 逻辑层。负责向NN汇报数据块存储状态、发送心跳、扫描损坏的数据块等。有3个模块：
BlockPoolManager: 管理块池的接口类
DataBlockScanner: 周期性扫描数据块并检查数据块的校验和是否正常。
DirectoryScanner: 定时对磁盘数据块扫描，对比内存中的元数据与实际磁盘存储数据块的差异，然后更新内存中的元数据，保持一致。
* 服务层。对外提供服务，HttpServer用于展示DN内部状态；IPCServer响应Client/NN以及其他Dn的RPC请求；DataXceiverServer响应流式接口请求。

***************
**DN升级机制**
这里说的DN，实际是指整个HDFS集群，因为HDFS并不支持单独升级某一个组件，而是整体去升级的。HDFS不支持向下降级，但是允许升级过程中回滚。
![18e6649a73def90d8b0520efadd0e431.png](en-resource://database/1030:1)
升级分3步骤：
1. 升级。升级时会将当前的current目录改为previous.tmp，然后新版本数据重建current目录。接下来建立currnt与previous.tmp中数据块文件与检验和文件之间的硬链接。最后将previous.tmp改为previous，完成升级。
2. 回滚。主要是发生异常，回滚到旧版本。先将current目录改为removed.tmp，然后将previous目录改为current，最后删除removed.tmp目录。
3. 升级提交。提交后无法再回滚。先将previous目录改为finalized.tmp，然后删除finalized.tmp目录。

************
**DataNode磁盘存储结构**
DN并不会保存HDFS文件和目录的元数据，但是需要保存自身的元数据。一个DN可以定义多个存储目录。通过配置项：
```xml
<property>
    <name>dfs.data.dir</name>
    <value>/dfs/data, /dfs/data2</value>
<property>
```
![bfd346feb58e10d38dc3c051aa688235.png](en-resource://database/1032:1)
[1] BP-xxxx-ip-time：这是一个块池目录，保存了一个块池在当前存储目录下的所有数据快。联邦机制中会有多个块池目录，其他的只有一个。这个目录会带有NN的节点的IP地址，最后还会有块池的创建时间。
[2] VERSION：在NN和JN目录中都会有这个文件，保存了文件系统布局版本，集群ID，创建时间，还有存储类型等。下图是NN,DN,JN目录中VERION文件的差异。
![6c4b6991ba871a85f3a715854cc62387.png](en-resource://database/1034:1)
[3] finalized.raw：存放数据，包括数据块文件和对应的校验和文件。rbw（replica being written，正在写入副本）保存了客户端正在写入的数据块；finalized目录包含已经完成写入的数据块。由于数据块可能很多，finalized目录会以特定的目录结构管理(多级目录)。
[4] lazyPersist：HDFS2引入的新特性，支持将临时数据写入内存，然后通过懒持久化方式写入磁盘。
[5] dncp_lock_verification.log：记录DN最后一次确认所有数据快的内容和检验值匹配的时间。
[6] in_use.lock：DN线程持有的锁，防止多个DN线程启动并且并发修改这个目录。

*******
**DataNode数据块副本状态**
DN保存的数据块副本有5种状态：
* FINALIZED：已经完成写操作的副本。
* RBW(replica being written)：客户端新建的副本或正在追加操作的副本，数据正在被写入。部分数据对客户端可见。
* RUR(replica under recovery)：进行恢复时的副本。
* RWR(replica waiting to be recorvery)：DN重启或宕机，所有RBW状态的副本在Db重启后被加载为RWR，并等待恢复。
* TEMPORARY：DN之间复制数据块，会进行数据块平衡时，正在写入副本的状态。其对客户端不可见，且Db重启会直接删除处于TEMPORARY状态的副本。

*******
**DataNode启动**
![eb9c4636f5c5e22ef4bc2b84999b651e.png](en-resource://database/1036:1)
注意到这里并没有对FsDatasetImpl对象、DataBlockScanner对象以及DirectoryScanner对象做任何的初始化操作，因为它们是在DataNode与NN握手时，在initBlockPool（）方法中完成的。

*******
**DataNode停止**
DataNode停止是通过调用shutdown()方法完成。
①将DataNode.shouldRun字段设置为false。
②BPServiceActor/DataXceiver/PacketResponser/DataBlockScanner线程循环条件结束，线程退出。
③关闭DataBlockScanner以及DirectoryScanner。关闭WebServer。
④在DataXceiverServer对象上调用join()方法，等待DataXceiverServer线程退出。
⑤一次关闭IPCServer/BlockPoolManager对象，DataStorage对象以及FSDatasetImpl对象。

*******
**Federation 与 HA**
这里对比一下联邦机制与高可用集群：
①解决的问题不同：HA解决的是单点NameNode故障导致集群不可用的问题，通过引入多个NN，standby状态的只提供元数据的备份能力，真正提供对外服务的只有一个active的NN；联邦机制解决的是NN水平拓展能力，使HDFS不再受NN单点能力的限制。多个NN彼此独立，共同对外服务。
②命名空间：HA中只有一个命名空间。联邦机制每一个NN都会有一个命名空间。一个DN中存放不同命名空间的数据。

#### 4. NameNode
**文件系统目录树**
HDFS命名空间以"/"为根，以树的形式存储。不管是文件还是目录，都被看做一个叫做INode节点。如果是目录对应的类是INodeDirectory，如果是文件对应的类是INodeFile，这两个都是INode的子类。如果是INodeDirectory类，包含一个集合变量children，保存这个目录下的文件及目录的INode对象引用。
命名空间在HDFS运行时在内存中有一份，同时还有一份持久化在磁盘中的镜像(二进制文件)：fsimage。利用它可以在重启时重构恢复整个命名空间。此外，对HDFS的各种操作，都会以日志的形式记录，写到edits文件中。

fsimage保存了文件系统目录树中每一个文件或目录的记录，包含该文件/目录的名称、大小、用户、用户组、修改时间、创建时间等信息。当NN重启时会读取这个文件重构命名空间。但是由于这个文件一般都很大，而且是磁盘上的一个文件，导致它基本不可能及时跟踪NN的变化(实际上默认一个小时才更新一次)，那么两次fsimage之间对NN的操作记录保存在哪里？edits文件保存的正是这份信息。HDFS客户端执行的所有操作都记录在这里，HDFS会定期将fsimage与edits文件合并以保持最新状态。
下图是NN元数据存放的目录结构：
![f936e938d8cac8ea712bfc9ece34b0a7.png](en-resource://database/1038:1)
*`关于transactionId:`* 客户端每发起一次RPC请求对NN的命名空间进行修改，editlog中还有一个新的唯一的reansactionID用于记录这个操作。

* edits_start_trans_id-end_trans_id：edits文件。文件名中记录了其内部transactionId的范围。
* edits_inprogress_start_trans_id：正在写的edits文件，文件名记录了起始的transactionId。
* fsimage_end_trans_id：保存了transaction小于此文件名后缀的所有记录。同时还附属一个.md5的文件，用于判断是否损坏。
* seen_txid：保存上一个检查点以及编辑日志(roll edits)时最新的事务i（transactionId），须注意这个Id并不是最新的，因为edits.inprogress中的记录还没有记载。这个文件作用于NN启动，NN需要确定加载的fsimage和edits文件的事务ID大于等于seen_txid才能启动成功。

*`检查点机制`*
正常来讲一个edits文件是几十个字节大小，但在极端情况下可能会非常大，甚至写满磁盘。NameNode启动除了加载fsimage，还需要加载edits文件，这种情况下导致启动非常缓慢。为了解决这个问题引入了检查点机制。
NN触发检查点机制有两种方法：①超过配置的时间②上一次检查点后，发生事务数量超过一定的次数。
根据集群部署方式不同（是否开启HA），有两种不同的完成方法：
（一）非HA，依赖SecondaryNamenode。
SNN是NN的热备份，二者没有共享的edits文件目录，SNN通过RPC获取最新的seen_id以及最新的fsimage和edits文件，重构SNN的命名空间，并且将最新的命名空间写入到最新的fsimage文件。
（二）HA，依赖Standby Namenode。
Standby的NN与active的NN是完全一致的（通过JournalNode机制实现），Standby的NN只需要从JN获取最新的edits文件并合并即可。

******
**数据块管理**
NN中维护者HDFS两个很重要的关系：

* [ ] HDFS文件系统的目录树以及每个文件对应的数据块列表。
* [ ] 数据块与节点的对应关系，即指定数据块的副本保存在哪些节点上。这个信息是DataNode启动时自己上报给NN的，fsimage中并没有存放这个信息。

几个重要的类：
`Block:`唯一标识NN中的数据块，其包含3个字段①blockId：唯一标识②numBytes：数据块的大小③generationStamp：数据块时间戳。
`BlocksMap:`它管理者NN上数据块的元数据，包括当前数据块属于哪一个文件，以及这些block存在于哪一个DN。非常重要。
Block与Replica的对比：
如果数据是NN中的，我们称其为Block；如果是在DN中的，我们称为Replica。

******
**租约管理**
HDFS文件是一次写入多次读取的，且不支持客户端并行写入。而实现这一机制靠的就是租约(Lease)。租约是NN给与租约持有者在规定时间内拥有写文件权限的合同。
HDFS客户端要写文件需要先从NN中的租约管理器(LeaseManager)申请一个租约，申请成功后客户端就成为租约持有者，拥有了对文件的独占权限，其他客户端无法再打开这个文件进行操作。租约管理其包含了文件与租约，租约与租约持有者之间的对应关系，并且还会有单独的线程定期检查租约是否过期。过期的租约会强制回收，所以租约持有者需氢气更新租约，当完成操作后必须在租约管理器中释放租约。
`软限制与硬限制`
软限制是写文件规定的租约超时时间；超过这个时间其他客户端可以争夺这个文件的所有权。
硬限制用于文件关闭异常时，强制回收该租约并销毁。

******
**High Availability**
高可用机制下，通过JN协调使Standby状态的NN可以及时更新元数据。为了在状态切换时更迅速，需要DN同时向两个NN同时发送心跳以及汇报数据信息。
![fbb0aa4459fd28a988e46147785136e4.png](en-resource://database/1040:1)

HA架构中需要解决脑裂问题：两个NN同时处于Active状态导致数据丢失或误操作。HDFS提供3种隔离机制防止脑裂：
* 共享存储隔离：同一时刻只允许一个NN向JN写入edits数据；
* 客户端隔离：同一时刻只允许一个Namenode响应客户端请求；
* Datanode隔离：同一时刻只允许一个NN向DN下发名字节点指令，如删除、复制等。

******
**NN的启动与停止**
启动过程：
![99be57a4af5bf1a4651aeb2df55851f2.png](en-resource://database/1042:1)

停止流程
(略)

******

#### 5. Hadoop Client

#### 6. Hadoop RPC
