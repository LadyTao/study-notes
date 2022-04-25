#### 什么是kafka
分布式消息队列。
通过订阅-发布模式在应用之间实时传递数据流，同时保证可靠性和容错性。

#### 基本概念
* broker: 即kafka进程实例，既可以以单点方式运行，也可以通过多个节点组成集群运行；
* record:  kafka中的每条记录称为一个record，由key、value、timestamp 3个部分组成；
* topic: 消息可以分类，每个类别称作一个topic，一个topic可以理解为一个逻辑上的消息队列；
* partition: 一个topic所包含的数据可以通过"分区"存放到不同的物理机上或者存放到同一物理机的不同目录。引入partition表示topic在物理上的分区，对应文件系统的一个目录（存放分区对应的record和索引文件）。
* producer: 产生消息的一组进程，负责将消息放入到broker；
* consumer: 处理消息的一组进程，负责读取broker中的消息并执行业务处理；
* consuner group: 用来指定同属一个组的consumer，同一组的consumer不能重复消费同一条消息，因为同一个group的consumer分别对接消费不同的partition；


#### 核心API
* Producer API：用于编程实现producer逻辑，发布消息到一个或者多个topic；
* Consumer API：用于编程实现consumer逻辑，从一个或者多个topic中订阅并且处理消息；
* Stream API：将应用程序作为一个流式处理器，从topic中订阅消息、并进行处理，然后再发布到其它topic中；
* Connector API：用于负责对接broker和外部系统之间的数据读写操作，kafka已经提供很多现成的connector实现。可以帮助建立一个可以重用的Producer或者Consumer，比如：通过基于关系型数据库的connector可以在数据表中保存每次变更；基于文件系统的connector，可以实现broker与文件之间的数据传递

#### 关于partition
![3e87246f2ff890981ecc7728e73661c5.png](en-resource://database/606:1)

* 一个topic可以有多个分区(partition)，消息在每个分区内部是有序的，但无法保证全局有序(如果只有一个partition则当然也是全局有序)。partition会为每条消息分配一个唯一ID，称为offset，用来标识分区中的唯一一条数据。
* paitition主要有两个作用：扩容和并发。首先，一个topic可以有多个partition，多个partition可以分布在多个机器上，因而可以保存更多的数据；其次，多个partition可以被多个消费者消费，提高了并发性。
* kafka会保留所有已经发布的消息，无论是否被消费，kafka的性能随着数据量增大而常数据下降，因而保存大量的数据也不会有太大问题。
* consumer处理数据的标识由其自身维护，每个consumer保留需要的offset元数据，记录当前读取的消息在日志中的位置。consumer既可以按顺序读取每一条数据，也可以通过改变offset位置用来跳过或者退到之前的位置消费。
* 每个partition可以配置指定数量的副本，存放在不同的broker上，提高容错性。每个分区的副本包含一个leader，0个或多个follower。leader负责处理当前分区的所有读写请求，follower只是同步leader的数据。当leader所在的broker宕掉后，follower中的一个经过选举成为新的leader。集群中的每个broker都可以看成是一部分partition的leader，又是另一些分区的follower，保证了集群的负载均衡。但是也不难发现在正常情况下，follower并不能对外提供服务，仅仅是数据副本而已，更多的副本并不能解决吞吐问题。
* 一个topic的partition只能分配给同一个消费组的一个consumer。假如说某个topic有3个分区，在同一个消费组下，那它最多被三个consumer同时消费，如果超过3个消费者，其余部分实际上是空闲的，它得不到任何数据。

#### 应用场景
1. 消息队列    
  适用于高吞吐、内建分区、可复制、可容错的消息队列
2. 网站活动追踪
3. 运行数据统计
    用于监控运行数据，汇聚之后进行统计。
4. 日志聚合
    用于替代Flume、Scribe的等日志收集工具，提供高性能、持久性、低延迟的日志收集。
5. 流式处理
    用作流式处理工具，进行数据的聚合、富集、转换等操作。类似与Apache Storm等工具。
6. 事件源模式实现









