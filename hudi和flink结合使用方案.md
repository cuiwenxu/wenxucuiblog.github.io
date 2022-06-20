Apache Hudi(简称Hudi)是Uber创建的一个数据湖框架。2019年1月，Hudi加入Apache孵化器进行孵化，并于2020年5月晋升为Apache顶级项目。它是最流行的数据湖框架之一。

# 1. hudi与spark解耦
自诞生以来，Hudi一直使用Spark作为其数据处理引擎。如果用户想要使用Hudi作为他们的数据湖框架，他们必须将Spark引入到他们的平台技术栈中。几年前，使用Spark作为大数据处理引擎可以说是非常普遍甚至是自然的。因为Spark既可以执行批处理，也可以使用微批处理来模拟流，所以一个引擎可以同时解决流和批处理问题。然而，近年来，随着大数据技术的发展，Flink作为大数据处理引擎逐渐进入人们的视野，并在计算引擎领域占据了一定的市场。在大数据技术社区、论坛等领域，关于Hudi是否支持Flink的声音逐渐出现并变得越来越频繁。因此，让Hudi支持Flink引擎是很有价值的事情，而集成Flink引擎的第一步就是让Hudi和Spark解耦。

此外，看看大数据中成熟、活跃和可行的框架，所有框架在设计上都很优雅，可以与其他框架集成，并利用彼此的专业知识。因此，将Hudi与Spark脱钩，成为独立于引擎的数据湖框架，无疑将为Hudi与其他组件的整合创造更多可能性，使Hudi更好地融入大数据生态系统。

# 2. 挑战
Hudi内部对Spark API的使用就像我们平时开发和使用List一样普遍。由于数据源读取数据，最后将数据写到表中，所以到处都使用Spark RDD作为主要的数据结构，甚至普通的工具都是通过Spark API实现的。可以说，Hudi是Spark实现的一个通用数据湖框架。Hudi还利用深度Spark功能，如自定义分区、内存缓存，使用工作负载启发式实现索引和文件大小。对于其中的一些，Flink提供了更好的开箱即用支持(例如使用Flink的状态存储进行索引)，事实上，Hudi越来越接近实时延迟。

此外，在此分离后集成的主要引擎是Flink。Flink和Spark在核心抽象上有很大的不同。Spark认为数据是有限的，它的核心抽象是一组有限的数据。Flink认为数据的本质是流，其核心抽象的DataStream包含了对数据的各种操作。Hudi有一个流优先设计(记录级更新，记录级流)，这更适合Flink模型。同时，在Hudi中有多个RDD同时运行，一个RDD的处理结果与另一个RDD相结合。这种抽象上的差异和实现过程中中间结果的重用使得Hudi很难使用统一的API来操作RDD和DataStream。

# 3.解耦spark
从理论上讲，Hudi使用Spark作为计算引擎，利用Spark的分布式计算能力和RDD丰富的运算能力。除了分布式计算能力，Hudi更多地将RDD用作数据结构，而RDD本质上是一个有界的数据集。因此，从理论上讲，用List替换RDD是可行的(当然，它可能会牺牲性能/规模)。为了尽可能的保证Hudi Spark版本的性能和稳定性。我们可以继续将有界数据集设置为基本操作单元。Hudi的主要操作API保持不变，RDD被提取为泛型类型。Spark引擎实现仍然使用RDD，其他引擎根据实际情况使用List或其他绑定数据集。

## 解耦原则

- 1、统一的泛型。Spark API使用的输入记录`JavaRDD<hoodierecord>`，输入记录的键`JavaRDD< hoodierkey>`，写操作的结果`JavaRDD<writestatus>`，使用通用的`I,K,O`;

- 2、De-sparkization。抽象层的所有api必须与Spark无关。涉及抽象层中难以实现的具体操作，将其重写为抽象方法，并引入Spark子类。
例如:Hudi在很多地方使用`JavaSparkContext#map()`方法。要消除spark，您需要隐藏JavaSparkContext。针对这个问题，我们引入了`hoodieengineecontext #map()`方法，该方法将阻塞map的具体实现细节，从而在抽象中实现去spark化。

- 3、尽量减少抽象层的变化，保证Hudi原有的功能和性能;

- 4、将`JavaSparkContext`替换为`HoodieEngineContext`抽象类，以提供运行环境上下文。

此外，Hudi中的一些核心算法，如回滚，已经被重做，而不需要提前计算工作负载配置文件，这过去依赖于Spark缓存。

# 4. Flink集成设计
Hudi的写操作本质上是批处理，并且`DeltaStreamer`的连续模式是通过循环批处理来实现的。为了使用统一的API，当Hudi集成Flink时，我们选择先收集一批数据再进行处理，最后以统一的方式提交(这里我们在Flink中使用List收集数据)。

考虑批处理操作的最简单方法是使用时间窗口。然而，在使用窗口时，当没有数据在窗口中流动时，将没有输出数据，并且Flink sink很难判断给定批处理的所有数据是否已经被处理。因此，我们使用Flink的检查点机制来收集批。每两个屏障之间的数据是一个批处理。当子任务中没有数据时，就生成模拟结果数据。这样，在接收端，当每个子任务发出结果数据时，就可以认为已经处理了一批数据，可以执行提交。

DAG如下:
![在这里插入图片描述](https://img-blog.csdnimg.cn/4599819579c149718712738b0e54557f.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
- Source:接收Kafka数据并将其转换为`List<hoodierecord>`;
- InstantGeneratorOperator:生成一个全局唯一的瞬态。当前一个瞬态未完成或当前批处理没有数据时，不会创建新的瞬间;
- KeyBy partitionPath:根据partitionPath进行分区，避免多个子任务写同一个分区;
- WriteProcessOperator:执行写操作。当当前分区中没有数据时，向下游发送空的结果数据以补上编号;
- CommitSink:接收上游任务的计算结果。当接收到并行结果时，认为所有的上游子任务都已完成并执行了提交。

注意:InstantGeneratorOperator和WriteProcessOperator都是自定义的Flink操作符。InstantGeneratorOperator将阻止检查前一个瞬态的状态，以确保只有一个瞬态处于全局(或被请求)状态。WriteProcessOperator是实际执行。在执行写操作时，写操作在检查点触发。
4.1 Index design based on Flink State#
Stateful computing is one of the highlights of the Flink engine. Compared with using external storage, using Flink's built-in State can significantly improve the performance of Flink applications. Therefore, it would be a good choice to implement a Hudi index based on Flink's State.

The core of the Hudi index is to maintain the mapping of the Hudi key HoodieKey and the location of the Hudi data HoodieRecordLocation. Therefore, based on the current design, we can simply maintain a MapState<HoodieKey, HoodieRecordLocation> in Flink UDF to map the HoodieKey and HoodieRecordLocation, and leave the fault tolerance and persistence of State to the Flink framework.

## 4.1基于Flink State的index生成
状态计算是Flink引擎的亮点之一。与使用外部存储相比，使用Flink的内置State可以显著提高Flink应用程序的性能。因此，基于Flink的State实现一个Hudi索引将是一个很好的选择。

Hudi索引的核心是维护HoodieKey与Hudi数据HoodieRecordLocation的映射关系。因此，基于目前的设计，我们可以简单地在Flink UDF中维护一个`MapState<hoodiekey, hoodierecordlocation>`来映射HoodieKey和HoodieRecordLocation，并将State的容错和持久化留给Flink框架。
![在这里插入图片描述](https://img-blog.csdnimg.cn/ef83af9c8df7403b8079f29b9a4aa162.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
# 5. 接口举例
## HoodieTable
```java
/**
  * Abstract implementation of a HoodieTable.
  *
  * @param <T> Sub type of HoodieRecordPayload
  * @param <I> Type of inputs
  * @param <K> Type of keys
  * @param <O> Type of outputs
  */
public abstract class HoodieTable<T extends HoodieRecordPayload, I, K, O> implements Serializable {

   protected final HoodieWriteConfig config;
   protected final HoodieTableMetaClient metaClient;
   protected final HoodieIndex<T, I, K, O> index;

   public abstract HoodieWriteMetadata<O> upsert(HoodieEngineContext context, String instantTime,
       I records);

   public abstract HoodieWriteMetadata<O> insert(HoodieEngineContext context, String instantTime,
       I records);

   public abstract HoodieWriteMetadata<O> bulkInsert(HoodieEngineContext context, String instantTime,
       I records, Option<BulkInsertPartitioner<I>> bulkInsertPartitioner);

   ...
}
```
HoodieTable是Hudi的核心抽象之一，它定义了表所支持的insert、upsert、bulkInsert等操作。以upsert为例，输入数据由原来的`JavaRDD<hoodierecord> inputRdds`改为`I records`，运行时`JavaSparkContext jsc`改为`hoodieengineecontext`上下文。

从类注释中可以看出，T、I、K、O分别表示Hudi操作的加载数据类型、输入数据类型、主键类型和输出数据类型。这些泛型将贯穿整个抽象层。
## HoodieEngineContext
```java
/**
 * Base class contains the context information needed by the engine at runtime. It will be extended by different
 * engine implementation if needed.
 */
public abstract class HoodieEngineContext {

  public abstract <I, O> List<O> map(List<I> data, SerializableFunction<I, O> func, int parallelism);

  public abstract <I, O> List<O> flatMap(List<I> data, SerializableFunction<I, Stream<O>> func, int parallelism);

  public abstract <I> void foreach(List<I> data, SerializableConsumer<I> consumer, int parallelism);

  ......
}
```
HoodieEngineContext扮演了JavaSparkContext的作用,它不仅提供了JavaSparkContext能提供的所有信息,但也封装了许多方法,如map、flatMap、foreach。

以map方法为例。在Spark实现类HoodieSparkEngineContext中，map方法如下:
```java
@Override
  public <I, O> List<O> map(List<I> data, SerializableFunction<I, O> func, int parallelism) {
    return javaSparkContext.parallelize(data, parallelism).map(func::apply).collect();
  }
```
在HoodieEngineContext引擎中，实现可以如下(不同的方法需要注意线程安全问题，使用parallel()时要谨慎):
```java
@Override
  public <I, O> List<O> map(List<I> data, SerializableFunction<I, O> func, int parallelism) {
    return data.stream().parallel().map(func::apply).collect(Collectors.toList());
  }
```
