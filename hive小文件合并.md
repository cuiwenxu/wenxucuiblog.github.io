@[TOC](hive小文件合并)
# 小文件带来的问题

关于这个问题的阐述可以读一读Cloudera的这篇文章。简单来说，HDFS的文件元信息，包括位置、大小、分块信息等，都是保存在NameNode的内存中的。每个对象大约占用150个字节，因此一千万个文件及分块就会占用约3G的内存空间，一旦接近这个量级，NameNode的性能就会开始下降了。

此外，HDFS读写小文件时也会更加耗时，因为每次都需要从NameNode获取元信息，并与对应的DataNode建立连接。对于MapReduce程序来说，小文件还会增加Mapper的个数，每个脚本只处理很少的数据，浪费了大量的调度时间。当然，这个问题可以通过使用CombinedInputFile和JVM重用来解决。

# Hive小文件产生的原因

Reducer数量太多

# 解决方法
解决小文件的问题可以从两个方向入手：
- 1、 输入合并。即在Map前合并小文件
- 2、 输出合并。即在输出结果的时候合并小文件

思维导图如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210524205729403.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)


## 配置Map输入合并
### 开关
`set  hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat; `可以开启Map前进行小文件合并
### 阈值
```sql
-- 每个Map最大输入大小，决定合并后的文件数
set mapred.max.split.size=256000000;
-- 一个节点上split的至少的大小，决定了多个data node上的文件是否需要合并
set mapred.min.split.size.per.node=100000000;
-- 一个交换机下split的至少的大小，决定了多个交换机上的文件是否需要合并
set mapred.min.split.size.per.rack=100000000;
```


## 配置Hive结果合并(分map only任务和mapred两种)
### 开关
```sql
hive.merge.mapfiles       #在map-only job后合并文件，默认true
hive.merge.mapredfiles    #在map-reduce job后合并文件，默认false
```

我们可以通过一些配置项来使Hive在执行结束后对结果文件进行合并：
```sql
hive.merge.size.per.task      #合并后每个文件的大小，默认256000000
hive.merge.smallfiles.avgsize #平均文件大小，是决定是否执行合并操作的阈值，默认16000000
```

## 配置完成后hive具体怎么合并小文件
`Hive在对结果文件进行合并时会执行一个额外的map-only任务，mapper的数量是文件总大小除以size.per.task参数所得的值`，
触发合并的条件是：
> 根据查询类型不同，相应的mapfiles/mapredfiles参数需要打开；
   结果文件的平均大小需要大于avgsize参数的值。
