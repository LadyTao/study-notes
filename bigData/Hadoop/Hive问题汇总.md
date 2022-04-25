[TOC]
#### 1. 小文件过多
hive表数据有多种写入的方式，常见的insert overwrite、spark写入、flink实时写入都会造成小文件过多问题。过多的小文件占用NN的内存，影响HDFS性能；对hive本身来讲，在执行MR任务时，每一个小文件都被当作一个块，会启用一个MAP任务，而MAP任务的启动初始化非常耗时，资源被极大浪费。

解决方法：
1）使用hive自带的concatenate命令，自动合并小文件：
```sql
#非分区表
alter table A concatenate;
# 分区表
alter table B partition(day=20210101) concatenate;
```
**注意：** 这个命令只能用于ORC或RCFILE类型的文件；单个文件最大到256M(与配置项相关)
2）调整Map或Reduce的数量。有多少reduce就会有多少个文件；
3）使用hadoop archive归档小文件。
```shell
set hive.archive.enable=true;
set hive.archive.har.parentdir.settable=true;
set har.partfile.size=1099511627776;

# 归档
alter table A archive partition(dt='2021-01-02', hr='12');

# 解除归档
alter table A unarchive partition(dt='2021-01-02', hr='12');
```
**注意：** 归档后的分区数据无法被overwrite，必须解除归档后才可以。


#### 2. 数据倾斜
如果数据量很小（几百万）你是无法察觉这个问题的，只有当数据量达到一定程度后，节点发生内存溢出或无法正常完成计算时你才会感知到这个问题的严重。

这里说的是Hive解决数据倾斜(MR)，实际上对于spark等其他引擎来讲，手段是相似的。

**什么是数据倾斜，数据倾斜何时发生？**
在map-reduce任务中，reduce阶段是最容易发生数据倾斜的地方，因为map端的数据经过shuffle发送到reduce端时，默认按照key进行hash，如果相同的key数据量过多，hash结果就是大量相同key的数据进入同一个reduce任务，导致其压力过大，计算缓慢。

还有一个情况是在map阶段，map端读取hdfs数据时会对文件进行切分，默认128M一个文件块。如果hdfs中的数据格式不支持分割操作(GZIP)，map无法切分，就需要整个文件都读进来，不仅慢，甚至直接OOM。

所以总结如下：发生数据倾斜原因有两个：一个是任务需要处理大量相同key的数据；而是读取不可分割的大文件块。

**解决方案**
① 空值引发数据倾斜。
> join时两个表中的key有大量的null数据，直接进行join，在shuffle阶段会导致所有的null都被分配到一个reduce处理，导致倾斜。有两种方式解决：不让null值参与计算；给null随机赋值。
> ```sql
> -- 方式1
> SELECT *
> FROM log a
>  JOIN users b
>  ON a.user_id IS NOT NULL
>   AND a.user_id = b.user_id
> UNION ALL
> SELECT *
> FROM log a
> WHERE a.user_id IS NULL;
> 
> -- 方式2
> SELECT *
> FROM log a
>  LEFT JOIN users b ON CASE 
>    WHEN a.user_id IS NULL THEN concat('hive_', rand())
>    ELSE a.user_id
>   END = b.user_id;
> ```

② 不同数据类型引发数据倾斜。
> 进行关联的两个表，如果key的数据类型不一致会导致数据倾斜。需要将数据类型统一
> ```sql
> SELECT *
> FROM users a
>  LEFT JOIN logs b ON a.usr_id = CAST(b.user_id AS string);
>```

③ 不可拆分的大文件引发倾斜。
> 如果使用GZIP导致数据倾斜，是没有解决方案的。只能在建表时注意不要使用不支持split的压缩类型。

④ 数据膨胀引发倾斜。
> 在多维聚合时，分过分组聚合的字段太多，会导致数据倾斜。比如说使用rollup/grouping > sets/cube等。首先可以通过改造语句解决：
> ```sql
> --原语句： select a，b，c，count（1）from log group by a，b，c with rollup;
> -- 改造语句：
> SELECT a, b, c, COUNT(1)
> FROM log
> GROUP BY a, b, c;
> 
> SELECT a, b, NULL, COUNT(1)
> FROM log
> GROUP BY a, b;
> 
> SELECT a, NULL, NULL, COUNT(1)
> FROM log
> GROUP BY a;
> 
> SELECT NULL, NULL, NULL, COUNT(1)
> FROM log;
> ```
> 但是如果维度更多一点，几十个，几百个，就不能直接这么改造语句了。还有一个方法是通过配置参数：`hive.new.job.grouping.set.cardinality`自动控制作业拆解，其表示当使用cubes/rollups等多维聚合操作时，如果最后拆解出来的键组合数量大于该值，会启用新的任务处理超出的那部分组合。可以适当调小该值解决倾斜。

⑤ 表连接引发倾斜。
> 这里讲的是大表join小表时的优化策略，小表通过广播的方式发送到各个map节点，在map阶段就完成join，避免shuffle。目前hive已经自动开启该功能，当小表大小超过25M时会自动使用这个策略。如果想关闭这个功能，或自己调节小表达到多大进行这个流程，可以使用以下参数：
> hive.auto.convert.join=true;
> hive.mapjoin.smalltable.filesize=2500000;

⑥ 数据量确实无法减少引发的倾斜。
> 有些情况下，我们确实无法再减少数据大小了，比如使用collect_list函数等：
> ```sql
> select s_age,collect_list(s_score) list_score
> from student
> group by s_age
> ```
> 在这个sql中，s_age如果存在数据倾斜，数据量达到一定程度时，会导致reduce阶段的内存溢出。可以通过参数`mapreduce.reduce.memory.mb`增大reduce阶段的内存。

⑦ groupBy引起的倾斜。
> 当进行group by操作时，相同key的数据大量进入同一个reduce造成的倾斜。
> * 使用参数：hive.groupby.skewindata=true; 查询计划将被划分为两个MR任务，第一个MR中，map端的结果随机发送到reduce端，第二个MR再前面一个的基础上继续进行聚合。第一个阶段随机由map发送reduce的操作可以降低数据的不均衡。但是有局限的是，优化的前提是通过第一个MR可以显著的降低数据量，否则就没有意义了。这也是上面⑥中的例子不能使用这个参数的原因，因为collect_list就是在记录全量的记录，不管你的MR被拆分成几个，最后的结果还是一样的。
> * 开启combiner，map端做预聚合：set hive.map.aggr=true；对map端的内存需求较高。

#### 3. 其他
①：节点内存很大，但还是报OOM异常：
> mapreduce.map.memory.mb（调节map端内存大小）
> mapreduce.reduce.memory.mb（调节reduce端内存大小）
