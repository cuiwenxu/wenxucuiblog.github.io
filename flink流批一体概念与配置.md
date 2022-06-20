@[TOC](flink流批一体概念与配置)
# 什么时候可以/应该使用批处理执行模式?
BATCH执行模式只能用于有界的数据源。有界性是数据源的一个属性，它告诉我们来自该数据源的所有输入在执行之前是否已知，或者是否会无限期地出现新数据。反过来，如果一个作业的所有源都是有界的，那么它就是有界的，否则就是无界的。

`另一方面，流执行模式既可以用于有边界作业，也可以用于无边界作业。`

根据经验，当程序有边界时，应该使用BATCH执行模式，因为这样效率更高。当您的程序是无界的时，您必须使用流执行模式，因为只有这种模式足够通用，能够处理连续的数据流。
# 配置批处理执行模式
执行模式可以通过execute .runtime-mode设置进行配置。有三种可能的值:

- STREAMING:经典的DataStream执行模式(默认)
- BATCH:DataStream API上的批处理式执行
- AUTOMATIC:让系统根据源的有界性来决定

可以通过bin/flink run…的命令行参数进行配置，或在创建/配置StreamExecutionEnvironment时以编程方式。

下面是如何通过命令行配置执行模式:

>$ bin/flink run - execution.runtime-mode=BATCH examples/streaming/WordCount.jar

下面这个例子展示了如何在代码中配置执行模式:

>StreamExecutionEnvironment env = StreamExecutionEnvironment. getExecutionEnvironment ();
env.setRuntimeMode (RuntimeExecutionMode.BATCH);

我们建议用户不要在他们的程序中设置运行时模式，而是在提交应用程序时使用命令行设置。可以提供更大的灵活性，因为相同的应用程序可以在任何执行模式下执行。

# 流批执行模式的区别
本节概述BATCH执行模式的执行行为，并将其与STREAMING执行模式进行对比。更多的细节，请参考介绍该特性的flip: FLIP-134和FLIP-140。

## 任务调度和网络Shuffle 
Flink作业由不同的操作组成，这些操作在数据流图中连接在一起。系统决定如何在不同的进程/机器(TaskManagers)上调度这些操作的执行，以及数据如何在它们之间被shuffle(send)。

多个operator可以使用一个称为chain的特性链接在一起。Flink将一个或多个operator(chain在一起)视为调度单元的一组称为task。术语subtask通常用于指在多个taskmanager上并行运行的任务的单个实例，但这里我们只使用术语task。

批处理和流执行在任务调度和网络shuffle的工作方式不同。这主要是因为BATCH执行模式下，Flink会使用更有效的数据结构和算法。

我们将用这个例子来解释任务调度和网络传输的区别:
```java
StreamExecutionEnvironment env = StreamExecutionEnvironment. getexecutionenvironment ();

DataStreamSource<string> source = env.fromElements(…);</string>

source.name(“source”)
. map(…). name(“map1”)
. map(…). name (map2)
.rebalance ()
. map(…). name(“map3”)
. map(…). name(“map4”)
.keyBy((value)->value)
. map(…). name(“map5”)
. map(…). name(“map6”)
.sinkTo(…). name(“sink”);
```
一对一连接的operator，如map()、filter()，可以直接将数据转发到下一个operator，这允许将这些操作链接在一起。这意味着Flink通常不会在它们之间插入网络shuffle。

另一方面，keyBy()或rebalance()等操作要求数据在不同的并行任务实例之间被打乱。这就产生了网络shuffle。

在上面的例子中，Flink会像这样将操作分组为任务:

- Task1: source, map1, map2

- Task2: map3 map4

- Task3: map5, map6, sink

我们在任务1和任务2之间有一个网络shuffle，任务2和任务3也一样。这是该工作的视觉表现:

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210701131419866.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
### 流执行模式
在流执行模式下，所有任务需要一直在线/运行。这允许Flink通过整个管道立即处理新记录，这是连续和低延迟流处理所需要的。这也意味着分配给作业的TaskManagers需要有足够的资源来同时运行所有任务。

网络shuffle是流水线的，这意味着记录立即被发送到下游任务，网络层有一些缓冲。同样，这是必需的，因为在处理连续的数据流时，不存在可以在任务(或pipeline)之间物化数据的时间点。这与BATCH执行模式不同，在BATCH执行模式中，中间结果可以物化，如下所述。

### 批量执行模式
在BATCH执行模式中，作业的任务可以被划分为多个阶段，一个阶段接着一个阶段执行。我们可以这样做是因为输入是有界的，因此Flink可以在进入下一个阶段之前完全处理管道的一个阶段。在上面的示例中，作业将有三个阶段，分别对应由shuffle barrier分隔的三个任务。

不像前面流模式中解释的那样将记录立即发送给下游任务，分阶段处理需要Flink将任务的中间结果物化到一些非临时存储中，这样在上游任务已经离线后，下游任务才能读取这些数据。这将增加处理的延迟，但会带来其他有趣的属性。首先，这允许Flink在出现故障时回溯到最新的可用结果，而不是重新启动整个作业。另一个副作用是BATCH作业可以在更少的资源上执行(就TaskManagers的可用插槽而言)，因为系统可以一个接一个顺序执行任务。

taskmanager将保持中间结果，至少只要下游任务没有使用它们。(从技术上讲，它们将一直保存到使用流水线的区域产生它们的输出。)在此之后，它们将在空间允许的情况下被保留，以便在出现故障时能够回溯到前面提到的早期结果。

## 状态后端/状态
在STREAMING模式下，Flink使用state backend来控制状态的存储方式和检查点的工作方式。

在BATCH模式中，配置的后端状态将被忽略。相反，keyed operator的输入按key分组(使用排序)，然后依次处理键的所有记录。这允许同时只保留一个键的状态。当移动到下一个键时，给定键的状态将被丢弃。

有关这方面的背景信息，请参阅flip 140。

## 处理顺序
BATCH和STREAMING在operator或用户定义函数(udf)中处理记录的顺序之间有所不同。

在STREAMING模式下，用户定义函数不应该对传入记录的顺序做任何假设。数据一到就被处理。

在BATCH执行模式中，有一些操作Flink是保证顺序的。

我们可以区分三种类型的输入:

- 广播输入:来自广播流的输入(参见广播状态)
- 常规输入:既不是广播也不是键控的输入
- keyyed input:来自KeyedStream的输入

有多种输入类型的函数或operator将按照以下顺序处理它们:

- 首先处理广播输入
- 常规输入是第二步处理的
- 最后处理键控输入

对于使用多个常规或广播输入的函数(例如CoProcessFunction)， Flink有权以任何顺序处理来自该类型的任何输入的数据。

对于使用多个键输入的函数(例如KeyedCoProcessFunction)， Flink会处理来自所有键输入的单个键的所有记录，然后再转移到下一个键输入。

## 事件时间/水印
当涉及到支持事件时间时，Flink的流运行时建立在悲观的假设上，即事件可能会无序，即时间戳为t的事件可能在时间戳为t+1的事件之后。正因为如此，系统永远不能确保将来不再有对于给定时间戳t具有时间戳t < T的元素。为了平摊这种无序对最终结果的影响，同时使系统实用，在STREAMING模式下，Flink使用了一种叫做Watermarks的启发式方法。带有时间戳T的水印表示没有带有时间t < T的元素会跟随。

在BATCH模式中，输入数据集是预先知道的，不需要这样的启发式方法，至少可以按时间戳对元素排序，以便按时间顺序处理它们。即BATCH中我们可以假设存在“完美的水印”。

如上所述，在批处理模式中，我们只需要在与每个键相关联的输入的末尾添加MAX_WATERMARK，或者如果输入流没有键入关键字，则在输入的末尾添加MAX_WATERMARK。基于该方案，所有注册的计时器将在时间结束时触发，用户定义的WatermarkAssigners或WatermarkGenerators将被忽略。但是，指定一个WatermarkStrategy仍然很重要，因为它的TimestampAssigner仍将用于为记录分配时间戳。

## 处理时间
处理时间是在处理记录的特定实例上，处理记录的机器上的时钟时间。根据这个定义，我们可以看到基于处理时间的计算结果是不可重复的。这是因为处理两次的同一记录将具有两个不同的时间戳。

尽管如此，在STREAMING模式中使用处理时间还是很有用的。原因在于，流管道经常实时地摄取无边界的输入，因此事件时间和处理时间之间存在相关性。此外，由于上述原因，在STREAMING模式下，事件时间的1h通常可以接近处理时间的1h，或wall-clock时间。因此，使用处理时间可以用于窗口计算的触发(如ContinuousProcessingTimeTrigger)。

这种相关性在批处理中不存在，因为在批处理中，输入数据集是静态的，并且是预先知道的。鉴于此，在BATCH模式中，我们允许用户请求当前处理时间并注册处理时间计时器，但是，与Event time的情况一样，所有计时器都将在输入结束时触发。
## 故障恢复
在流执行模式中，Flink使用检查点进行故障恢复。故障恢复检查点的一个特征是，在出现故障时，Flink将从检查点重新启动所有正在运行的任务。这可能比我们必须在BATCH模式下执行的操作(如下所述)代价更高，这也是您应该在作业允许时使用BATCH执行模式的原因之一。

在BATCH执行模式中，Flink将尝试并回溯到之前的中间结果仍然可用的处理阶段。可能只有失败的任务(或jobgraph中它的前置节点)需要重新启动，与从一个检查点重新启动所有任务相比，这可以提高处理效率和作业的总体处理时间。

# 需要注意的点
与传统的流执行模式相比，在批处理模式中有些事情可能不会像预期的那样工作。有些特性的工作方式略有不同，而其他特性则不受支持。

BATCH模式下的行为改变:
- 像reduce()或sum()这样的“滚动”操作会对每条以STREAMING模式到达的新记录发出一个增量更新。在BATCH模式中，这些操作不是“滚动”的。它们只释放最终的结果。

BATCH模式下不支持:

- 检查点和任何依赖于检查点的操作都不能工作。
- 迭代

应该小心地实现自定义操作符，否则它们可能会出现不当行为。有关更多细节，请参见下面的其他解释。

## 检查点
如上所述，批处理程序的故障恢复不使用检查点。

重要的是要记住，因为没有检查点，某些特性，如CheckpointListener，因此，Kafka的EXACTLY_ONCE模式或StreamingFileSink的OnCheckpointRollingPolicy不会工作。如果您需要一个以BATCH模式工作的事务接收器，请确保它使用flip143中提议的统一接收器API。

您仍然可以使用所有的状态原语，只是用于故障恢复的机制将有所不同。
## 编写自定义操作符
在编写自定义操作符时，一定要记住BATCH执行模式的假设。否则，在STREAMING模式下正常工作的操作符可能会在BATCH模式下产生错误的结果。操作符从来没有限定到特定的键，这意味着它们看到了Flink试图利用的BATCH处理的一些属性。

首先，您不应该缓存操作符中最后看到的水印。在BATCH模式中，我们逐个键处理记录。因此，水印将在每个键之间从MAX_VALUE切换到MIN_VALUE。您不应该假定水印在运算符中总是升序的。出于同样的原因，计时器将首先按键顺序触发，然后在每个键内按时间戳顺序触发。此外，不支持手动更改密钥的操作。
