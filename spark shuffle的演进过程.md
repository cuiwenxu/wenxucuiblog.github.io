spark各版本shuffle的变化
+ Spark 0.8及以前 Hash Based Shuffle
+ Spark 0.8.1 为Hash Based Shuffle引入File Consolidation机制
+ Spark 1.1 引入Sort Based Shuffle，但默认仍为Hash Based Shuffle
+ Spark 1.2 默认的Shuffle方式改为Sort Based Shuffle
+ Spark 1.4 引入Tungsten-Sort Based Shuffle
+ Spark 1.6 Tungsten-sort并入Sort Based Shuffle
+ Spark 2.0 Hash Based Shuffle退出历史舞台

接下来详细研究版本演化的驱动因素
##### Hash Based Shuffle
最开始的时候使用的是 Hash Based Shuffle， 这时候每一个Mapper会根据Reducer的数量创建出相应的bucket，bucket的数量是M*R ，其中M是MapTask的个数，R是ReduceTask的个数。这样会产生大量的小文件，对文件系统压力很大，而且也不利于IO吞吐量。如下图
![在这里插入图片描述](https://img-blog.csdnimg.cn/202010181758324.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70#pic_center)
后面做了优化，把在同一个core上运行的多个Mapper task 输出合并到同一个文件，这样文件数目就变成了 cores*R 个了
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201018175926246.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70#pic_center)
##### Sort Based Shuffle
经过FileConsolidation之后，同一个core上会产出



##### SortShuffleManager代码解析
```java
override def registerShuffle[K, V, C](
      shuffleId: Int,
      numMaps: Int,
      dependency: ShuffleDependency[K, V, C]): ShuffleHandle = {
    if (SortShuffleWriter.shouldBypassMergeSort(conf, dependency)) {
      // If there are fewer than spark.shuffle.sort.bypassMergeThreshold partitions and we don't
      // need map-side aggregation, then write numPartitions files directly and just concatenate
      // them at the end. This avoids doing serialization and deserialization twice to merge
      // together the spilled files, which would happen with the normal code path. The downside is
      // having multiple files open at a time and thus more memory allocated to buffers.
      new BypassMergeSortShuffleHandle[K, V](
        shuffleId, numMaps, dependency.asInstanceOf[ShuffleDependency[K, V, V]])
    } else if (SortShuffleManager.canUseSerializedShuffle(dependency)) {
      // Otherwise, try to buffer map outputs in a serialized form, since this is more efficient:
      new SerializedShuffleHandle[K, V](
        shuffleId, numMaps, dependency.asInstanceOf[ShuffleDependency[K, V, V]])
    } else {
      // Otherwise, buffer map outputs in a deserialized form:
      new BaseShuffleHandle(shuffleId, numMaps, dependency)
    }
  }
  ```
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201019211555650.png#pic_center)

##### SortShuffleWriter代码解析
SortShuffleWriter使用ExternalSorter，write的方法接收的参数为Iterator[Product2[K, V]]，一个KEY、VALUE的集合，经过ExternalSorter排序之后，向
```java
/** Write a bunch of records to this task's output */
  override def write(records: Iterator[Product2[K, V]]): Unit = {
    sorter = if (dep.mapSideCombine) {
      require(dep.aggregator.isDefined, "Map-side combine without Aggregator specified!")
      new ExternalSorter[K, V, C](
        context, dep.aggregator, Some(dep.partitioner), dep.keyOrdering, dep.serializer)
    } else {
      // In this case we pass neither an aggregator nor an ordering to the sorter, because we don't
      // care whether the keys get sorted in each partition; that will be done on the reduce side
      // if the operation being run is sortByKey.
      new ExternalSorter[K, V, V](
        context, aggregator = None, Some(dep.partitioner), ordering = None, dep.serializer)
    }
    sorter.insertAll(records)

    // Don't bother including the time to open the merged output file in the shuffle write time,
    // because it just opens a single file, so is typically too fast to measure accurately
    // (see SPARK-3570).
    val output = shuffleBlockResolver.getDataFile(dep.shuffleId, mapId)
    val tmp = Utils.tempFileWith(output)
    try {
      val blockId = ShuffleBlockId(dep.shuffleId, mapId, IndexShuffleBlockResolver.NOOP_REDUCE_ID)
      val partitionLengths = sorter.writePartitionedFile(blockId, tmp)
      shuffleBlockResolver.writeIndexFileAndCommit(dep.shuffleId, mapId, partitionLengths, tmp)
      mapStatus = MapStatus(blockManager.shuffleServerId, partitionLengths)
    } finally {
      if (tmp.exists() && !tmp.delete()) {
        logError(s"Error while deleting temp file ${tmp.getAbsolutePath}")
      }
    }
  }
```
