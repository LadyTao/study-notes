#### 基础架构
构建在Hadoop之上的数据仓库，将结构化的文件映射成表，并提供类SQL查询功能。用于查询的SQL将被转化为MR作业，提交到Yarn上运行。

**特点：**
> 1. 简单易上手；
> 2. 灵活性高，可自定义UDF和存储格式；
> 3. 统一的元数据管理，可以与presto/impala/spark共享数据；
> 4. 执行延迟高，不适合实时查询与点查；
> 5. 可以处理超大数据集；

![image]()

**command-line & Thrift/JDBC**
这是两种操作Hive的方式。command line 需要在部署hive的节点上通过命令行运行任务；而thrift/jdbc通过thrift协议按照标准的JDBC远程操作。

**Metastore**
表名，表结构，字段，数据类型，分隔符等都统一称为元数据。默认使用内置的derby数据库。但是derby同一时刻只允许一个实例连接，所以一般都采用mysql管理保存元数据。
在hive中创建一张表，然后再presto/impala/spark中是可以直接使用的，它会从metastore中获取元数据信息；同理，在presto/impla/spark中创建的表，hive也可以直接使用。

**HiveSQL执行流程**

******
> 1. 语法解析：Antlr将SQL转化为抽象语法树AST Tree;
> 2. 语义解析：遍历AST Tree，抽象出查询的基本组成单元QueryBlock；
> 3. 生成逻辑执行计划：遍历QueryBlock，翻译为执行操作树OperatorTree;
> 4. 优化逻辑执行计划：逻辑层优化器进行OperatorTree变换，合并不必要的ReduceSinkOperator，减少shuffle数据量；
> 5. 生成物理执行计划：遍历OperatorTree，翻译为MR任务；
> 6. 优化物理执行计划：物理层优化器进行MR任务的变换，生成最终的执行计划。

**数据类型**
******

|数据大类|实际支持类型|
|--|--|
| integer | tinyint | 
| integer | smallint | 
| integer | int | 
| integer | bigint | 
| boolean | boolean | 
| float | float | 
| float | double | 
| fixed point number | decimal | 
| string | varchar | 
| string | string | 
| string | char | 
| time | timestamp | 
| time | timestamp with local zone | 
| time | date | 
| binary | binary | 
| struct | STRUCT('xxx', 12, false) | 
| MAP | map('a',1,'b',2) | 
| ARRAY | array('a','b','c') | 

**内容分隔符**
*****
hive的数据实际上是HDFS上的文件，要被转换为表，那么就需要一定的分隔符，区分一行数据，一行数据中不同的字段。可以在建表时指定具体的分隔符。

| 分隔符(默认) | 描述 |
|--|--|
| \n | 默认使用换行符区分一条完整数据 |
| ^A(Ctrl A) | 分隔字段，区分每行中具体的列，同八进制数据`\001` |
| ^B | 用于分隔ARRAY或STRUCT中的元素，或者MAP中键值对的分隔，同八进制数据`\002` |
| ^C | 用于在MAP中键和值的分隔，同八进制数据`\003` |

使用示例：

```sql
CREATE TABLE page_view(viewTime INT, userid BIGINT)
 ROW FORMAT DELIMITED
   FIELDS TERMINATED BY '\001'
   COLLECTION ITEMS TERMINATED BY '\002'
   MAP KEYS TERMINATED BY '\003'
 STORED AS SEQUENCEFILE;
```
**存储格式**
*****
注意与压缩格式区分开。

| 格式 | 说明 |
|--|--|
| TextFile | 纯文本，默认的方式。不进行压缩，磁盘开销大，解析开销也大 |
| SequenceFile | 一种Hadoop提供的二进制文件，数据以<key, value>的形式存在 |
| RCFile | 表被分为几个行组，每个行组内的数据按列存储，每一列的数据分开存储 |
| ORC File | RC File的升级版 |
| Avro File | 支持大批量数据交换应用，支持二进制序列化方式，快速处理大量数据 |
| Parquet | 列式存储格式，面向分析。压缩算法高效，特殊的编码方式，IO效率高 |

#### DDL操作
**建表**
```sql
-- 普通分区表
  CREATE  [ EXTERNAL ] TABLE emp (
    empno INT,
    ename STRING,
    job STRING,
    mgr INT,
    hiredate TIMESTAMP,
    sal DECIMAL(7,2),
    comm DECIMAL(7,2),
    deptno INT)    
    PARTITIONED BY (deptno INT)   -- 按照部门编号进行分区
    ROW FORMAT DELIMITED FIELDS TERMINATED BY "\t"
    LOCATION '/hive/emp_partition';;
    
    
--分桶表
  CREATE EXTERNAL TABLE emp_bucket(
    empno INT,
    ename STRING,
    job STRING,
    mgr INT,
    hiredate TIMESTAMP,
    sal DECIMAL(7,2),
    comm DECIMAL(7,2),
    deptno INT)
    CLUSTERED BY(empno) SORTED BY(empno ASC) INTO 4 BUCKETS  --按照员工编号散列到四个 bucket 中
    ROW FORMAT DELIMITED FIELDS TERMINATED BY "\t"
    LOCATION '/hive/emp_bucket';


--倾斜表（如果一个或多个列的值中，有某些特定的几个值大量出现，可以考虑，场景不是特别多）
  CREATE EXTERNAL TABLE emp_skewed(
    empno INT,
    ename STRING,
    job STRING,
    mgr INT,
    hiredate TIMESTAMP,
    sal DECIMAL(7,2),
    comm DECIMAL(7,2)
    )
    SKEWED BY (empno) ON (66,88,100)  --指定 empno 的倾斜值 66,88,100。这几个值的数据会单独形成文件，查询时就不需要扫描全部文件了
    ROW FORMAT DELIMITED FIELDS TERMINATED BY "\t"
    LOCATION '/hive/emp_skewed';


-- 临时表（仅当前会话有效，会话结束自动删除。不支持分区，不支持索引）
  CREATE TEMPORARY TABLE emp_temp(
    empno INT,
    ename STRING,
    job STRING,
    mgr INT,
    hiredate TIMESTAMP,
    sal DECIMAL(7,2),
    comm DECIMAL(7,2)
    )
    ROW FORMAT DELIMITED FIELDS TERMINATED BY "\t";
    
-- 加载文件数据进表 (local表示文件是否在物理机本地或是hdfs上)
load data [local] inpath "/usr/file/emp.txt" into table emp;
```

**修改表**
```sql
-- 重命名表名
ALTER TABLE emp_temp RENAME TO new_emp; --把 emp_temp 表重命名为 new_emp

-- 修改列

-- 修改字段名和类型
ALTER TABLE emp_temp CHANGE empno empno_new INT;

-- 修改字段 sal 的名称 并将其放置到 empno 字段后
ALTER TABLE emp_temp CHANGE sal sal_new decimal(7,2)  AFTER ename;

-- 为字段增加注释
ALTER TABLE emp_temp CHANGE mgr mgr_new INT COMMENT 'this is column mgr';

-- 新增列
ALTER TABLE emp_temp ADD COLUMNS (address STRING COMMENT 'home address');

------------------注意--------------------
-- 1. 转换数据类型只能向"上"转型，比如varchar可以转为string，但是string不要转为varchar，数据可能被截断。如果是浮点型，精度可能丢失;
-- 2. 不同类型的存储格式对数据类型的转换是不一样的。比如说将string转为int，将double转为int，在textfile中起码不会报错，但是在parquet中会直接抛异常；
--3. hive不能直接删除列，如需删列需要重建表；

-------------------------------------------
```

#### DML
```sql
-- 加载文件到表
LOAD DATA [LOCAL] INPATH 'filepath' [OVERWRITE]

-- 查询结果插入到表
INSERT [OVERWRITE] TABLE tablename1 [PARTITION (partcol1=val1, partcol2=val2 ...) [IF NOT EXISTS]]
select_statement1 FROM from_statement;

-- 动态分区插入（有最大的限制，好像一次最多动态插入几百个分区）
set hive.exec.dynamic.partition.mode=nonstrict;
INSERT OVERWRITE TABLE emp_ptn PARTITION (deptno)SELECT empno,ename,job,mgr,hiredate,sal,comm,deptno FROM emp WHERE deptno<30;

-- 查询结果导出
-- 定义列分隔符为'\t'
insert overwrite local directory './test-04'
row format delimited FIELDS TERMINATED BY '\t'
COLLECTION ITEMS TERMINATED BY ','
MAP KEYS TERMINATED BY ':'
select * from src; 
```

#### 视图和索引
视图：没啥好说的。hive2不支持物化视图。
索引：更没啥好说的。索引无法自动更新，每次数据改变都必须重新rebuild。没人用，hive3的物化视图完全替代。

#### 查询
```sql
-- order by & sort by
-- order by 数据保证全局有序；sort by 只保证在一个reduce task中有序；

-- distribute by
-- 如果需要将所有拥有相同key的数据发送到同一个Reducer执行，可以使用它。但是它不保证数据有序；如果需要数据有序，可以使用sort by 

-- join
-- 1. map join -> 用于将小表加载到内存，在map阶段完成join，省略reduce阶段。可以不使用显示的配置MAPJOIN，可以通过配置，自动根据表大小决定是否加载进内存。
SELECT /*+ MAPJOIN(d) */ e.*,d.*
FROM emp e 
JOIN dept d
ON e.deptno = d.deptno
WHERE job='CLERK';
```

#### 部署运维


注意：
1. 以下都是测试部署。
2. 跟正式部署的区别就是 yarn 是否开启 lxc cgroup ，还有一些端口绑定的操作。
3. 正式部署 yarn 参考这个Yarn 部署(非HA)。
4. 必须使用 hadoop 3.1.x 版本。 hadoop 3.3.0 container 存在bug

特性：     
1. Tez/Spark on yarn 方式支持。

##### hive 部署

**注意**
- hadoop spark tez 等 host， hostname 必须在所有配置前处理(简单处理办法，各个组件可以独立绑定，但是独立绑定参数特别麻烦)。
> host 解决办法：DNS or /etc/hosts 解决。hostname 解决办法：hostnamectl set-hostname yourname （hostname 与host 一一对应）


##### 依赖

**1. java 依赖**
- 注意
> reference: https://cwiki.apache.org/confluence/display/HADOOP/Hadoop+Java+Versions
> reference: https://cwiki.apache.org/confluence/display/HADOOP2/HadoopJavaVersions

必须使用java 8，这次使用oracle java 8.261 版本jdk。
hadoop中java 11 只支持 run time(jre);hive 未能支持 java 11 (HiveMetaStoreClient 报bug，官方没说支持11)。 
- 版本
> oracle java 8.261


**2. hdfs 依赖**
- 注意
> reference: https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/Superusers.html
> reference: https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/SingleCluster.html

1. 官方配置下，额外配置代理用户绕开 hdfs 写入权限控制

**思考：为什么hive beeline 指定用户后连接到hdfs 是一个 xxx 的匿名用户？跟ACL 机制好像不是一个检测机制。**

- etc/hadoop/core-site.xml 中添加：
```
<property>
<!-- hadoop.proxyuser.xxx.groups； xxx 为连接指明的用户；groups 为模拟用户组 -->
<name>hadoop.proxyuser.root.groups</name>
<value>*</value>
</property>
<property>
<!-- 允许代理 root 用户连接的ip -->
<name>hadoop.proxyuser.root.hosts</name>
<value>*</value>
</property>
```

2. 启动命令
> 按照节点角色启动，不要使用start cluster，这样可以做到无需配置机器互信

- 执行
```
hdfs -daemon start namenode
hdfs -daemon start secondarynamenode
hdfs -daemon start datanode
首次启动：
hdfs namenode -format
非首次启动(刷新配置方式，每一台机器都需要执行)：
hdfs dfsadmin –refreshSuperUserGroupsConfiguration
```

**3. yarn 依赖**

1. 最简单的yarn 依赖配置方式如下
> 该配置只能简单运行,实际必须将yarn 配置为 LinuxContainerExecutor 容器，否则部分语句会因为安全问题禁止运行

- etc/hadoop/mapred-site.xml
```
<configuration>
<property>
<name>mapreduce.framework.name</name>
<value>yarn</value>
</property>
<property>
<name>mapreduce.application.classpath</name>
<value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
</property>
</configuration>
```
- etc/hadoop/yarn-site.xml:
```
<configuration>
<property>
<name>yarn.nodemanager.aux-services</name>
<value>mapreduce_shuffle</value>
</property>
<property>
<name>yarn.nodemanager.env-whitelist</name>
<value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
</property>
</configuration>
```


**4. mysql 依赖**
- 注意
> 提前建立 hive databases；


**5. hive 配置**

reference: 

1. 第一步 下载&解压
wget https://downloads.apache.org/hive/hive-3.1.2/apache-hive-3.1.2-bin.tar.gz
tar -xzvf apache-hive-3.1.2-bin.tar.gz

2. 第二步 配置基础信息
reference: https://cwiki.apache.org/confluence/display/Hive/AdminManual+Metastore+Administration (remote metastore )
```
> cd apache-hive-3.1.2-bin
> hive-site.xml
><configuration>
<property>
<name>javax.jdo.option.ConnectionURL</name>
<value>jdbc:mysql://node-test:3306/hive?serverTimezone=GMT%2B8</value>
</property>
<property>
<!-- value 值需要与driver 一一对应 -->
<name>javax.jdo.option.ConnectionDriverName</name>
<value>com.mysql.cj.jdbc.Driver</value>
</property>
<property>
<name>javax.jdo.option.ConnectionUserName</name> 
<value>i_don't_know</value> 
</property>
<property>
<name>javax.jdo.option.ConnectionPassword</name> 
<value>password</value>
</property>
</configuration>" >>./conf/hive-site.xml
```
3. mysql 初始化，建立以上信息的库(hive库)，该账号需要拥有该库所有权限。

1. mysql 库配置
reference: https://www.cnblogs.com/wangyueping/p/11258028.html

- 执行
```
> mysql -u root -p 
> create user 'hive'@'%' identified by 'hive';
> create database hive;
> grant all privileges on hive.* to "hive"@'%';
> flush privileges;
```

2. hive schema tool 初始化 hive 元数据存储
reference： https://cwiki.apache.org/confluence/display/Hive/Hive+Schema+Tool
以mysql 为例

- 执行
``` 
schematool -dbType mysql -initSchema
```

4. JAVA_HOME HADOOP_HOME HIVE_HOME 环境变量配置
5. hdfs 配置工作目录，数据目录
reference: https://cwiki.apache.org/confluence/display/Hive/GettingStarted (Running Hive 部分)

- 执行
```
$HADOOP_HOME/bin/hadoop fs -mkdir -p /tmp
$HADOOP_HOME/bin/hadoop fs -mkdir -p /user/hive/warehouse
$HADOOP_HOME/bin/hadoop fs -chmod g+w /tmp
$HADOOP_HOME/bin/hadoop fs -chmod g+w /user/hive/warehouse
```

6. 替换guava依赖(解决hive 中依赖的这个包与hadoop 需要的版本不一致)
```
cp $HADOOP_HOME/share/hadoop/common/lib/guava-27.0-jre.jar $HIVE_HOME/lib/
```
7. 启动 metastore
reference: reference: https://cwiki.apache.org/confluence/display/Hive/AdminManual+Metastore+Administration (remote metastore )

- 执行
```
nohup $HIVE_HOME/bin/hive --service metastore >> metastore_log 2>&1 &
```

8. 启动 HiveServer2
reference: https://cwiki.apache.org/confluence/display/Hive/Setting+Up+HiveServer2

- 执行
```
nohup $HIVE_HOME/bin/hiveserver2 >> hiveserver2_log 2>&1 &
```

9. beeline 登录测试
reference: https://cwiki.apache.org/confluence/display/Hive/HiveServer2+Clients#HiveServer2Clients-Beeline--NewCommandLineShell

- 执行


```
$HIVE_HOME/bin/beeline -u jdbc:hive2://127.0.0.1:10000 -n root
> create table temp (show int);
> insert into temp values(111);
> select * from temp;
``` 

**5. Tez 配置**

- **注意**

> 1. 不要将编译源码后的TEZ放到 Yarn 上。hive 3.1.2 配合 hadoop 3.1.3 只能使用 Tez 0.9.X (0.9.0 版本有bug)。
> 2. 本地 tez 需要重新编译为 3.1.3 的 hadoop 依赖，yarn 集群中tez 依赖必须使用0.9.x 中自带的。
> Tez 部分方法强依赖 guava-11/ guava-19 ，编译通过也无法启动(提交到集群后，tez master 执行DAG新建container 会失败)。




- 下载 tez-0.9.2(部署可以跳过)

- 执行
1. 本地编译 tez 0.9.2 源码(需要hadoop shim 为2.8那个版本的源码，guava 仍为19那个版本，因为hive-exec 3.1.2 里面gua仍为19)
- 官方做法
```
> mvn clean package -Dmaven.test.skip=true -Dmaven.javadoc.skip=true
> cd /tez-dist/target
> cp tez-0.9.2-SNAPSHOT-minimal.tar.gz /data/
> cd /data/
> mkdir apache-tez-0.9.2-bin
> mv tez-0.9.2-SNAPSHOT-minimal.tar.gz ./apache-tez-0.9.2-bin
> tar -xzvf tez-0.9.2-SNAPSHOT-minimal.tar.gz
```
- 线上包处理，已经编译好，直接从84拉过去就好

2. yarn 集群中放入 TEZ 相关配置(无需重启yarn)
- 官方源码处理步骤
```
> tar -xzvf apache-tez-0.9.2-bin.tar.gz
> $HADOOP_HOME/bin/hadoop fs -mkdir -p /apps/tez-0.9.2
> $HADOOP_HOME/bin/hadoop fs -copyFromLocal share/tez.tar.gz /apps/tez-0.9.2
```
- 线上包处理步骤
```
> tar -xzvf tez-0.9.2-hadoop3.1-1.1.2.tar.gz
> $HADOOP_HOME/bin/hadoop fs -mkdir -p /apps/
> $HADOOP_HOME/bin/hadoop dfs -put ./tez-0.9.2-hadoop3.1-1.1.2 /apps
```

3. 配置 hive-env.sh

```
export TEZ_HOME=/opt/software/tez-0.9.2-hadoop3.1-1.1.2
export TEZ_CONF_DIR=/opt/software/hadoop-3.1.3/etc/hadoop
#tez conf 不起效，只能放入到 $HADOOP_HOME/etc/hadoop 下才能起效
export HADOOP_CLASSPATH=${TEZ_HOME}/*:${TEZ_HOME}/conf:${TEZ_HOME}/lib/*:${HADOOP_CLASSPATH}
```

4. 配置 hive-site.xml
```
<configuration>
<property>
<name>javax.jdo.option.ConnectionURL</name>
<value>jdbc:mysql://172.16.7.63:3306/test_hive_lzy?serverTimezone=GMT%2B8</value>
</property>
<property>
<!-- value 值需要与driver 一一对应 -->
<name>javax.jdo.option.ConnectionDriverName</name>
<value>com.mysql.cj.jdbc.Driver</value>
</property>
<property>
<name>javax.jdo.option.ConnectionUserName</name> 
<value>123456</value> 
</property>
<property>
<name>javax.jdo.option.ConnectionPassword</name> 
<value>123456</value>
</property>
<property>
<name>hive.execution.engine</name>
<value>tez</value>
</property>
<property>
<name>hive.tez.container.size</name>
<value>512</value>
<!-- 不起效，只有大于1G 才有效 -->
<description>By default Tez will spawn containers of the size of a mapper. This can be used to overwrite.</description>
</property>
<property>
<name>hive.metastore.uris</name>
<value>thrift://hive69:9083</value>
</property> 
</configuration>
```

5. 配置 tez-site.xml

```
> vim $HADOOP_HOME/etc/hadoop/tez-site.xml
<configuration> 
<property> 
<name>tez.lib.uris</name>
<value>${fs.defaultFS}/apps/tez-0.9.2-hadoop3.1-1.1.2/,${fs.defaultFS}/apps/tez-0.9.2-hadoop3.1-1.1.2/lib/</value> 
<!-- <value>${fs.defaultFS}/apps/tez-0.9.2/tez.tar.gz</value> -->
<!-- <value>${fs.defaultFS}/apps/tez-0.10.0/tez-0.10.1-SNAPSHOT-minimal.tar.gz</value> -->
</property> 
<property> 
<name>tez.use.cluster.hadoop-libs</name> 
<value>false</value> 
<!-- 貌似不起效-->
</property> 
<property> 
<name>tez.am.resource.memory.mb</name> 
<value>512</value> 
<!-- 不起效，只有大于1G 才有效-->
</property> 
<property> 
<name>tez.am.resource.cpu.vcores</name> 
<value>1</value> 
</property> 
<property> 
<name>tez.task.resource.memory.mb</name> 
<value>512</value> 
<!-- 不起效，只有大于1G 才有效-->
</property> 
<property> 
<name>tez.task.resource.cpu.vcores</name> 
<value>1</value> 
</property> 
</configuration>

```
6. 重启hiveserver2 ，登入 beeline
```
$HIVE_HOME/bin/beeline -u jdbc:hive2://127.0.0.1:10000 -n root
```


**6. Spark 支持**

**reference:**
1. https://spark.apache.org/docs/latest/configuration.html （spark 参数相关）
2. https://spark.apache.org/docs/latest/running-on-yarn.html （spark on yarn 相关）
3. https://cwiki.apache.org/confluence/display/Hive/Hive+on+Spark%3A+Getting+Started
**注意：**
1. spark on yarn 除了 spark history server（历史作业服务），可能需要启动，其他服务无需启动。


- 执行
1. 下载 spark without hadoop && 配置环境变量
```
> vim conf/spark-env.sh
export SPARK_DIST_CLASSPATH=`$HADOOP_HOME/bin/hadoop classpath`
HADOOP_CONF_DIR=${HADOOP_HOME}/etc/hadoop
YARN_CONF_DIR=${HADOOP_HOME}/etc/hadoop
SPARK_EXECUTOR_CORES=2
SPARK_EXECUTOR_MEMORY=512MB
SPARK_DRIVER_MEMORY=512MB

# 可以不配置，因为我们不会直接启动spark 或者直接通过spark 提交作业
> vim conf/spark-defaults.conf
spark.yarn.historyServer.address=hive69:18080
spark.yarn.historyServer.allowTracking=true
spark.eventLog.dir=hdfs://hive69/logs/spark/eventlogs
spark.eventLog.enabled=true
spark.history.fs.logDirectory=hdfs://hive69/logs/spark/hisLog
```
2. HIVE_HOME/lib 下放入SPARK_HOEM/jars 下依赖
```
scala-library
spark-core
spark-network-common

ps : spark 2.4.7 与 hive 3.1.2 需要额外导入一个 nosafe 包
```

3. HIVE_HOME/hive-env.sh 改为

```
export TEZ_HOME=/data/apache-tez-0.9.2-bin
# conf 不起效，只能放入到 $HADOOP_HOME/etc/hadoop 下才能起效，估计是版本更新问题
export HADOOP_CLASSPATH=${TEZ_HOME}/*:${TEZ_HOME}/conf:${TEZ_HOME}/lib/*:${HADOOP_CLASSPATH}
export SPARK_HOME=/data/spark-2.4.7-bin-without-hadoop
```

4. HIVE_HOME/hive-site.xml 改为

```
<configuration>
<property>
<name>javax.jdo.option.ConnectionURL</name>
<value>jdbc:mysql://172.16.7.63:3306/test_hive_lzy?serverTimezone=GMT%2B8</value>
</property>
<property>
<!-- value 值需要与driver 一一对应 -->
<name>javax.jdo.option.ConnectionDriverName</name>
<value>com.mysql.cj.jdbc.Driver</value>
</property>
<property>
<name>javax.jdo.option.ConnectionUserName</name>
<value>test_lzy</value>
</property>
<property>
<name>javax.jdo.option.ConnectionPassword</name>
<value>123456</value>
</property>
<property>
<name>hive.execution.engine</name>
<value>tez</value>
</property>
<property>
<name>hive.tez.container.size</name>
<value>512</value>
<description>By default Tez will spawn containers of the size of a mapper. This can be used to overwrite.</description>
</property>
<property>
<name>hive.server2.thrift.bind.host</name>
<value>hive69</value>
</property>
<property>
<name>spark.eventLog.enabled</name>
<value>true</value>
</property>
<property>
<name>spark.eventLog.dir</name>
<value>hdfs://hive69:9000/logs/spark/eventlogs</value>
</property>
<property>
<name>spark.executor.memory</name>
<value>512m</value>
</property>
<property>
<name>spark.driver.memory</name>
<value>512m</value>
</property>
<property>
<name>spark.serializer</name>
<value>org.apache.spark.serializer.KryoSerializer</value>
</property>
<property>
<name>spark.executor.cores</name>
<value>2</value>
</property>
<property>
<name>spark.executor.instances</name>
<value>10</value>
</property>
<property>
<name>spark.master</name>
<value>yarn-cluster</value>
</property>
</configuration>

```

5. 解决跨节点读取hive 配置问题(Spark on hive 缺陷)。

存在问题：
> 以上配置，非 hive GetWay 机器会因为无法读取 hive-site.xml 配置引发各种问题。

**reference:**
1. https://stackoverflow.com/questions/45477155/missing-hive-site-when-using-spark-submit-yarn-cluster-mode

解决办法：
1. spark.yarn.dist.files。
2. 将HIVE_HOME/conf/hive-site.xml 复制一份到 SPARK_HOME/CONF 下。




