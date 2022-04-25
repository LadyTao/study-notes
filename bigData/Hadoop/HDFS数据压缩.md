#### Hadoop使用压缩可以带来的好处：
1：节省磁盘空间
2：加快数据在磁盘和网络中的传输速度。

#### 常见的压缩格式及特点

| 压缩格式 | 工具 | 算法 | 扩展名 | 多文件 | 可分割性 |
| --- | --- | --- | --- | --- | --- |
| DEFAULT | 无 |DEFAULT  | .default | NO | NO |
| GZIP | gzip | DEFAULT | .gzp | NO | NO |
| ZIP | zip | DEFAULT | .zip | YES | YES |
| BZIP2 | bzip2 | BZIP2 | .bz2 | NO | YES |
| LZO | lzop | LZO | .lzo | NO | YES |

特征比较：

| 压缩格式 | split | native | 压缩率 | 是否hadoop自带 | 速度 | 换成压缩格式后，原来的程序是否需要更改 | 应用场景 |
| --- | --- | --- | --- | --- | --- | --- | --- | 
| gzip | NO | YES | 很高 | YES | 比较快 | 无需修改 | 压缩后文件单个大小在130M以内可以考虑 | 
| lzo | YES | YES | 比较高 | NO | 很快 | 需修改，建索引，指定格式 | 一个很大的文件，文件越大lzo优势越明显 | 
| snappy | NO | YES | 比较高 | NO | 很快 | 无需修改 | MR任务当map的输出数据很大时可以考虑，作为中间结果 | 
| bzip2 | YES | NO | 最高 | YES | 慢 | 无需修改 | 速度要求不高，但是需要极高的压缩率；单个文件比较大同时想支持split特性的可以考虑 | 


**补充1：**
native特性：java的native特性，直接调用系统指令，效率高；
split特性：文件如果可以被split，那么在执行mr任务时这个文件可以被切分为多个map处理。如果文件不能切分且文件太大超出内存限制，则可能报OOM异常。

**补充2：关于zstd压缩格式**
zstd是facebook在2016年开源的一种无损压缩算法，其压缩率与解压率性能都很突出。
hadoop需要先在linux环境安装zstd：
> yum -y install \*zstd\*

安装完成后需要check一下：
> hadoop checknative -a

如果发现zstd是true，说明已经可用了。

如果需要在Hive表中使用这个压缩算法，有两种方式：
* 如果是parquet的，TBLPROPERTIES中新加属性：`"parquet.compression"="ZSTD"`。ORC目前不支持。如果orc一定要用，需要重新编译hive，集成高版本ORC。

* 全局设置
修改mapred-site.xml文件
```xml
<!--PARQUET格式的配置-->
<property>
    <name>mapreduce.output.fileoutputformat.compress</name>
    <value>true</value>
</property>
<property>
    <name>mapreduce.output.fileoutputformat.compress.codec</name>
    <value>org.apache.hadoop.io.compress.ZStandardCodec</value>
</property>
```
