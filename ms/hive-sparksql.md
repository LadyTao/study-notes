#### Hive
##### 1. HiveSQL执行流程是什么，有哪些优化操作？
我们写的sql提交到集群上不能直接被识别执行，需要经过解析、优化等处理。
1）语法解析。将SQL转为抽象语法树(AST)，进行语法校验。
2）语义解析。遍历AST树，生成QueryBlock。
3）生成逻辑执行计划。将对应的操作翻译为Operator Tree。
4）优化逻辑执行计划。对各个Operator进行变化，减少shuffle数量。
5）生成物理执行计划。将逻辑执行计划翻译为MR任务，生成task Tree。
6）优化物理执行计划，提交执行。

优化操作包括：列裁剪，分区裁剪，谓词下推

##### 2. 数据格式有哪些，哪些支持压缩，各有哪些特点
Hive支持的数据存储格式包括TextFile,SequenceFile,ORCFile,ParquetFile。目前我们一般采用ORC和parquet这两种格式。

* text不能压缩，磁盘开销大，数据解析开销大，但是加载速度快（不怎么需要解析）；

* sequenceFile是个二进制文件，不能直接查看，存储空间消耗非常大。数据可压缩，可分割，可切片。

* orc是rc文件的改进型，是行存与列存的结合体，数据压缩效率最高，查询速度快，可压缩和分割，内部有简单的索引。但是它支持的引擎较少，目前只有MR和Pig。

* parquet是一种列存的格式，压缩性能好，反序列化时间短，对嵌套的数据支持非常好，对spark有良好的优化。

hive支持的压缩格式有多种，常见的有Gzip, Bzip, Lzo, Snappy。最近几年比较火的ZSTD（需安装依赖，依赖系统插件）压缩格式。除了Gzip外，其他均支持文件分割。snappy和zstd压缩速度较高，压缩率zstd和bzip较高。

##### 3. 有哪些join，各有哪些特点
1）Common Join。
> 常见的且效率最低的一种join。分为任务执行阶段分为：Map、Shuffle和Reduce三个阶段。Map从数据源读取表文件，从join on 的条件中拿到key对应的值作为键，值为join后需要的列的值。Shuffle阶段从Map端获取数据，按照key的hash值发送到不同的reduce，确保相同key的值在同一个reduce任务中。Reduce阶段按照key的值完成join的逻辑。

2）Map Join。
> Map join顾名思义就是丢掉Reduce，直接在map阶段完成join操作。但不是所有任务都可以这么做，一般使用场景都是大表和小表join。其原理是小表变成hashtable被广播到所有map端（大表被切分为多个分片，启动多个map任务），用大表的每一行数据去匹配小表数据。目前，新版hive已经默认开启这个功能`hive.auto.convert.jopin=true`。小表默认大小超过25M就会走map join流程。

3）Sort Merge Join。
> 不清楚。

4）Skew Join。
> join数据倾斜时考虑的解决方法。一般来讲每个节点的reduce默认处理1G大小的数据，开启参数后可以控制倾斜的阈值，超过这个值，新的值就会发送给还没达到阈值的reduce。
> set hive.optimize.skewjoin=true;
> set hvie.skewjoin.key=100000;

5）Left Semi Join。
> 由于hive中没有in() 或 exist xxx这样的语法，类似的语句转化成left semi join实现。

##### 4. 写入小文件过多怎么解决
1）使用hive自带的concatenate命令，自动合并小文件：
```sql
#非分区表
alter table A concatenate;
# 分区表
alter table B partition(day=20210101) concatenate;
```
**注意：** 这个命令只能用于ORC或RCFILE类型的文件；单个文件最大到256M(与配置项相关)
2）调整Map或Reduce的数量。有多少reduce就会有多少个文件，对于map任务来讲由map数量决定文件个数；
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

##### 5. 数据倾斜怎么解决
如果数据量很小（几百万）你是无法察觉这个问题的，只有当数据量达到一定程度后，节点发生内存溢出或无法正常完成计算时你才会感知到这个问题的严重。

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
> 如果是大表join大表怎么办？可以考虑分治+盐的思路。即数据倾斜时发生在部分key上的，对于发散较小的key单独正常处理，数据量大的可以在key上拼接其hash取模，目的就是将原本的一个reduce处理的量转到多个reduce上执行，最后把两部分结果union all。

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

##### 6. order by ,sort by ,distribute by ,cluster by分别什么作用

* sort by 一个reducer输出有序；
* order by 全局有序；
* distribute by常与sort by配合使用，distribute用于将对应的key按照不同的值分到不同的reducer处理。
* cluster by 就是distribute by和sort by的组合。但是其排序只能倒序，无法自己指定。

##### 7. mapreduce任务执行流程
整体的流程是：输入文件split分片 -> map阶段 -> combiner阶段（可选）-> shuffle阶段 -> reduce阶段

![4b96b33b7b44a8392f84d4e4d152ea17.png](en-resource://database/1215:1)

* 输入分片阶段：在计算之前，mapreduce会根据输入的文件计算分片的个数（文件需是可切分的），每个分片启动一个map任务，默认是64m一个分片。如果文件比较小，默认也会启动一个map任务。所以如果有太多的小文件，会启动非常多的map任务，带来坏的影响。
* map阶段：用户自定义的业务逻辑。
* combiner阶段：这是一个可选的阶段，不是所有的场景都可以使用的。combiner可以看作是一个执行在map端的reduce，它是map端任务执行完的后续操作，将数据量做尽量的缩减，用于节省网络带宽的消耗。
* shuffle阶段：mapreduce优化重灾区。shuffle的输入就是map的计算结果(文件)，但是我们知道处理大量的数据时，map不可能将其输出结果放在内存中，只能放在磁盘才能容纳。所以shuffle也是从磁盘读数据的。实际上map端写磁盘这个流程比较复杂，它还需要按照key对数据进行排序，且需要根据reduce的数量对数据进行分区，内存开销比较大。在写磁盘时借助于**环形缓冲区**，缓冲区默认是100M。一旦其容量达到80%，就会启动spill溢写流程，将数据刷入磁盘，那么一个map的输出就会出现多个溢写文件。等到map的全部数据都溢写完成，map端还要将同一个map任务的文件合并成一个。shuffle将获取的数据按照需要分发到不同的reduce处理任务中。

* reduce阶段：在map完成后才启动执行，客户端的具体逻辑，最后输出一个文件，或另一个map的输入。

#### Spark
