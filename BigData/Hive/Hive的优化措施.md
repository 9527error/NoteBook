#### Hive相较于MapReduce的优势

1.hive强大之处不要求数据转换成特定的格式，而是利用hadoop本身InputFormat API来从不同的数据源读取数据，同样地使用OutputFormat API将数据写成不同的格式。

2.Hive拥有统一的元数据管理，所以和Spark、Impala等SQL引擎是通用的。通用是指，在拥有了统一的metastore之后，在Hive中创建一张表，在Spark/Impala中是能用的；反之在Spark中创建一张表，在Hive中也是能用的，只需要共用元数据，就可以切换SQL引擎

3.不仅如此Hive使用SQL语法，提供快速开发的能力，还可以通过用户定义的函数（UDF），用户定义的聚合（UDAF）和用户定义的表函数（UDTF）进行扩展，避免了去写mapreducce，减少开发人员的学习成本。



#### Hive的函数

**UDF、UDAF、UDTF**的区别：

- UDF（User-Defined-Function）一进一出
- UDAF（User-Defined Aggregation Funcation）聚集函数，多进一出
- UDTF（User-Defined Table-Generating Functions）一进多出，如lateral view explore()



#### Hive的优化



##### 慎用API

大数据场景下不害怕数据量大，害怕的是数据倾斜，怎样避免数据倾斜，找到可能产生数据倾斜的函数尤为关键，数据量较大的情况下，**慎用count(distinct)，count(distinct)容易产生倾斜问题。**



##### 自定义UDAF函数优化

sum，count，max，min等UDAF，不怕数据倾斜问题，hadoop在map端汇总合并优化，是数据倾斜不成问题



##### 设置合理的map reduce的task数量

###### map阶段优化

```
mapred.min.split.size: 指的是数据的最小分割单元大小；min的默认值是1B
mapred.max.split.size: 指的是数据的最大分割单元大小；max的默认值是256MB
通过调整max可以起到调整map数的作用，减小max可以增加map数，增大max可以减少map数。
需要提醒的是，直接调整mapred.map.tasks这个参数是没有效果的。
```

**小文件多的情况：减少map数量，map执行前合并小文件**

**大文件的情况：增加map数量，拆分成多个map任务执行**



###### reduce阶段优化

Reduce的个数对整个作业的运行性能有很大影响。如果Reduce设置的过大，那么将会产生很多小文件，对NameNode会产生一定的影响，而且整个作业的运行时间未必会减少；如果Reduce设置的过小，那么单个Reduce处理的数据将会加大，很可能会引起OOM异常。

　如果设置了mapred.reduce.tasks/mapreduce.job.reduces参数，那么Hive会直接使用它的值作为Reduce的个数；如果mapred.reduce.tasks/mapreduce.job.reduces的值没有设置（也就是-1），那么Hive会根据输入文件的大小估算出Reduce的个数。根据输入文件估算Reduce的个数可能未必很准确，因为Reduce的输入是Map的输出，而Map的输出可能会比输入要小，所以最准确的数根据Map的输出估算Reduce的个数。

1. Hive自己如何确定reduce数：

　　reduce个数的设定极大影响任务执行效率，不指定reduce个数的情况下，Hive会猜测确定一个reduce个数，基于以下两个设定：

```
方法1：
set hive.exec.reducers.bytes.per.reducer=500000000; （500M）
方法2：
set mapred.reduce.tasks=15;
```

2.reduce个数不是越多越好

同map一样，启动和初始化reduce也会消耗时间和资源；

　　另外，有多少个reduce，就会有个多少个输出文件，如果生成了很多个小文件，那么如果这些小文件作为下一个任务的输入，则也会出现小文件过多的问题





##### 小文件的合并优化

我们知道文件数目小，容易在文件存储端造成瓶颈，给HDFS带来压力，影响处理效率。对此，可以通过合并Map和Reduce的结果文件来消除这样的影响。

　　用于设置合并的参数有：

- - 是否合并Map输出文件：hive.merge.mapfiles=true（默认值为true）
  - 是否合并Reduce端输出文件：hive.merge.mapredfiles=false（默认值为false）
  - 合并文件的大小：hive.merge.size.per.task=256*1000*1000（默认值为256000000）



###### 小文件是怎么产生的

- - 动态分区插入数据，产生大量的小文件，从而导致map数量剧增；
  - reduce数量越多，小文件也越多（reduce的个数和输出文件是对应的）；
  - 数据源本身就包含大量的小文件。

###### 小文件问题的影响

- - 从Hive的角度看，小文件会开很多map，一个map开一个JVM去执行，所以这些任务的初始化，启动，执行会浪费大量的资源，严重影响性能。
  - 在HDFS中，每个小文件对象约占150byte，如果小文件过多会占用大量内存。这样NameNode内存容量严重制约了集群的扩展。

**小文件的解决方案**

从小文件产生的途径就可以从源头上控制小文件数量，方法如下：

- - 使用Sequencefile作为表存储格式，不要用textfile，在一定程度上可以减少小文件；
  - 减少reduce的数量（可以使用参数进行控制）；
  - 少用动态分区，用时记得按distribute by分区；

　　　　对于已有的小文件，我们可以通过以下几种方案解决：

- - 使用hadoop archive命令把小文件进行归档；

  - 重建表，建表时减少reduce数量；

  - 通过参数进行调节，设置map/reduce端的相关参数，如下：

    ```shell
    
    ```



##### SQL优化

###### 	列裁剪

可以只读取查询中所需要用到的列，而忽略其他列。

###### 	分区裁剪

可以在查询的过程中减少不必要的分区。

###### 	使用union all替换union

###### 	使用group by 替换count(distinct)

###### 	Map Join操作

​	可以用MapJoin把小表全部加载到内存在map端进行join，避免reducer处理

- 开启MapJoin参数设置：

　　　　1) 设置自动选择MapJoin

　　　　　　set hive.auto.convert.join = true;默认为true

　　　　2) 大表小表的阀值设置（默认25M一下认为是小表）：

　　　　　　set hive.mapjoin.smalltable.filesize=25000000;



##### 存储格式

使用parquet列式存储，只需要遍历列数据，可以大大减少处理数据量

##### 压缩格式

Hive最终是转为MapReduce程序来执行的，而**MapReduce的性能瓶颈在于网络IO和磁盘IO**，要解决性能瓶颈，最主要的是减少数据量，对数据进行压缩是个好的方式。**压缩虽然是减少了数据量，但是压缩过程要消耗CPU的，但是在Hadoop中，往往性能瓶颈不在于CPU，CPU压力并不大，所以压缩充分利用了比较空闲的CPU。**



##### Jvm重用

Hadoop通常是使用派生JVM来执行map和reduce任务的。这时JVM的启动过程可能会造成相当大的开销，尤其是执行的job包含成百上千的task任务的情况。JVM重用可以使得JVM示例在同一个job中时候，通过参数mapred.job.reuse.jvm.num.tasks来设置。