##### 1. 架构
![99542a893cd6583d17e4c02eac1a2046.png](en-resource://database/1207:1)
在kafka2.8版本中，kafka移除了zk的依赖，使用其内部实现的Quorum控制器代替，以后部署kafka不再额外需要zk了。
* 生产者。用于生产消息，生产的消息由主题(topic)进行分类，最终写入broker中。
* 消费者。用于消费使用消息，消费者必须归属于一个消费者组中。
* broker。kafka是一个集群，集群中的每一个节点都是一个broker/一个实例。
* zookeeper。本身不属于kafka，之前的版本中，zk负责协调各个broker，保存消费者的offset信息等。
* topic。kafka的消息做分类，存入不同的主题。kafka是订阅模式的，不同的消费者可以重复消费同一个主题的数据。
* partition。每个主题可以有1-max_node个分区，每个分区只保存部分主题的消息。消息在一个partition内连续，全局不连续(partition大于1时)。分区数多可以提高性能，因为同一个主题下同一个partition，相同消费者组内只允许一个消费者取数据。
* replicas。副本，kafka容错的保障，每个partition都可以配置多个副本，当leader副本挂掉后，正常的follower副本可以成为leader。副本平时不参与写或读，它定期拉leader的数据保持最新。

##### 2. 副本与分区的关系
![9883fa7ec746b1b208d0fa86432fe13c.png](en-resource://database/1209:1)
消息按主题写入kafka中，每个主题下有多个partition分区，为了保证数据的安全，每个分区要配置多个replica副本。副本分为主副本leader和从副本follower，同一个分区的多个副本不能放在同一个broker上。主副本负责接收并写入数据，并接收消费者的请求读取数据。从副本基本啥也不做，只是不断地同步主副本的数据，保持数据一致。但是由于机器故障等原因，从副本可能挂掉，或者状态不正常，此时它是不能再同步数据的。那些有资格同步数据的副本叫做ISR(in-sync-replicas)，当主副本挂掉后，新的主副本将在ISR中选举诞生。

##### 3. 存储原理，为什么kafka速度这么快？
数据写入kafka后，根据主题名与分区名，被写入不同目录的磁盘。每个目录下，数据是按段存储的(segment)。默认每个partition下的segment文件大小最大时1G，超出后将被分裂。每个segment又分为两个文件：.log和.index文件。其中.log保存真实的数据，.log是一个稀疏的索引文件，保存一部分数据(采样一部分)所在的磁盘偏移量，可以加速定位数据。正是通过按主题、分段，以及索引文件的作用，且消费者顺序读取数据，使得kafka吞吐量很高，速度特别快。

##### 4. 如何确保顺序消费
同一个主题下，kafka无法做到全局的数据有序，消费者也无法确保读到的数据有序。但是如果主题只有一个topic，那么读取的数据一定是有序的。

##### 5. kafka事务？exactly once如何保证？
* kafka的事务机制是其精准消费一次语义的前提。所谓kafka的事务，可以实现对多个主题多个分区的原子写入(要么同时成功，要么同时失败)，保证了ACID。kafka的事务机制在底层依赖于幂等生产者，开启事务时会自动开启幂等生产者。
* 为了支持事务，kafka引入了两个新组件：Transaction Coordinator和Transaction Log。Transaction Coordinator与每一个broker集成在一起，是broker的一部分功能。Transaction Log是一个隐藏的topic，用户看不到。它有N的partition，其中N就是broker的数量，每个partition都会对应一个broker上的Transaction Coordinator，且只有这个coordinator才可以对这个partition进行读写。
* Transaction Log这个主题内存储的是事务的最新状态和其相关的元数据，不存具体的源数据。事务的状态分为Ongoing, Prepare Commit 和 Completed。
* 为了支持事务，kafka还对数据的格式做了拓展，在原有的基础上新增两个字段，表示这条数据是否处于事务中，这条数据是否是控制消息(commit 或 abort)。
* transaction.id起到是一个配置参数，每一个事务要使用不同且唯一的id，通过它可以找到transaction log对应的分区。当客户端携带这个id连接broker时，会关闭所有相同transaction.id且处于pending状态的事务，同时增加epoach，屏蔽僵尸生产者。这个操作对于每个producer session只执行一次。
* kafka提交事务是两阶段提交模式，第一阶段coordinator更新事务状态为prepare_commit，并持久化到transaction log中；第二阶段写transaction marker标记到源数据的topic中(commit / abort)，表示事务是否完成。
* 对于消费者而言，可以通过trad_commited选项，只读取成功提交的数据。
* 开启事务后，性能可能下降3%左右。


##### 6. 是否存在丢失数据，如何确保数据一致？读写分离是否支持？
* 从生产者角度讲：数据被写入leader副本，然后数据再同步到各从副本才算真正意义上写入成功，但是有可能主副本写入成功，从副本还没来得及同步主副本就挂了，就有可能丢失数据。kafka考虑到这一点，就提供了一个配置，是写入主副本/全部副本/大多数副本成功，才收到确认消息，由用户自己决定使用哪种策略，所以kafka是有机制保证写入数据不丢失的。当数据写入磁盘后，由于副本的存在数据就更不会丢失了

* 从消费者讲：消费者通过偏移offset去获取数据，如果某次消费到数据，并处理完，但是offset的移动消息却没有上传通知给kafka怎么办，这样就有可能丢失或重复消费数据呀？解决途径：①比如说消费者可以手动提交offset消息，只有数据写入后自己手动提交，这样可以做到不丢数据，但是可能会重复消费。②通过事务，确保exactly-once消费，参考第五个问题的答案。

kafka的数据一致性是通过主读主写机制实现的，也就天然不支持读写分离。主读主写可以避免网络等故障下导致的数据不一致问题，也避免了网络造成的延迟问题。

