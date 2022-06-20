查看spark源码
```java
/**
   * Returns a new Dataset that has exactly `numPartitions` partitions, when the fewer partitions
   * are requested. If a larger number of partitions is requested, it will stay at the current
   * number of partitions. Similar to coalesce defined on an `RDD`, this operation results in
   * a narrow dependency, e.g. if you go from 1000 partitions to 100 partitions, there will not
   * be a shuffle, instead each of the 100 new partitions will claim 10 of the current partitions.
   *
   * However, if you're doing a drastic coalesce, e.g. to numPartitions = 1,
   * this may result in your computation taking place on fewer nodes than
   * you like (e.g. one node in the case of numPartitions = 1). To avoid this,
   * you can call repartition. This will add a shuffle step, but means the
   * current upstream partitions will be executed in parallel (per whatever
   * the current partitioning is).
   *
   * @group typedrel
   * @since 1.6.0
   */
  def coalesce(numPartitions: Int): Dataset[T] = withTypedPlan {
    Repartition(numPartitions, shuffle = false, logicalPlan)
  }

```
- 1、coalesce操作只能减少分区，它是使用现有分区来减少shuffer的数据量
- 2、repartion操作可以增加分区，也可减少分区，它创建新的分区，进行完全的shuffer操作

在减少分区时使用coalesce或者partition需要具体分析，如果需要急剧的缩小分区数量，则使用带有shuffle的repartition



