@[TOC](flink如何做join)

# 概览
在单节点小数据量的场景下，最简单的join方式是nested-loop join，该join会产生笛卡尔积，复杂度为o(n2)，显然不适合大数据量join。
在分布式系统上的join通常被分为两步：
- 1、各节点上的数据重分区，具有相同key的数据放到同一节点
- 2、单个节点进行的stand alone join
用flink术语来说，第一阶段称为`ship stategy（分发）`，第二阶段称为`local strategy（本地处理）`。 在下文中，我将介绍flink的两种策略，下文称做join的两个数据集为R和S。
# flink join的两个阶段
## ship strategy(分发)

flink提供了两种分发策略：

- the Repartition-Repartition strategy (RR，join左右都要重分区)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210317160243727.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
- the Broadcast-Forward strategy (BF，把其中一个表广播到另一个表所在节点上)  
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210317160309622.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
## local strategy(本地处理)
在数据分发完毕后，就进入了本地处理阶段，本地处理阶段,flink的本地join有两种策略
- the Sort-Merge-Join strategy (SM) 
- the Hybrid-Hash-Join strategy (HH).


Sort-Merge-Join的工作方式是首先根据两个输入数据集的join key属性进行排序（排序阶段），然后将排序后的数据集合并（合并阶段）。如果数据集的本地分区足够小，则在内存中进行排序。否则，将借助硬盘进行外部合并排序。下图为Sort-Merge-Join的工作流程：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210317175334476.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
Hybrid-Hash-Join将其输入区分为构建侧输入和探针侧输入，并分为两个阶段工作，先进行构建阶段再进行探针阶段。在构建阶段，flink进程读取构建侧输入，并将所有数据元素插入到内存中的哈希表中，该哈希表索引为join key。如果哈希表大小超出内存，则哈希表的部分内容将被写入本地文件系统。在完全消耗了构建侧输入之后，构建阶段结束。在探测阶段，flink进程读取探测侧输入，并使用join key去构建侧哈希表查找内容。如果该元素落入散列到磁盘的哈希索引范围内，则该元素也会写入磁盘。否则，该元素将立即与哈希表中的所有匹配元素结合在一起。如果哈希表完全适合工作内存，则在完全消耗完探针侧输入之后，将完成join。否则，将当前的哈希表移出内存，并使用生成端输入的溢出部分来构建新的哈希表。该哈希表由溢出的探测器侧输入的相应部分探测。最终，所有数据都被加入。如果哈希表完全适合工作内存，则Hybrid-Hash-Joins的性能最佳。但是，即使构建侧输入不适合内存，Hybrid-Hash-Join也具有非常好的性能。在这种情况下，将部分保留内存中的处理，并且仅一部分构建侧和探针侧数据需要写入本地文件系统或从本地文件系统读取。下图说明了Hybrid-Hash-Join的工作方式。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210317182333626.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
# flink怎么选择join策略？
分发阶段和本地处理阶段互不依赖，可以独立选择。因此，Flink可以通过将三种分发策略（RR，BF与R广播，BF与S广播）中的任何一个与三个本地处理策略（SM, HH with R being build-side, HH with S being build-side）结合起来，以九种不同的方式执行两个数据集R和S的联接。 这些策略组合中的每一种都会根据数据大小和可用内存量而导致不同的执行性能。对于较小的数据集R和较大的数据集S，广播R并将其用作Hybrid-Hash-Join的构建侧输入通常是一个不错的选择。如果两个数据集都很大，或者在许多并行实例上执行了join，则对两个输入进行重新分区是一个明智的选择。

flink有基于成本的优化器，该优化器会自动为所有运算符（包括join）选择执行策略。优化器估计通过网络传送并写入磁盘的数据量。如果无法获得输入数据的可靠大小估计，则优化器将退回到可靠的默认选择。 但有一种情况可能是用户比优化器更了解如何执行特定的联接。与关系数据库系统类似，Flink提供了optimizer hint，以告诉优化器选择哪种连接策略。

# 如果数据量过大，怎么避免分发（pre partition、类似spark bucket）
我们已经看到，现成的分布式join在Flink中工作得很好。但是，如果您的数据如此之大以至于您不想在整个群集中对其进行shuffle，那该怎么办？最近，我们在Flink中添加了一些功能，用于在输入拆分中指定语义属性（分区和排序），并在同一位置读取本地输入文件。有了这些工具，就可以从本地文件系统中加入预先分区的数据集，而无需在群集的网络上分发（类似spark的分桶）。


