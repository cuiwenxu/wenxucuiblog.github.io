@[TOC](hudi介绍)
# 数据湖产生背景
数据湖的产出原因是数据处理架构的升级，最初版本的lambda架构，在Processor上是两套结构（stream processor和batch processor），数据服务层自然也就分为两套（实时对应speed serving，离线对应batch serving）,如下图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210608205406878.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
在引入kappa架构之后，processor只剩下stream processor，数据服务层却难有一个框架来进行统一,如下图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210608204854980.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
kappa架构下对数据服务层的要求如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210608210902927.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
>从功能特点上来分析，数据湖非常像性能非常好的关系型数据库（支持upsert、delete，有table schema，支持事务acid）

# hudi介绍
Apache Hudi(发音为“hoodie”)在hadoop兼容存储上提供流操作api，比如：
- 更新/删除记录(如何更改表中的记录?)
- 更改流(如何获取更改的记录?)

在本节中，我们将讨论关键概念和术语，这些概念和术语对于理解和有效使用这些api非常重要。
## Timeline
其核心是，Hudi维护了在不同时刻在表上执行的所有操作的时间轴，这有助于提供表的瞬时视图，同时还有效地支持按到达顺序检索数据。一个hudi instant(瞬态)由以下组件组成

- instant action(瞬态动作):对表执行的动作类型
- instant time(瞬态时间):瞬态时间通常是一个时间戳(例如:20190117010349)，它按照动作的开始时间顺序单调地增加。
- State:当前状态

Hudi保证在时间轴上执行的动作是原子的&时间轴基于瞬态时间一致。

关键的instant action包括：
- COMMITS (提交)——提交表示将一批记录原子地写到一个表中。
- CLEANS(清除)——后台进程，作用是清除表中不再需要的旧版本文件。
- DELTA_COMMIT(增量提交)——增量提交指的是将一批记录原子地写入到一个MergeOnRead类型表中，其中一些/所有数据都可以写入增量日志。
- COMPACTION(压缩)——后台进程，作用是协调Hudi中不同数据结构的例如:将更新从基于行的日志文件移动到列式格式。在内部，压缩显示为时间轴上的特殊提交
- ROLLBACK(回滚) -表示一个提交/增量提交不成功&回滚，删除了在这样的写过程中产生的任何部分文件
- SAVEPOINT(检查点) -将某些文件组标记为“已保存”，这样cleaner就不会删除它们。在灾难/数据恢复场景下，它有助于将表恢复到时间轴上的某个点。

任何给定的瞬态都可能处于下列状态之一
- REQUESTED —— 表示某个操作已被计划，但尚未启动
- INFLIGHT —— 表示当前正在执行的动作
- COMPLETED —— 表示完成
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210609102900387.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)

上面的例子显示了在10点到10点20之间发生在Hudi表上的upserts事件，大约每5分钟发生一次，commit元数据和其他后台清理/压缩一起留在了Hudi时间轴上。要特别注意的一点是：提交时间对应数据的到达时间(10:20AM)，而实际数据组织是根据事件时间。这是两个不同的概念。

当出现延迟到达的数据时(预计9点到达的数据在1小时后10点20分到达)，我们可以看到upsert将新数据生成到更老的时间桶/文件夹中。在时间轴的帮助下，尝试获取10:00小时以来成功提交的所有新数据的增量查询能够非常有效地只使用更改的文件，而不必扫描所有的时间桶> 07:00。

## 文件Layout
Hudi organizes a table into a directory structure under a basepath on DFS. Table is broken up into partitions, which are folders containing data files for that partition, very similar to Hive tables. Each partition is uniquely identified by its partitionpath, which is relative to the basepath.

Within each partition, files are organized into file groups, uniquely identified by a file id. Each file group contains several file slices, where each slice contains a base file (*.parquet) produced at a certain commit/compaction instant time, along with set of log files (*.log.*) that contain inserts/updates to the base file since the base file was produced. Hudi adopts a MVCC design, where compaction action merges logs and base files to produce new file slices and cleaning action gets rid of unused/older file slices to reclaim space on DFS.

Hudi将表组织成DFS上的一个基础路径下的目录结构。表被分成几个分区，这些分区是包含该分区的数据文件的文件夹，非常类似于Hive表。每个分区都由它的partitionpath唯一标识，它是相对于basepath的。

在每个分区中，文件被组织成由文件id唯一标识的文件组。每个文件组包含几个文件片，其中每个片包含一个在特定提交/压缩瞬间产生的基本文件(*.parquet)，以及一组日志文件(*.log.*)，这些日志文件包含自生成基本文件以来对基本文件的插入/更新。Hudi采用MVCC设计，其中压缩操作合并日志和基本文件以生成新的文件片，而清理操作删除未使用的/旧的文件片以回收DFS上的空间。

