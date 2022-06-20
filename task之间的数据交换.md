Flink中的数据交换是围绕以下设计原则构建的：
- 数据交换的控制流（即启动交换的控制流）是由接收方发起的，就像原始的MapReduce一样。
- 用于数据交换的数据流被抽象成为IntermediateResult，并且是可插拔的。这意味着该系统可以通过实现同一个接口来支持流数据传输和批处理数据传输。

数据交换涉及多个对象，包括：
- JobManager 
作为主节点的JobManager负责调度任务，恢复和协调，并通过ExecutionGraph数据结构掌握工作的概况。
- TaskManagers
  工作节点。 TaskManager（TM）在线程中同时执行许多任务。每个TM还包含一个CommunicationManager（CM-在任务之间共享）和一个MemoryManager（MM-在任务之间共享）。 TM可以通过需要时创建的固定TCP连接相互交换数据。

请注意，在Flink中，通过网络交换数据的是TaskManagers（而不是任务），即驻留在同一TM中的任务之间的数据交换是通过一个网络连接进行多路复用的。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210402165540129.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
ExecutionGraph：执行图是一个数据结构，其中包含有关作业计算的“基本事实”。 它由代表计算任务的顶点（ExecutionVertex）和代表任务产生的数据的中间结果（IntermediateResultPartition）组成。 顶点通过ExecutionEdges（EE）链接到它们消耗的中间结果：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210402174627195.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
These are logical data structures that live in the JobManager. They have their runtime equivalent structures that are responsible for the actual data processing that live at the TaskManagers. The runtime equivalent of the IntermediateResultPartition is called ResultPartition.

ResultPartition (RP) represents a chunk of data that a BufferWriter writes to, i.e., a chunk of data produced by a single task. A RP is a collection of Result Subpartitions (RSs). This is to distinguish between data that is destined to different receivers, e.g., in the case of a partitioning shuffle for a reduce or a join.

ResultSubpartition (RS) represents one partition of the data that is created by an operator, together with the logic for forwarding this data to the receiving operator. The specific implementation of a RS determines the actual data transfer logic, and this is the pluggable mechanism that allows the system to support a variety of data transfers. For example, the PipelinedSubpartition is a pipelined implementation to support streaming data exchange. The SpillableSubpartition is a blocking implementation to support batch data exchange.

InputGate: The logical equivalent of the RP at the receiving side. It is responsible for collecting buffers of data and handing them upstream.

InputChannel: The logical equivalent of the RS at the receiving side. It is responsible for collecting buffers of data for a specific partition.

这些是存在于JobManager中的逻辑数据结构。在TM上有与其等价的结构，负责实际的数据处理。与IntermediateResultPartition等价的运行时版本称为ResultPartition。

ResultPartition (RP)表示一个BufferWriter写入的数据块，即单个任务产生的数据块。RP是结果子分区(RSs)的集合。这是为了区分目的地不同接收器的数据，例如，在reduce或join的分区shuffle的情况下。

ResultSubpartition (RS)表示一个操作符创建的数据分区，以及将数据转发给接收操作符的逻辑。RS的具体实现决定了实际的数据传输逻辑，这就是可插拔机制，允许系统支持各种数据传输。例如，PipelinedSubpartition是一个支持流数据交换的管道实现。SpillableSubpartition是一个支持批处理数据交换的阻塞实现。

输入门:接收端RP的逻辑等价。它负责收集数据缓冲区并向上游处理它们。

InputChannel:接收端RS的逻辑等价。它负责收集特定分区的数据缓冲区。
