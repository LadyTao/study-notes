#### 架构
![4f4f9335ce67a5064c19eef314b51084.jpeg](en-resource://database/889:1)

HDFS是一个主/从（master/slave）体系架构，主要由三部分组成：NameNode,DataNode以及SeondaryNamenode。
* NameNode：管理整个文件系统的元数据，每个文件对应的block信息。
* DataNode：管理用户的文件块，每个数据块可以在多个DN上存在(副本)。
* SecondaryNamenode：监控HDFS的后台辅助程序，定时地获取NN元数据快照。


#### HDFS的特性
1. master/slave架构。NN是集群主节点，为管理类型的角色；DN为从节点，负责存储数据。
2. 分块存储。真实的文件被切分为block的形式存储，每个block默认128M。
3. 名字空间。与传统的层次型文件组织架构类似，提供目录的形式管理文件。
4. NN元数据管理。HDFS中的元数据特指目录结构以及文件的block位置信息。NN维护整个系统的目录结构，以及每个block的块信息（块id，块所在的DN节点）
5. DataNode数据存储。DN定时向NN汇报自己持有的block信息
6. 副本机制。所有block都有副本，每个文件的block大小和副本都是可控的。副本数可以在文件创建后再次修改。
7. 一次写入，多次读取。HDFS文件不支持对文件数据的修改，不适用于数据频繁更新的场景。

#### 2个重要的流程：
2.1：HDFS文件写入
![210f33bd3f026b9cf7d2b297beac0aa1.jpeg](en-resource://database/891:1)
* 1，client向NN发起请求，NN检测文件是否已经存在，client是否有写入的权限。然后申请对该文件的租约，只有持有租约才允许写入，且租约需要定期续约。确认后返回是否可以上传。
* 2，client负责将文件切分为block，并向NN再次发起请求，询问该block可以存在哪个DN。
* 3，NN根据配置文件中指定的备份数量以及机架感知原理进行分配，返回可用的Dn地址，如：A, B，C。
* 4，client请求3台DN中的一个A上传数据。A收到请求后会继续调用B，B调用C，建立起一个PipeLine管道，建好后返回client，告知开始传数据。
* 5，block以packet（默认64kb）的形式被发送到A，A收到完整的packet后转发给B，B转发到C。client端维护着一个ack确认队列，当该packet写入成功后，就移除该packet的记录。
* 6，一个个packet在pipeline中依次传输。每个packet都带有chunk校验信息，如果数据正常，那么chunksum值应该是一致的，在pipeline的反方向，逐个发送ack应答，最终由第一个DN将pipeline ack发送给client。
* 7，当第一个block发送成功后，接着再次请求NN，循环执行上述过程。
* 8，block是穿行写入，不能并行写。


**异常情况**
1：写入过程中某一个DN故障：
关闭pipeline，将未成功发送的packet重新发回到带发送队列(未收到ack确认的所有packet)。为当前Block生成新的ID并发送给NN，使得故障DN恢复后可以删除掉这部分数据。从正常的DN中再选一个作为主节点，获取每个DN当前数据块的大小，从中选一个最小值，让他其DN同步到最小的状态，恢复管道。然后NN发现副本数量不足，发出指令创建新副本。

2：多个DN故障：
集群中有一个配置项dfs.replication.min的副本数（默认为1），只要有一个写入成功就可以。剩余的m个副本由NN负责通知DN复制。
3：ack迟迟未收到导致超时：
当某个DN迟迟未能提交packet的确认消息，pipeline会认为其发生故障，将其从管道中“下线”，从写3个副本变为写2个副本。同时为了避免发生一致性冲突，当前的block块，它的blockID也要进行一次更新；管道中的各个节点再次统一一下数据，看当前写入的packet是第几个，避免重复提交。后续NN发现副本缺失会自动帮助我们在其他正常节点补上这个副本。
4：NN异常：
GG
2.2：HDFS文件读取
![070db64766ed711a8675c0dc74baee10.jpeg](en-resource://database/893:0)
* 1，cliect向NN发起RPC请求，确定block所在节点的列表。
* 2，NN视情况返回部分或全部block列表。对于每个block，NN都会返回含有该block副本的DN地址。这些返回的DN地址中，NN会按照集群拓扑结构找到与client距离最近的DN，并作排序。排序有两个原则：①网络距离，②心跳时间。client会拿到一个block的多个地址，最靠前的优先级最高。
* 3，client优先选取最近的Dn读取数据。如果客户端本身就是一个DN，则直接获取数据（短路读取）。
* 4，建立Scoect Stream，调用read方法，读取这个block的全部数据，若还有其他block没有读完，则继续请求NN。
* 5，当一个block读取完成后会进行checksum验证，如果读取一个DN时出现数据错误，则会去另一个DN上获取。
* 6，并行的方式读取block信息，读的时候没有packet这个东西。
* 7，读取的所有block合成完整文件。


