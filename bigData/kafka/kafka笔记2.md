#### 常见问题


######  1. 如何获取 topic 主题的列表
>bin/kafka-topics.sh --list --zookeeper localhost:2181
######  2. 生产者和消费者的命令行是什么？
>bin/kafka-console-producer.sh --broker-list node1:9092 --topic my-kafka-topic

> bin/kafka-console-consumer.sh --bootstrap-server node1:9092 --topic my-kafka-topic

######  3. consumer 是推还是拉？
> 在kafka中，消费者是采用pull的方式获取数据，consumer可以根据自身情况决定获取数据的策略。而生产者是采用push的方式写入数据。

######  4. 讲讲 kafka 维护消费状态跟踪的方法
> Topic是分区的，每个分区的数据在同一时间只能被一个消费者消费，也就是说每个分区被消费的消息在日志中的位置就是一个数字: offset。offset记录的就是在某个消费组中每个分区最后消费所停留的位置。kafka消费的at least once和at most once其实就是在消费重启后，是否继续消费offset所在位置的数据。而exactly once消费模式则比较复杂，需要与客户端配合才行。

######  5. 讲一下主从同步
> 

######  6. 为什么需要消息系统，mysql 不能满足需求吗？
> (1)解耦：
> 允许你独立的扩展或修改两边的处理过程，只要确保它们遵守同样的接口约束。

> (2)冗余：
> 消息队列把数据进行持久化直到它们已经被完全处理，通过这一方式规避了数据丢失风险。许多消息队列所采用的”插入-获取-删除”范式中，在把一个消息从队列中删除之前，需要你的处理系统明确的指出该消息已经被处理完毕，从而确保你的数据被安全的保存直到你使用完毕。

> (3)扩展性：
> 因为消息队列解耦了你的处理过程，所以增大消息入队和处理的频率是很容易的，只要另外增加处理过程即可。

> (4)灵活性 & 峰值处理能力：
> 在访问量剧增的情况下，应用仍然需要继续发挥作用，但是这样的突发流量并不常见。如果为以能处理这类峰值访问为标准来投入资源随时待命无疑是巨大的浪费。使用消息队列能够使关键组件顶住突发的访问压力，而不会因为突发的超负荷的请求而完全崩溃。

> (5)可恢复性：
> 系统的一部分组件失效时，不会影响到整个系统。消息队列降低了进程间的耦合度，所以即使一个处理消息的进程挂掉，加入队列中的消息仍然可以在系统恢复后被处理。

> (6)顺序保证：
> 在大多使用场景下，数据处理的顺序都很重要。大部分消息队列本来就是排序的，并且能保证数据会按照特定的顺序来处理。(Kafka 保证一个 Partition 内的消息的有序性)

> (7)缓冲：
> 有助于控制和优化数据流经过系统的速度，解决生产消息和消费消息的处理速度不一致的情况。

> (8)异步通信：
> 很多时候，用户不想也不需要立即处理消息。消息队列提供了异步处理机制，允许用户把一个消息放入队列，但并不立即处理它。想向队列中放入多少消息就放多少，然后在需要的时候再去处理它们。

######  7. 数据传输的事务定义有哪三种？
>（1）最多一次: 消息不会被重复发送，最多被传输一次，但也有可能一次不传输 
>（2）最少一次: 消息不会被漏发送，最少被传输一次，但也有可能被重复传输.
>（3）精确的一次（Exactly once）: 不会漏传输也不会重复传输,每个消息都传输 
######  8. Kafka 判断一个节点是否还活着有那两个条件？

######  9. Kafka 与传统 MQ 消息系统之间有三个关键区别

>(1).Kafka 持久化日志，这些日志可以被重复读取和无限期保留
> (2).Kafka 是一个分布式系统：它以集群的方式运行，可以灵活伸缩，在内部通过 复制数据提升容错能力和高可用性 
> (3).Kafka 支持实时的流式处理
 
######  10. 讲一讲 kafka 的 ack 的三种机制
>ack机制，即producer发送消息的确认机制，会影响到kafka的消息吞吐量和安全可靠性，二者不可兼得，只能平均；ack的取值有三个1、0、-1

>ack=0，producer只发送一次消息，无论consumer是否收到；
>ack=-1，producer发送的消息，只有收到分区内所有副本都成功写入的通知后才认为发动成功；
>ack=1，producer发送的消息只有leader接收成功后才认为消息发送成功，无论leader是否成功将消息同步到follower，所以，ack值为1 也不一定是安全的。
######  11. 消费者故障，出现活锁问题如何解决？

######  12. 如何控制消费的位置
> kafka 使用 seek(TopicPartition, long)指定新的消费位置。用于查找服务器保留的最早和最新的 offset 的特殊的方法也可用（seekToBeginning(Collection) 和 seekToEnd(Collection)）

######  13. kafka 分布式（不是单机）的情况下，如何保证消息的顺序消费?
>Kafka分布式的单位是 partition，同一个 partition 用一个 write ahead log 组织，所以可以保证 FIFO 的顺序。不同 partition 之间不能保证顺序。但是绝大多数用户都可以通过 message key 来定义，因为同一个key的message可以保证只发送到同一个 partition。Kafka 中发送 1 条消息的时候，可以指定(topic, partition, key) 3 个参数。partiton 和 key 是可选的。如果你指定了 partition，那就是所有消息发往同 1个 partition，就是有序的。并且在消费端，Kafka 保证，1 个 partition 只能被1个consumer消费。或者你指定 key（比如 order id），具有同1个key的所有消息，会发往同 1 个 partition。

######  14. kafka 如何减少数据丢失
###### 14.1 producer 生产端是如何保证数据不丢失的
>1 ack配置策略：通过配置ack=0/1/all确保生产者发送的消息到底应该用那个级别的策略。
>2 retries配置：通过区分不同的异常写入情况，对那些可以处理的问题(网络抖动等)进行尝试，对于不可恢复的错误(数据长度太长等)写入DB后续再特殊处理。

###### 14.2 consumer端是如何保证数据不丢失的
> 在consumer消费阶段，对offset的处理，关系到是否丢失数据，是否重复消费数据，因此，我们把处理好offset就可以做到exactly-once && at-least-once(只消费一次)数据。

> 当enable.auto.commit=true时，表示由kafka的consumer端自动提交offset,当你在pull(拉取)30条数据，在处理到第20条时自动提交了offset,但是在处理21条的时候出现了异常，当你再次pull数据时，由于之前是自动提交的offset，所以是从30条之后开始拉取数据，这也就意味着21-30条的数据发生了丢失。

> 当enable.auto.commit=false时，由于上面的情况可知自动提交offset时，如果处理数据失败就会发生数据丢失的情况。那我们设置成手动提交。当设置成false时，由于是手动提交的，可以处理一条提交一条，也可以处理一批，提交一批，由于consumer在消费数据时是按一个batch来的，当pull了30条数据时，如果我们处理一条，提交一个offset，这样会严重影响消费的能力，那就需要我们来按一批来处理，或者设置一个累加器，处理一条加1，如果在处理数据时发生了异常，那就把当前处理失败的offset进行提交(放在finally代码块中)注意一定要确保offset的正确性，当下次再次消费的时候就可以从提交的offset处进行再次消费。

> 如何保证消息只获取一次并且确定被处理呢？这就需要我们在处理消息的时候要添加一个unique key。假如pull 一个batch 100条的消息，在处理到第80条的时候，由于网络延迟、或者crash的原因没有来得及提交offset，被处理的80条数据都添加了unique key, 可以存到到DB中或者redis中(推荐，因为这样更快), 当consumer端会再次poll消费数据时，因为没有提交offset，所以会从0开始消费数据，如果对之前已经消息过的数据没有做unique key的处理，那么会造成重复消息之前的80条数据，但是如果把每条对应的消息都添加了unique key，那就只需要对被处理的消息进行判断，有没有unique key 就可以做到不重复消费数据的问题，这样也同时保证了幂等性。其实kafka的exactly once消费不是由kafka自身机制完成的，需要自己处理。

###### 14.3 broker端是如何保证数据不丢失的
> min.insync.replica
> 在一个topic中，1个分区 有3个副本，在创建时设置了min.insync.replica=2,假如此时在ISR中只有leader副本(1个)存在，在producer端生产数据时,此时的acks=all,这也就意味着在producer向broker端写数据时，必须保证ISR中指定数量的副本(包含leader、fllow副本)全部同步完成才算写成功，这个数量就是由min.insync.replica来控制的，这样producer端向broker端写数据是不成功，因为ISR中只有leader副本，min.insync.replica要求2个副本，此时的producer生产数据失败(异常)，当然consumer端是可以消费数据的，只不过是没有新数据产生而已.这样保证了数据的一致性，但这样会导致高可用性降低了。一般的配置是按： n/2 +1 来配置min.insync.replicas 的数量的，同时也要将unclean.leader.election.enable=false

> unclean.leader.election.enable。假如现在有leader 0 fllow 1 fllow 2 三个副本，存储的数据量分别是10 9 8，此时的broker的配置是：min.insync.replica=2 acks=all,leader的数据更新到了15，在没有同步到fllow 1 fllow 2时挂掉了，此时的ISR队列中是有fllow 1 和fllow 2的，如果unclean.leader.election.enable设置的是true,表示在ISR中的副本是可以竞选leader这样就会造成9-15或8-15之间的数据丢失，所以unclean.leader.election.enable必须设置成成false，这样整个kafka cluster都不读写了，这样就保证了数据的高度一致性

######  15. kafka 如何不消费重复数据？比如扣款，我们不能重复的扣
> 其实还是得结合业务来思考，我这里给几个思路：比如你拿个数据要写库，你先根据主键查一下，如果这数据都有了，你就别插入了，update 一下好吧。比如你是写 Redis，那没问题了，反正每次都是 set，天然幂等性。比如你不是上面两个场景，那做的稍微复杂一点，你需要让生产者发送每条数据 的时候，里面加一个全局唯一的 id，类似订单 id 之类的东西，然后你这里消费 到了之后，先根据这个 id 去比如 Redis 里查一下，之前消费过吗？如果没有消 费过，你就处理，然后这个 id 写 Redis。如果消费过了，那你就别处理了，保证别重复处理相同的消息即可。比如基于数据库的唯一键来保证重复数据不会重复插入多条。因为有唯一键约束了，重复数据插入只会报错，不会导致数据库中出现脏数据。

######  16. Zookeeper 对于 Kafka 的作用是什么？ 
###### 1、Broker注册
> Broker是分布式部署并且相互之间相互独立，但是需要有一个注册系统能够将整个集群中的Broker管理起来，此时就使用到了Zookeeper。在Zookeeper上会有一个专门用来进行Broker服务器列表记录的节点：/brokers/ids。每个Broker在启动时，都会到Zookeeper上进行注册，即到/brokers/ids下创建属于自己的节点，如/brokers/ids/[0...N]。Kafka使用了全局唯一的数字来指代每个Broker服务器，不同的Broker必须使用不同的Broker ID进行注册，创建完节点后，每个Broker就会将自己的IP地址和端口信息记录到该节点中去。其中，Broker创建的节点类型是临时节点，一旦Broker宕机，则对应的临时节点也会被自动删除。

###### 2、Topic注册
> 在Kafka中，同一个Topic的消息会被分成多个分区并将其分布在多个Broker上，这些分区信息及与Broker的对应关系也都是由Zookeeper在维护，由专门的节点来记录，如：/borkers/topics。Kafka中每个Topic都会以/brokers/topics/[topic]的形式被记录，如/brokers/topics/login和/brokers/topics/search等。Broker服务器启动后，会到对应Topic节点（/brokers/topics）上注册自己的Broker ID并写入针对该Topic的分区总数，如/brokers/topics/login/3->2，这个节点表示Broker ID为3的一个Broker服务器，对于"login"这个Topic的消息，提供了2个分区进行消息存储，同样，这个分区节点也是临时节点。

###### 3、生产者负载均衡
> 由于同一个Topic消息会被分区并将其分布在多个Broker上，因此，生产者需要将消息合理地发送到这些分布式的Broker上，那么如何实现生产者的负载均衡，Kafka支持传统的四层负载均衡，也支持Zookeeper方式实现负载均衡。(1) 四层负载均衡，根据生产者的IP地址和端口来为其确定一个相关联的Broker。通常，一个生产者只会对应单个Broker，然后该生产者产生的消息都发往该Broker。这种方式逻辑简单，每个生产者不需要同其他系统建立额外的TCP连接，只需要和Broker维护单个TCP连接即可。但是，其无法做到真正的负载均衡，因为实际系统中的每个生产者产生的消息量及每个Broker的消息存储量都是不一样的，如果有些生产者产生的消息远多于其他生产者的话，那么会导致不同的Broker接收到的消息总数差异巨大，同时，生产者也无法实时感知到Broker的新增和删除。(2) 使用Zookeeper进行负载均衡，由于每个Broker启动时，都会完成Broker注册过程，生产者会通过该节点的变化来动态地感知到Broker服务器列表的变更，这样就可以实现动态的负载均衡机制。

###### 4、消费者负载均衡
> 与生产者类似，Kafka中的消费者同样需要进行负载均衡来实现多个消费者合理地从对应的Broker服务器上接收消息，每个消费者分组包含若干消费者，每条消息都只会发送给分组中的一个消费者，不同的消费者分组消费自己特定的Topic下面的消息，互不干扰。

###### 5、分区 与 消费者 的关系
> 对于每个消费者组 (Consumer Group)，Kafka都会为其分配一个全局唯一的Group ID，Group 内部的所有消费者共享该 ID。订阅的topic下的每个分区只能分配给某个 group 下的一个consumer(当然该分区还可以被分配给其他group)。
同时，Kafka为每个消费者分配一个Consumer ID，通常采用"Hostname:UUID"形式表示。在Kafka中，规定了每个消息分区 只能被同组的一个消费者进行消费，因此，需要在 Zookeeper 上记录 消息分区 与 Consumer 之间的关系，每个消费者一旦确定了对一个消息分区的消费权力，需要将其Consumer ID 写入到 Zookeeper 对应消息分区的临时节点上，例如：/consumers/[group_id]/owners/[topic]/[broker_id-partition_id]其中，[broker_id-partition_id]就是一个 消息分区 的标识，节点内容就是该 消息分区 上 消费者的Consumer ID。

###### 6、消息消费进度Offset 记录
>在消费者对指定消息分区进行消息消费的过程中，需要定时地将分区消息的消费进度Offset记录到Zookeeper上，以便在该消费者进行重启或者其他消费者重新接管该消息分区的消息消费后，能够从之前的进度开始继续进行消息消费。Offset在Zookeeper中由一个专门节点进行记录，其节点路径为:/consumers/[group_id]/offsets/[topic]/[broker_id-partition_id]节点内容就是Offset的值。

###### 7、消费者注册
> 消费者服务器在初始化启动时加入消费者分组的步骤如下:

> 注册到消费者分组。每个消费者服务器启动时，都会到Zookeeper的指定节点下创建一个属于自己的消费者节点，例如/consumers/[group_id]/ids/[consumer_id]，完成节点创建后，消费者就会将自己订阅的Topic信息写入该临时节点。

> 对消费者分组中的消费者的变化注册监听。每个消费者都需要关注所属消费者分组中其他消费者服务器的变化情况，即对/consumers/[group_id]/ids节点注册子节点变化的Watcher监听，一旦发现消费者新增或减少，就触发消费者的负载均衡。对Broker服务器变化注册监听。消费者需要对/broker/ids/[0-N]中的节点进行监听，如果发现Broker服务器列表发生变化，那么就根据具体情况来决定是否需要进行消费者负载均衡。进行消费者负载均衡。为了让同一个Topic下不同分区的消息尽量均衡地被多个消费者消费而进行消费者与消息分区分配的过程，通常，对于一个消费者分组，如果组内的消费者服务器发生变更或Broker服务器发生变更，会发出消费者负载均衡。

######  17. kafka 的高可用机制是什么？
需要先明确几个概念：
>1、LEO（last end offset）：当前replica存的最大的offset的下一个值
>2、HW（high watermark）：小于 HW 值的offset所对应的消息被认为是“已提交”或“已备份”的消息，才对消费者可见。
>3、leader里存着4个数据：leader_LEO、leader_HW、remote_LEO集合、remote_HW集合。
>4、follower里只保存自身的：follower_LEO、follower_HW。

###### 1、HW与LEO的更新
> 假设有一个leader和2个follower。初始时，leader_LEO = 5、leader_HW = 0、所有 follower 的 follower_LEO = 0、follower_HW = 0：

> 1、follower向leader发送fetch请求，来拉取消息，这个fetch请求会带上各自的follower_LEO（先省略leader处理fetch请求的过程）
> 2、leader接收fetch请求后，处理并返回，这个fetch请求的响应会带上 leader_HW
> 3、follower拿到了fetch请求的response，同步消息数据，更新自己的 follower_LEO（假设此时 这2个 follower_HW 分别为3和4）
> 4、接下来计算一下 follower_HW 并更新> 公式：follower_HW = min(leader_HW，follower_LEO)。所以 两个follower此时的 follower_HW都是 min(0，0) = 0
> 5、follower再次发送fetch请求，并带上各自的 follower_LEO
> 6、leader处理fetch请求过程如下：把这2个 follower_LEO 加上自己的 leader_LEO，取最小值作为新的 leader_HW
公式：leader_HW = min(leader_LEO，RemoteIsrLEO)
（RemoteIsrLEO：ISR中的所有follower的follower_LEO）
leader_HW = min(l5，3，4) = 3
> 7、leader_HW = 3 被 fetch请求的response带上，返回
> 8、2个follower收到response后，依然是，同步消息数据，更新 follower_LEO，更新 follower_HW（假设此时 这2个 follower_HW 分别更新至 6和8）
follower_HW1 =  min(3，6) = 3
follower_HW2 =  min(3，8) = 3

![images](https://img-blog.csdnimg.cn/20201104141702441.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhaW1hX2NhaWdvdQ==,size_16,color_FFFFFF,t_70)
实际上，不是所有副本中存的数据都是对用户可见的，在小于等于leader的HW的那部分数据是可见的，在（HW， LEO）范围内的数据，只有在in.insync.replica个副本都同步完成后(ack=all时)，才对外可见。

###### 2、HW的丢数据及数据不一致
在 0.11.0.0 版本之前， Kafka使用的是基于HW的同步机制，但这样有可能出现数据丢失或leader副本和follower副本数据不一致的问题。
* 丢数据
![images](https://img-blog.csdnimg.cn/20201105142926475.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhaW1hX2NhaWdvdQ==,size_16,color_FFFFFF,t_70)
1、此时leader收到fetch请求，没有消息要同步了，返回leader_HW=2，理论上讲follower应该接到response，再更新follower_HW = 2就OK了。但是异常发生，follower挂掉了，当它重启后，会根据当前HW的位置进行截断，此时follower上offset=1的数据就没了。
![images](https://img-blog.csdnimg.cn/20201105143007714.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhaW1hX2NhaWdvdQ==,size_16,color_FFFFFF,t_70)
2、follower截断日志后，理论上讲它应该去追赶leader，先一次fetch，重新把offset=1的消息拿过来，再一次fetch，把HW更新为2。但不幸的是这依然没发生，而是leader宕机了，follower成了新leader，重启旧leader成为了follower。
![images](https://img-blog.csdnimg.cn/20201105143127983.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhaW1hX2NhaWdvdQ==,size_16,color_FFFFFF,t_70)
3、follower_HW不能比leader_HW 大，必须以leader为准，所以还会做一次日志截断，以此将follower_HW调整为 1
![images](https://img-blog.csdnimg.cn/2020110514321619.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhaW1hX2NhaWdvdQ==,size_16,color_FFFFFF,t_70)。
至此，offset=1的数据就彻底丢失了......

* 数据不一致 
场景：leader有 2 条消息，且 HW 和 LEO 都为 2, follower有 1 条消息，且 HW 和 LEO 都为 1
![images](https://img-blog.csdnimg.cn/2020110514552645.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhaW1hX2NhaWdvdQ==,size_16,color_FFFFFF,t_70)
1、leader和follower同时宕机，follower率先复苏，成为新leader，并接收一个消息m并将LEO和HW更新至 2 （假设所有场景中的 min.insync.replicas 参数配置为 1)
![images](https://img-blog.csdnimg.cn/20201105145825702.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhaW1hX2NhaWdvdQ==,size_16,color_FFFFFF,t_70)

2、旧leader重启，成为follower，需要根据 HW 截断日志及发送 FetchRequest 至新leader ，不过此时HW相等，那么就可以不做任何调整了。
![images](https://img-blog.csdnimg.cn/20201105150041483.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2RhaW1hX2NhaWdvdQ==,size_16,color_FFFFFF,t_70)

但是此时leader和follower存的数据并不一样，但是由于HW相同，导致他们彼此认为不需要再做同步。

为了解决这个问题，新版的kafka引入了epoch的概念，这个就暂时mark一下，后面有时间再记录。

###### 3、所以kafka到底怎么保证数据的可靠性？
kafka以topic的形式区分不同的数据来源，每个topic可以有多个partition，每个partition又有1到多个副本。数据分为leader和follower，读写由leader负责，follower只去同步leader的数据，并不对外提供服务。follower要与leader数据一致，在重启或切换leader后，依然能获取到最新数据。
换句话说，副本是kafka数据可靠性的保障。那么，kafka的副本如何同步数据，使其能一定与leader一致？
**副本同步策略**
和zookeeper不同的是，Kafka选择的是全部完成同步，才发送ack。但是又有所区别。这里引入几个概念：
1、AR（Assigned Repllicas）一个partition的所有副本（就是replica，不区分leader或follower）
2、ISR（In-Sync Replicas）能够和 leader 保持同步的 follower + leader本身 组成的集合。
3、OSR（Out-Sync Relipcas）不能和 leader 保持同步的 follower 集合
4、公式：AR = ISR + OSR

Kafka对外声称是完全同步，实际上只保证对ISR集合中的所有副本保证完全同步。而最坏的情况下，ISR中只剩leader自己。

我们可以把ISR理解为一个高级的鱼塘，kafka会挑选符合要求的鱼（副本）进入这个鱼塘，同时也会把劣质的鱼扔出去，这是一个动态调整的过程。那问题来了，是么样的鱼有资格进入ISR这个鱼塘？
以前有两个配置项：
* #默认10000 即 10秒
replica.lag.time.max.ms
* #允许 follower 副本落后 leader 副本的消息数量，超过这个数量后，follower 会被踢出 ISR
replica.lag.max.messages

说白了，就是去衡量leader与follower之间差距的大小。一个是基于时间间隔，一个是基于消息条数。在0.9版本之后移除了replica.lag.max.messages这个配置，因为kafka允许生产者批量写入数据，这样就很容易超过这个限制，造成follower被不断踢出ISR集合。再来说replica.lag.time.max.ms这个参数，它表示follower在过去的replica.lag.time.max.ms时间内，已经至少追赶上leader一次。

当发生leader选举的时候，只有在ISR列表的副本才有资格参与选举。











