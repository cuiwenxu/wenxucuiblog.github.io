罗西的思考
一手伸向技术，一手伸向生活
博客园首页新随笔联系订阅管理随笔 - 47  文章 - 0  评论 - 4
[源码分析] 从源码入手看 Flink Watermark 之传播过程
[源码分析] 从源码入手看 Flink Watermark 之传播过程
0x00 摘要
本文将通过源码分析，带领大家熟悉Flink Watermark 之传播过程，顺便也可以对Flink整体逻辑有一个大致把握。

0x01 总述
从静态角度讲，watermarks是实现流式计算的核心概念；从动态角度说，watermarks贯穿整个流处理程序。所以为了讲解watermarks的传播，需要对flink的很多模块/概念进行了解，涉及几乎各个阶段。我首先会讲解相关概念，然后会根据一个实例代码从以下几部分来解释：程序逻辑/计算图模型/程序执行。最后是详细Flink源码分析（略冗长，可以选择性阅读）。

0x02 相关概念
流计算被抽象成四个问题，what，where，when，how。

window解决的是where，也就是将无界数据划分成有界数据。

window的数据何时被计算是when？解决这个问题用的方式是watermark和trigger，watermark用来标记窗口的完整性。trigger用来设计窗口数据触发条件。

1. 乱序处理
乱序问题一般是和event time关联的， 对于一个流式处理系统的process time来说，是不存在乱序问题的。所以下面介绍的watermark/allowedLateness也只是在event time作为主时间才生效。

Flink中处理乱序依赖的 watermark+window+trigger，属于全局性的处理；Flink同时对于window而言，还提供了allowedLateness方法，使得更大限度的允许乱序，属于局部性的处理；

即watermark是全局的，不止针对window计算，而allowedLateness让某一个特定window函数能自己控制处理延迟数据的策略，allowedLateness是窗口函数的属性。

2. Watermark(水位线)
watermark是流式系统中主要用于解决流式系统中数据乱序问题的机制，方法是用于标记当前处理到什么水位的数据了，这意味着再早于这个水位的数据过来会被直接丢弃。这使得引擎可以自动跟踪数据中的当前事件时间，并尝试相应地清除旧状态。

Watermarking表示多长时间以前的数据将不再更新，您可以通过指定事件时间列来定义查询的Watermarking，并根据事件时间预测数据的延迟时间。也就是说每次窗口滑动之前会进行Watermarking的计算。当一组数据或新接收的数据事件时间小于Watermarking时，则该数据不会更新，在内存中就不会维护该组数据的状态。

换一种说法，阈值内的滞后数据将被聚合，但是晚于阈值到来的数据(其实际时间比watermark小)将被丢弃。

watermark和数据本身一样作为正常的消息在流中流动。

3. Trigger
Trigger 指明在哪些条件下触发window计算，基于处理数据时的时间以及事件的特定属性。一般trigger的实现是当watermark处于某种时间条件下或者窗口数据达到一定条件，窗口的数据开始计算。

每个窗口分配器都会有一个默认的Trigger。如果默认的Trigger不能满足你的需求，你可以指定一个自定义的trigger()。Flink Trigger接口有如下方法允许trigger对不同的事件做出反应：

* onElement():进入窗口的每个元素都会调用该方法。
* onEventTime():事件时间timer触发的时候被调用。
* onProcessingTime():处理时间timer触发的时候会被调用。
* onMerge():有状态的触发器相关，并在它们相应的窗口合并时合并两个触发器的状态，例如使用会话窗口。
* clear():该方法主要是执行窗口的删除操作。
每次trigger，都是要对新增的数据，相关的window进行重新计算，并输出。输出有complete, append,update三种输出模式：

Complete mode：Result Table 全量输出，也就是重新计算过的window结果都输出。意味着这种模式下，每次读了新增的input数据，output的时候会把内存中resulttable中所有window的结果都输出一遍。

Append mode (default)：只有 Result Table 中新增的行才会被输出，所谓新增是指自上一次 trigger 的时候。因为只是输出新增的行，所以如果老数据有改动就不适合使用这种模式。 更新的window并不输出，否则外存里的key就重了。

Update mode：只要更新的 Row 都会被输出，相当于 Append mode 的加强版。而且是对外存中的相同key进行update，而不是append，需要外存是能kv操作的！只会输出新增和更新过的window的结果。

从上面能看出来，流式框架对于window的结果数据是存在一个 result table里的！

4. allowedLateness
Flink中借助watermark以及window和trigger来处理基于event time的乱序问题，那么如何处理“late element”呢？

也许还有人会问，out-of-order element与late element有什么区别？不都是一回事么？答案是一回事，都是为了处理乱序问题而产生的概念。要说区别，可以总结如下：

通过watermark机制来处理out-of-order的问题，属于第一层防护，属于全局性的防护，通常说的乱序问题的解决办法，就是指这类；
通过窗口上的allowedLateness机制来处理out-of-order的问题，属于第二层防护，属于特定window operator的防护，late element的问题就是指这类。
默认情况下，当watermark通过end-of-window之后，再有之前的数据到达时，这些数据会被删除。为了避免有些迟到的数据被删除，因此产生了allowedLateness的概念。

简单来讲，allowedLateness就是针对event time而言，对于watermark超过end-of-window之后，还允许有一段时间（也是以event time来衡量）来等待之前的数据到达，以便再次处理这些数据。

5. 处理消息过程
windowoperator接到消息以后，首先存到state，存放的格式为k,v，key的格式是key + window，value是key和window对应的数据。
注册一个timer，timer的数据结构为 [key，window，window边界 - 1]，将timer放到集合中去。
当windowoperator收到watermark以后，取出集合中小于watermark的timer，触发其window。触发的过程中将state里面对应key及window的数据取出来，这里要经过序列化的过程，发送给windowfunction计算。
数据发送给windowfunction，实现windowfunction的window数据计算逻辑。
比如某窗口有三个数据：[key A, window A, 0], [key A, window A, 4999], [key A, window A, 5000]

对于固定窗口，当第一个watermark (Watermark 5000)到达时候，[key A, window A, 0], [key A, window A, 4999] 会被计算，当第二个watermark (Watermark 9999)到达时候，[key A, window A, 5000]会被计算。

6. 累加(再次)计算
watermark是全局性的参数，用于管理消息的乱序，watermark超过window的endtime之后，就会触发窗口计算。一般情况下，触发窗口计算之后，窗口就销毁掉了，后面再来的数据也不会再计算。

因为加入了allowedLateness，所以计算会和之前不同了。window这个allowedLateness属性，默认为0，如果allowedLateness > 0，那么在某一个特定watermark到来之前，这个触发过计算的窗口还会继续保留，这个保留主要是窗口里的消息。

这个特定的watermark是什么呢？ watermark-allowedLateness>=窗口endtime。这个特定watermark来了之后，窗口就要消失了，后面再来属于这个窗口的消息，就丢掉了。在 "watermark（=窗口endtime）" ～ “watermark（=endtime+allowedLateness）" 这段时间之间，对应窗口可能会多次计算。那么要window的endtime+allowedLateness <= watermark的时候，window才会被清掉。

比如window的endtime是5000，allowedLateness=0，那么如果watermark 5000到来之后，这个window就应该被清除。但是如果allowedLateness = 1000，则需要等water 6000(endtime + allowedLateness)到来之后，这个window才会被清掉。

Flink的allowedLateness可用于TumblingEventTimeWindow、SlidingEventTimeWindow以及EventTimeSessionWindows，这可能使得窗口再次被触发，相当于对前一次窗口的窗口的修正（累加计算或者累加撤回计算）；

注意：对于trigger是默认的EventTimeTrigger的情况下，allowedLateness会再次触发窗口的计算，而之前触发的数据，会buffer起来，直到watermark超过end-of-window + allowedLateness的时间，窗口的数据及元数据信息才会被删除。再次计算就是DataFlow模型中的Accumulating的情况。

同时，对于sessionWindow的情况，当late element在allowedLateness范围之内到达时，可能会引起窗口的merge，这样之前窗口的数据会在新窗口中累加计算，这就是DataFlow模型中的AccumulatingAndRetracting的情况。

7. Watermark传播
生产任务的pipeline中通常有多个stage，在源头产生的watermark会在pipeline的多个stage间传递。了解watermark如何在一个pipeline的多个stage间进行传递，可以更好的了解watermark对整个pipeline的影响，以及对pipeline结果延时的影响。我们在pipeline的各stage的边界上对watermark做如下定义：

输入watermark（An input watermark）：捕捉上游各阶段数据处理进度。对源头算子，input watermark是个特殊的function，对进入的数据产生watermark。对非源头算子，input watermark是上游stage中，所有shard/partition/instance产生的最小的watermark
输出watermark（An output watermark）：捕捉本stage的数据进度，实质上指本stage中，所有input watermark的最小值，和本stage中所有非late event的数据的event time。比如，该stage中，被缓存起来等待做聚合的数据等。
每个stage内的操作并不是线性递增的。概念上，每个stage的操作都可以被分为几个组件（components），每个组件都会影响pipeline的输出watermark。每个组件的特性与具体的实现方式和包含的算子相关。理论上，这类算子会缓存数据，直到触发某个计算。比如缓存一部分数据并将其存入状态（state）中，直到触发聚合计算，并将计算结果写入下游stage。

watermark可以是以下项的最小值：

每个source的watermark（Per-source watermark） - 每个发送数据的stage.
每个外部数据源的watermark（Per-external input watermark） - pipeline之外的数据源
每个状态组件的watermark（Per-state component watermark） - 每种需要写入的state类型
每个输出buffer的watermark（Per-output buffer watermark） - 每个接收stage
这种精度的watermark能够更好的描述系统内部状态。能够更简单的跟踪数据在系统各个buffer中的流转状态，有助于排查数据堵塞问题。

watermark以广播的形式在算子之间传播，当一个算子收到watermark时都要干些什么事情呢？

更新算子时间
遍历计时器队列触发回调
将watermark发送到下游
0x03. Flink 程序结构 & 核心概念
1. 程序结构
Flink程序像常规的程序一样对数据集合进行转换操作，每个程序由下面几部分组成：

获取一个执行环境
加载/创建初始化数据
指定对于数据的transformations操作
指定计算的输出结果（打印或者输出到文件）
触发程序执行
flink流式计算的核心概念，就是将数据从输入流一个个传递给Operator进行链式处理，最后交给输出流的过程。对数据的每一次处理在逻辑上成为一个operator，并且为了本地化处理的效率起见，operator之间也可以串成一个chain一起处理。

下面这张图表明了flink是如何看待用户的处理流程的：用户操作被抽象化为一系列operator。以source开始，以sink结尾，中间的operator做的操作叫做transform，并且可以把几个操作串在一起执行。

Source ---> Transformation ----> Transformation ----> Sink
以下是一个样例代码，后续的分析会基于此代码。

DataStream<String> text = env.socketTextStream(hostname, port);

DataStream counts = text
    .filter(new FilterClass())
    .map(new LineSplitter())
    .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessGenerator) 
    .keyBy(0)
    .timeWindow(Time.seconds(10))
    .sum(2)
  
counts.print()  
System.out.println(env.getExecutionPlan());
2. 核心类/接口
在用户设计程序时候，对应如下核心类/接口：

DataStream：描述的是一个具有相同数据类型的数据流，底层是通过具体的Transformation来实现，其负责提供各种对流上的数据进行操作转换的API接口。
Transformation：描述了构建一个DataStream的操作，以及该操作的并行度、输出数据类型等信息，并有一个属性，用来持有StreamOperator的一个具体实例；
上述代码逻辑中，对数据流做了如下操作：filter, map, keyBy, assignTimestampsAndWatermarks, timeWindow, sum。每次转换都生成了一个新的DataStream。

比如实例代码中的timeWindow最后生成了windowedStream。windowedStream之上执行的apply方法会生成了WindowOperator，初始化时包含了trigger以及allowedLateness的值。然后经过transform转换，实际上是执行了DataStream中的transform方法，最后生成了SingleOutputStreamOperator。SingleOutputStreamOperator这个类名字有点误导，实际上它是DataStream的子类。

    public <R> SingleOutputStreamOperator<R> apply(WindowFunction<T, R, K, W> function, TypeInformation<R> resultType) {
        KeySelector<T, K> keySel = input.getKeySelector(); //根据keyedStream获取key
        WindowOperator<K, T, Iterable<T>, R, W> operator;
        operator =  new WindowOperator<>(windowAssigner, ... ,
                    		new InternalIterableWindowFunction<>(function),
                   		  trigger,
                   		  allowedLateness,
                    	  legacyWindowOpType);
        return input.transform(opName, resultType, operator);//根据operator name，窗口函数的类型，以及window operator，执行keyedStream.transaform操作
    }
0x04. Flink 执行图模型
Flink 中的执行图可以分成四层：StreamGraph ---> JobGraph ---> ExecutionGraph -> 物理执行图。

StreamGraph：是对用户逻辑的映射，代表程序的拓扑结构，是根据用户通过 Stream API 编写的代码生成的最初的图。
JobGraph：StreamGraph经过优化后生成了 JobGraph，提交给 JobManager 的数据结构。主要的优化为，将多个符合条件的节点 chain 在一起作为一个节点，这样可以减少数据在节点之间流动所需要的序列化/反序列化/传输消耗。
ExecutionGraph：JobManager 根据 JobGraph 生成ExecutionGraph。ExecutionGraph是JobGraph的并行化版本，是调度层最核心的数据结构。
物理执行图：JobManager 根据 ExecutionGraph 对 Job 进行调度后，在各个TaskManager 上部署 Task 后形成的“图”，并不是一个具体的数据结构。
我们这里重点看StreamGraph，其相关重点数据结构是：

StreamNode 是用来描述 operator 的逻辑节点，并具有所有相关的属性，如并发度、入边和出边等。
StreamEdge 是用来描述两个 StreamNode（operator） 逻辑的链接边。
我们可以直接打印 Execution Plan

System.out.println(env.getExecutionPlan());
其内部调用 StreamExecutionEnvironment.getExecutionPlan 得到 StreamGraph。

public String getExecutionPlan() {
		return getStreamGraph(DEFAULT_JOB_NAME, false).getStreamingPlanAsJSON();
}
StreamGraph的转换流是：

* Source --> Filter --> Map --> Timestamps/Watermarks --> Window(SumAggregator) --> Sink
下面是我把 示例代码 打印StreamGraph结果整理出来一个静态架构。可以看出代码中的转换被翻译成了如下执行Unit（在下面图中，其执行序列是由上而下）。

*        +-----> Data Source(ID = 1) [ Source Socket Stream ]  
*        |      // env.socketTextStream(hostname, port) 方法中生成了一个 Data Source
*        |      
*        +-----> Operator(ID = 2) [ Filter ]
*        | 
*        |      
*        +-----> Operator(ID = 3) [ Map ]
*        | 
*        |      
*        +-----> Operator(ID = 4) [ Timestamps/Watermarks ]
*        | 
*        |      
*        +-----> Operator(ID = 6) [ Window(SumAggregator) ]
*        |       // 多个Operator被构建成 Operator Chain
*        | 
*        |      
*        +-----> Data Sink(ID = 7) [ Sink : Print to Std. Out ] 
*                // counts.print() 是在数据流最后添加了个 Data Sink，用于承接统计结果   
示例代码中，Flink生成StreamGraph的大致处理流程是：

首先处理的Source，生成了Source的StreamNode。
处理Filter，生成了Filter的StreamNode，并生成StreamEdge连接上游Source和Filter。
处理Map，生成了Map的StreamNode，并生成StreamEdge连接上游Filter和Map。
处理assignTimestampsAndWatermarks，生成了Timestamps/Watermarks的StreamNode，并生成StreamEdge连接上游Map和Timestamps/Watermarks。
处理keyBy/timeWindow/sum，生成了Window的StreamNode 以及 Operator Chain，并生成StreamEdge连接上游Timestamps/Watermarks和Window。
最后处理Sink，创建Sink的StreamNode，并生成StreamEdge与上游Window相连。
0x05. 执行模块生命周期
这里主要核心类是：

Function：用户通过继承该接口的不同子类来实现用户自己的数据处理逻辑。如子类SocketTextStreamFunction实现从指定hostname和port来接收数据，并转发字符串的逻辑；

Task: 是Flink中执行的基本单位，代表一个 TaskManager 中所起的并行子任务，执行封装的 flink 算子并运行，提供以下服务：消费输入data、生产 IntermediateResultPartition [ flink关于中间结果的抽象 ]、与 JobManager 交互。

StreamTask : 是本地执行的基本单位，由TaskManagers部署执行。包含了多个StreamOperator，封装了算子的处理逻辑。

StreamOperator：DataStream 上的每一个 Transformation 都对应了一个 StreamOperator，StreamOperator是运行时的具体实现，会决定UDF(User-Defined Funtion)的调用方式。

StreamSource 是StreamOperator接口的一个具体实现类，其构造函数入参就是SourceFunction的子类，这里就是SocketTextStreamFunction的实例。

Task 是直接受 TaskManager 管理和调度的，而 Task 又会调用 StreamTask(主要是其各种子类)，StreamTask 中封装了算子(StreamOperator)的处理逻辑。StreamSource是用来开启整个流的算子。我们接下来就说说动态逻辑。

我们的示例代码中，所有程序逻辑都是运行在StreamTask(主要是其各种子类)中，filter/map对应了StreamOperator；assignTimestampsAndWatermarks用来生成Watermarks，传递给下游的.keyBy.timeWindow(WindowOperator)。而keyBy/timeWindow/sum又被构建成OperatorChain。所以我们下面就逐一讲解这些概念。

1. Task
Task，它是在线程中执行的Runable对象，每个Task都是由一组Operators Chaining在一起的工作集合，Flink Job的执行过程可看作一张DAG图，Task是DAG图上的顶点（Vertex），顶点之间通过数据传递方式相互链接构成整个Job的Execution Graph。

Task 是直接受 TaskManager 管理和调度的，Flink最后通过RPC方法提交task，实际会调用到TaskExecutor.submitTask方法中。这个方法会创建真正的Task，然后调用task.startTaskThread();开始task的执行。而startTaskThread方法，则会执行executingThread.start，从而调用Task.run方法。
它的最核心的代码如下：

 * public class Task implements Runnable...
 * The Task represents one execution of a parallel subtask on a TaskManager.
 * A Task wraps a Flink operator (which may be a user function) and runs it
 *
 *  -- doRun()
 *        |
 *        +----> 从 NetworkEnvironment 中申请 BufferPool
 *        |      包括 InputGate 的接收 pool 以及 task 的每个 ResultPartition 的输出 pool
 *        +----> invokable = loadAndInstantiateInvokable(userCodeClassLoader, 
 *        |                  nameOfInvokableClass) 通过反射创建
 *        |      load and instantiate the task's invokable code
 *        |      invokable即为operator对象实例，例如OneInputStreamTask，SourceStreamTask等
 *        |      OneInputStreamTask继承了StreamTask，这里实际调用的invoke()方法是StreamTask里的
 *        +----> invokable.invoke()
 *        |      run the invokable, 
 *        |      
 *        |        
 * OneInputStreamTask<IN,OUT> extends StreamTask<OUT,OneInputStreamOperator<IN, OUT>>    
这个nameOfInvokableClass是哪里生成的呢？其实早在生成StreamGraph的时候，这就已经确定了，见StreamGraph.addOperator方法

        if (operatorObject instanceof StoppableStreamSource) {
            addNode(vertexID, slotSharingGroup, StoppableSourceStreamTask.class, operatorObject, operatorName);
        } else if (operatorObject instanceof StreamSource) {
            addNode(vertexID, slotSharingGroup, SourceStreamTask.class, operatorObject, operatorName);
        } else {
            addNode(vertexID, slotSharingGroup, OneInputStreamTask.class, operatorObject, operatorName);
        }
这里的OneInputStreamTask.class即为生成的StreamNode的vertexClass。这个值会一直传递

StreamGraph --> JobVertex.invokableClass --> ExecutionJobVertex.TaskInformation.invokableClassName --> Task
2. StreamTask
是本地执行的基本单位，由TaskManagers部署执行，Task会调用 StreamTask。StreamTask包含了headOperator 和 operatorChain，封装了算子的处理逻辑。可以理解为，StreamTask是执行流程框架，OperatorChain(StreamOperator)是负责具体算子逻辑，嵌入到StreamTask的执行流程框架中。

直接从StreamTask的注释中，能看到StreamTask的生命周期。

其中，每个operator的open()方法都被StreamTask的openAllOperators()方法调用。该方法(指openAllOperators)执行所有的operational的初始化，例如使用定时器服务注册定时器。单个task可能正在执行多个operator，消耗其前驱的输出，在这种情况下，该open()方法在最后一个operator中调用，即这个operator的输出也是task本身的输出。这样做使得当第一个operator开始处理任务的输入时，它的所有下游operator都准备好接收其输出。

OperatorChain是在StreamTask的invoke方法中被创建的，在执行的时候，如果一个operator无法被chain起来，那它就只有headOperator，chain里就没有其他operator了。

注意： task中的连续operator是从最后到第一个依次open。

以OneInputStreamTask为例，Task的核心执行代码即为OneInputStreamTask.invoke方法，它会调用StreamTask.invoke方法。

 * The life cycle of the task(StreamTask) is set up as follows:
 * {@code
 *  -- setInitialState -> provides state of all operators in the chain
 *        |   
 *        +----> 重新初始化task的state，并且在如下两种情况下尤为重要：
 *        |      1. 当任务从故障中恢复并从最后一个成功的checkpoint点重新启动时
 *        |      2. 从一个保存点恢复时。
 *  -- invoke()
 *        |
 *        +----> Create basic utils (config, etc) and load the chain of operators
 *        +----> operators.setup() //创建 operatorChain 并设置为 headOperator 的 Output
 *        --------> openAllOperators()
 *        +----> task specific init()
 *        +----> initialize-operator-states()
 *        +----> open-operators() //执行 operatorChain 中所有 operator 的 open 方法
 *        +----> run() //runMailboxLoop()方法将一直运行，直到没有更多的输入数据
 *        --------> mailboxProcessor.runMailboxLoop();
 *        --------> StreamTask.processInput()
 *        --------> StreamTask.inputProcessor.processInput()   
 *        --------> 间接调用 operator的processElement()和processWatermark()方法
 *        +----> close-operators() //执行 operatorChain 中所有 operator 的 close 方法
 *        +----> dispose-operators()
 *        +----> common cleanup
 *        +----> task specific cleanup()
 * }
3. OneInputStreamTask
OneInputStreamTask是 StreamTask 的实现类之一，具有代表性。我们示例代码中基本都是由OneInputStreamTask来做具体执行。

看看OneInputStreamTask 是如何生成的?

 * 生成StreamNode时候
 *
 *  -- StreamGraph.addOperator()
 *        |   
 *        +----> addNode(... OneInputStreamTask.class, operatorObject, operatorName);
 *        |      将 OneInputStreamTask 等 StreamTask 设置到 StreamNode 的节点属性中
 *     
 *  
 * 在 JobVertex 的节点构造时也会做一次初始化
 *        |      
 *        +----> jobVertex.setInvokableClass(streamNode.getJobVertexClass());   
后续在 TaskDeploymentDescriptor 实例化的时候会获取 jobVertex 中的属性。

再看看OneInputStreamTask 的 init() 和run() 分别都做了什么

 * OneInputStreamTask
 * class OneInputStreamTask<IN,OUT> extends StreamTask<OUT,OneInputStreamOperator<IN, OUT>>  * {@code
 *  -- init方法
 *        |
 *        +----> 获取算子对应的输入序列化器 TypeSerializer
 *        +----> CheckpointedInputGate inputGate = createCheckpointedInputGate();
 *               获取输入数据 InputGate[]，InputGate 是 flink 网络传输的核心抽象之一
 *               其在内部封装了消息的接收和内存的管理，从 InputGate 可以拿到上游传送过来的数据
 *        +----> inputProcessor = new StreamOneInputProcessor<>(input,output,operatorChain) 
 *        |      1. StreamInputProcessor，是 StreamTask 内部用来处理 Record 的组件，  
 *        |      里面封装了外部 IO 逻辑【内存不够时将 buffer 吐到磁盘上】以及 时间对齐逻辑【Watermark】
 *        |      2. output 是 StreamTaskNetworkOutput， input是StreamTaskNetworkInput
 *        |      这样就把input, output 他俩聚合进StreamOneInputProcessor
 *        +----> headOperator.getMetricGroup().gauge 
 *        +----> getEnvironment().getMetricGroup().gauge 
 *               设置一些 metrics 及 累加器
 * 
 * 
 *  -- run方法(就是基类StreamTask.run)
 *        +----> StreamTask.runMailboxLoop
 *        |      从 StreamTask.runMailboxLoop 开始，下面是一层层的调用关系
 *        -----> StreamTask.processInput()
 *        -----> StreamTask.inputProcessor.processInput()
 *        -----> StreamOneInputProcessor.processInput
 *        -----> input.emitNext(output) 
 *        -----> StreamTaskNetworkInput.emitNext()
 *        |      while(true) {从输入source读取一个record, output是 StreamTaskNetworkOutput}
 *        -----> StreamTaskNetworkInput.processElement()  //具体处理record
 *        |      根据StreamElement的不同类型做不同处理
 *        |      if (recordOrMark.isRecord()) output.emitRecord()
 *        ------------> StreamTaskNetworkOutput.emitRecord()  
 *        ----------------> operator.processElement(record)   
 *        |      if (recordOrMark.isWatermark()) statusWatermarkValve.inputWatermark()
 *        |      if (recordOrMark.isLatencyMarker()) output.emitLatencyMarker()
 *        |      if (recordOrMark.isStreamStatus()) statusWatermarkValve.inputStreamStatus()   
4. OperatorChain
flink 中的一个 operator 代表一个最顶级的 api 接口，拿 streaming 来说就是，在 DataStream 上做诸如 map/reduce/keyBy 等操作均会生成一个算子。

Operator Chain是指在生成JobGraph阶段，将Job中的Operators按照一定策略（例如：single output operator可以chain在一起）链接起来并放置在一个Task线程中执行。减少了数据传递/线程切换等环节，降低系统开销的同时增加了资源利用率和Job性能。

chained operators实际上是从下游往上游去反向一个个创建和setup的。假设chained operators为：StreamGroupedReduce - StreamFilter - StreamSink，而实际初始化顺序则相反：StreamSink - StreamFilter - StreamGroupedReduce。

 * OperatorChain(
 *			StreamTask<OUT, OP> containingTask,
 *			RecordWriterDelegate<SerializationDelegate<StreamRecord<OUT>>> recordWriterDelegate)
 * {@code
 *  -- collect
 *        |
 *        +----> pushToOperator(StreamRecord<X> record)
 *        +---------> operator.processElement(castRecord); 
 *        //这里的operator是chainedOperator，即除了headOperator之外，剩余的operators的chain。
 *        //这个operator.processElement，会循环调用operator chain所有operator，直到chain end。
 *        //比如 Operator A 对应的 ChainingOutput collect 调用了对应的算子 A 的 processElement 方法，这里又会调用 B 的 ChainingOutput 的 collect 方法，以此类推。这样便实现了可 chain 算子的本地处理，最终经由网络输出 RecordWriterOutput 发送到下游节点。   
5. StreamOperator
StreamTask会调用Operator，所以我们需要看看Operator的生命周期。

逻辑算子Transformation最后会对应到物理算子Operator，这个概念对应的就是StreamOperator。

StreamOperator是根接口。对于 Streaming 来说所有的算子都继承自 StreamOperator。继承了StreamOperator的扩展接口则有OneInputStreamOperator，TwoInputStreamOperator。实现了StreamOperator的抽象类有AbstractStreamOperator以及它的子类AbstractStreamUdfOperator。

其中operator处理输入的数据（elements）可以是以下之一：input element，watermark和checkpoint barriers。他们中的每一个都有一个特殊的单元来处理。element由processElement()方法处理，watermark由processWatermark()处理，checkpoint barriers由异步调用的snapshotState()方法处理，此方法会触发一次checkpoint 。

processElement()方法也是UDF的逻辑被调用的地方，例如MapFunction里的map()方法。

 * AbstractUdfStreamOperator, which is the basic class for all operators that execute UDFs.
 * 
 *         // initialization phase
 *         //初始化operator-specific方法，如RuntimeContext和metric collection
 *         OPERATOR::setup 
 *             UDF::setRuntimeContext
 *         //setup的调用链是invoke(StreamTask) -> constructor(OperatorChain) -> setup
 *         //调用setup时，StreamTask已经在各个TaskManager节点上 
 *         //给出一个用来初始state的operator   
 *  
 *         OPERATOR::initializeState
 *         //执行所有operator-specific的初始化  
 *         OPERATOR::open
 *            UDF::open
 *         
 *         // processing phase (called on every element/watermark)
 *         OPERATOR::processElement
 *             UDF::run //给定一个operator可以有一个用户定义的函数（UDF）
 *         OPERATOR::processWatermark
 *         
 *         // checkpointing phase (called asynchronously on every checkpoint)
 *         OPERATOR::snapshotState
 *                 
 *         // termination phase
 *         OPERATOR::close
 *             UDF::close
 *         OPERATOR::dispose
OneInputStreamOperator与TwoInputStreamOperator接口。这两个接口非常类似，本质上就是处理流上存在的三种元素StreamRecord，Watermark和LatencyMarker。一个用作单流输入，一个用作双流输入。

6. StreamSource
StreamSource是用来开启整个流的算子（继承AbstractUdfStreamOperator）。StreamSource因为没有输入，所以没有实现InputStreamOperator的接口。比较特殊的是ChainingStrategy初始化为HEAD。

在StreamSource这个类中，在运行时由SourceStreamTask调用SourceFunction的run方法来启动source。

 * class StreamSource<OUT, SRC extends SourceFunction<OUT>>
 *		extends AbstractUdfStreamOperator<OUT, SRC> implements StreamOperator<OUT> 
 * 
 *
 *  -- run()
 *        |   
 *        +----> latencyEmitter = new LatencyMarksEmitter
 *        |      用来产生延迟监控的LatencyMarker
 *        +----> this.ctx = StreamSourceContexts.getSourceContext
 *        |      据时间模式（EventTime/IngestionTime/ProcessingTime）生成相应SourceConext  
 *        |      包含了产生element关联的timestamp的方法和生成watermark的方法
 *        +----> userFunction.run(ctx);
 *        |      调用SourceFunction的run方法来启动source,进行数据的转发
 *        
public {
			//读到数据后，把数据交给collect方法，collect方法负责把数据交到合适的位置（如发布为br变量，或者交给下个operator，或者通过网络发出去）
	private transient SourceFunction.SourceContext<OUT> ctx;
	private transient volatile boolean canceledOrStopped = false;
	private transient volatile boolean hasSentMaxWatermark = false;
  
	public void run(final Object lockingObject,
			final StreamStatusMaintainer streamStatusMaintainer,
			final Output<StreamRecord<OUT>> collector,
			final OperatorChain<?, ?> operatorChain) throws Exception {
			userFunction.run(ctx);    
  }
}
7. StreamMap
StreamFilter，StreamMap与StreamFlatMap算子在实现的processElement分别调用传入的FilterFunction，MapFunction， FlatMapFunction的udf将element传到下游。这里用StreamMap举例：

public class StreamMap<IN, OUT>
		extends AbstractUdfStreamOperator<OUT, MapFunction<IN, OUT>>
		implements OneInputStreamOperator<IN, OUT> {

  public StreamMap(MapFunction<IN, OUT> mapper) {
		super(mapper);
		chainingStrategy = ChainingStrategy.ALWAYS;
	}

	@Override
	public void processElement(StreamRecord<IN> element) throws Exception {
		output.collect(element.replace(userFunction.map(element.getValue())));
	}
}
8. WindowOperator
Flink通过水位线分配器（TimestampsAndPeriodicWatermarksOperator和TimestampsAndPunctuatedWatermarksOperator这两个算子）向事件流中注入水位线。

我们示例代码中，timeWindow()最终对应了WindowStream，窗口算子WindowOperator是窗口机制的底层实现。assignTimestampsAndWatermarks 则对应了TimestampsAndPeriodicWatermarksOperator算子，它把产生的Watermark传递给了WindowOperator。

元素在streaming dataflow引擎中流动到WindowOperator时，会被分为两拨，分别是普通事件和水位线。

如果是普通的事件，则会调用processElement方法进行处理，在processElement方法中，首先会利用窗口分配器为当前接收到的元素分配窗口，接着会调用触发器的onElement方法进行逐元素触发。对于时间相关的触发器，通常会注册事件时间或者处理时间定时器，这些定时器会被存储在WindowOperator的处理时间定时器队列和水位线定时器队列中，如果触发的结果是FIRE，则对窗口进行计算。

如果是水位线（事件时间场景），则方法processWatermark将会被调用，它将会处理水位线定时器队列中的定时器。如果时间戳满足条件，则利用触发器的onEventTime方法进行处理。

而对于处理时间的场景，WindowOperator将自身实现为一个基于处理时间的触发器，以触发trigger方法来消费处理时间定时器队列中的定时器满足条件则会调用窗口触发器的onProcessingTime，根据触发结果判断是否对窗口进行计算。

 * public class WindowOperator<K, IN, ACC, OUT, W extends Window>
 * 	extends AbstractUdfStreamOperator<OUT, InternalWindowFunction<ACC, OUT, K, W>>
 * 	implements OneInputStreamOperator<IN, OUT>, Triggerable<K, W> 
 *
 *  -- processElement()
 *        |   
 *        +----> windowAssigner.assignWindows
 *        |      //通过WindowAssigner为element分配一系列windows
 *        +----> windowState.add(element.getValue())
 *        |      //把当前的element加入buffer state 
 *        +----> TriggerResult triggerResult = triggerContext.onElement(element)
 *        |      //触发onElment，得到triggerResult
 *        +----> Trigger.OnMergeContext.onElement()
 *        +----> trigger.onElement(element.getValue(), element.getTimestamp(), window,...)
 *        +----> EventTimeTriggers.onElement()
 *        |      //如果当前window.maxTimestamp已经小于CurrentWatermark，直接触发  
 *        |      //否则将window.maxTimestamp注册到TimeService中，等待触发   
 *        +----> contents = windowState.get(); emitWindowContents(actualWindow, contents)
 *        |      //对triggerResult做各种处理,如果fire，真正去计算窗口中的elements
   
 *  -- processWatermark()   
 *        -----> 最终进入基类AbstractStreamOperator.processWatermark
 *        -----> AbstractStreamOperator.processWatermark(watermark) 
 *        -----> timeServiceManager.advanceWatermark(mark); 第一步处理watermark
 *        -----> output.emitWatermark(mark) 第二步将watermark发送到下游
 *        -----> InternalTimeServiceManager.advanceWatermark      
0x06. 处理 Watermark 的简要流程
最后是处理 Watermark 的简要流程(OneInputStreamTask为例)

 *  -- OneInputStreamTask.invoke()
 *        |   
 *        +----> StreamTask.init 
 *        |      把StreamTaskNetworkOutput/StreamTaskNetworkInput聚合StreamOneInputProcessor
 *        +----> StreamTask.runMailboxLoop
 *        |      从 StreamTask.runMailboxLoop 开始，下面是一层层的调用关系
 *        -----> StreamTask.processInput()
 *        -----> StreamTask.inputProcessor.processInput()
 *        -----> StreamOneInputProcessor.processInput
 *        -----> input.emitNext(output)
 *        -----> StreamTaskNetworkInput.emitNext()
 *        -----> StreamTaskNetworkInput.processElement()
   
   
 *  下面是处理普通 Record  
 *  -- StreamTaskNetworkInput.processElement()  
 *        |   
 *        | 下面都是一层层的调用关系
 *        -----> output.emitRecord(recordOrMark.asRecord())
 *        -----> StreamTaskNetworkOutput.emitRecord()
 *        -----> operator.processElement(record)
 *               进入具体算子 processElement 的处理,比如StreamFlatMap.processElement
 *        -----> StreamFlatMap.processElement(record)
 *        -----> userFunction.flatMap()
    
   
 *  -- 下面是处理 Watermark
 *  -- StreamTaskNetworkInput.processElement()  
 *        |   
 *        | 下面都是一层层的调用关系
 *        -----> StatusWatermarkValve.inputWatermark()
 *        -----> StatusWatermarkValve.findAndOutputNewMinWatermarkAcrossAlignedChannels()
 *        -----> output.emitWatermark()
 *        -----> StreamTaskNetworkOutput.emitWatermark()
 *        -----> operator.processWatermark(watermark) 
 *        -----> KeyedProcessOperator.processWatermark(watermark) 
 *               具体算子processWatermark处理，如WindowOperator/KeyedProcessOperator.processWatermark 
 *               最终进入基类AbstractStreamOperator.processWatermark
 *        -----> AbstractStreamOperator.processWatermark(watermark) 
 *        -----> timeServiceManager.advanceWatermark(mark); 第一步处理watermark
 *               output.emitWatermark(mark) 第二步将watermark发送到下游
 *        -----> InternalTimeServiceManager.advanceWatermark   
 *        -----> 下面看看第一步处理watermark  
 *        -----> InternalTimerServiceImpl.advanceWatermark   
 *               逻辑timer时间小于watermark的都应该被触发回调。从eventTimeTimersQueue从小到大取timer，如果小于传入的water mark，那么说明这个window需要触发。注意watermarker是没有key的，所以当一个watermark来的时候是会触发所有timer，而timer的key是不一定的，所以这里一定要设置keyContext，否则就乱了
 *        -----> triggerTarget.onEventTime(timer);
 *               triggerTarget是具体operator对象，open时通过InternalTimeServiceManager.getInternalTimerService传递到HeapInternalTimerService  
 *        -----> KeyedProcessOperator.onEeventTime()
 *               调用用户实现的keyedProcessFunction.onTimer去做具体事情。对于window来说也是调用onEventTime或者onProcessTime来从key和window對應的状态中的数据发送到windowFunction中去计算并发送到下游节点  
 *        -----> invokeUserFunction(TimeDomain.PROCESSING_TIME, timer);
 *        -----> userFunction.onTimer(timer.getTimestamp(), onTimerContext, collector);
 
   
 *  -- DataStream 设置定时发送Watermark,是加了个chain的TimestampsAndPeriodicWatermarksOperator
 *  -- StreamTaskNetworkInput.processElement()        
 *        -----> TimestampsAndPeriodicWatermarksOperator.processElement
 *               会调用AssignerWithPeriodicWatermarks.extractTimestamp提取event time
 *               然后更新StreamRecord的时间
 *        -----> WindowOperator.processElement
 *               在windowAssigner.assignWindows时以element的timestamp作为assign时间
0x07 处理 Watermark 的详细流程（源码分析）
下面代码分析略冗长。

我们再看看样例代码

DataStream<String> text = env.socketTextStream(hostname, port);

DataStream counts = text
    .filter(new FilterClass())
    .map(new LineSplitter())
    .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessGenerator) 
    .keyBy(0)
    .timeWindow(Time.seconds(10))
    .sum(2)
  
counts.print()  
System.out.println(env.getExecutionPlan());
1. 程序逻辑 DataStream & Transformation
首先看看逻辑API。

DataStream是数据流概念。A DataStream represents a stream of elements of the same type。

Transformation是一个逻辑API概念。Transformation代表了流的转换，将一个或多个DataStream转换为新的DataStream。A Transformation is applied on one or more data streams or data sets and results in one or more output data streams or data sets。

我们认为Transformation就是逻辑算子，而 Transformation 对应的物理概念是Operators。

DataStream类在内部组合了一个 Transformation类，实际的转换操作均通过该类完成，描述了这个DataStream是怎么来的。

针对示例代码，"assignTimestampsAndWatermarks","Filter","Map"这几种，都被转换为 SingleOutputStreamOperator，继续由用户进行逻辑处理。SingleOutputStreamOperator这个类名字有点误导，实际上它是DataStream的子类。

@Public
public class DataStream<T> {
	protected final StreamExecutionEnvironment environment;
	protected final Transformation<T> transformation;  
    
  //assignTimestampsAndWatermarks这个操作实际上也生成了一个SingleOutputStreamOperator算子
	public SingleOutputStreamOperator<T> assignTimestampsAndWatermarks(
			AssignerWithPeriodicWatermarks<T> timestampAndWatermarkAssigner) {

		final int inputParallelism = getTransformation().getParallelism();
		final AssignerWithPeriodicWatermarks<T> cleanedAssigner = clean(timestampAndWatermarkAssigner);

		TimestampsAndPeriodicWatermarksOperator<T> operator =
				new TimestampsAndPeriodicWatermarksOperator<>(cleanedAssigner);

		return transform("Timestamps/Watermarks", getTransformation().getOutputType(), operator)
				.setParallelism(inputParallelism);
	}

  //Map是一个OneInputStreamOperator算子。
	public <R> SingleOutputStreamOperator<R> map(MapFunction<T, R> mapper, TypeInformation<R> outputType) {
		return transform("Map", outputType, new StreamMap<>(clean(mapper)));
	}

	@PublicEvolving
	public <R> SingleOutputStreamOperator<R> transform(
			String operatorName,
			TypeInformation<R> outTypeInfo,
			OneInputStreamOperatorFactory<T, R> operatorFactory) {
		return doTransform(operatorName, outTypeInfo, operatorFactory);
	}

	protected <R> SingleOutputStreamOperator<R> doTransform(
			String operatorName,
			TypeInformation<R> outTypeInfo,
			StreamOperatorFactory<R> operatorFactory) {

		// read the output type of the input Transform to coax out errors about MissingTypeInfo
		transformation.getOutputType();

		OneInputTransformation<T, R> resultTransform = new OneInputTransformation<>(
				this.transformation,
				operatorName,
				operatorFactory,
				outTypeInfo,
				environment.getParallelism());

    // SingleOutputStreamOperator 实际上是 DataStream 的子类，名字里面有Operator容易误导大家。
		@SuppressWarnings({"unchecked", "rawtypes"})
		SingleOutputStreamOperator<R> returnStream = new SingleOutputStreamOperator(environment, resultTransform);

		//就是把Transformation加到运行环境上去。
		getExecutionEnvironment().addOperator(resultTransform); 
		return returnStream;
	}	  
}
针对示例代码，绝大多数逻辑算子都转换为OneInputTransformation，每个Transformation里面间接记录了对应的物理Operator。注册到Env上。

// OneInputTransformation对应了单输入的算子
@Internal
public class OneInputTransformation<IN, OUT> extends PhysicalTransformation<OUT> {
	private final Transformation<IN> input;
	private final StreamOperatorFactory<OUT> operatorFactory; // 这里间接记录了本Transformation对应的物理Operator。比如StreamMap。
	private KeySelector<IN, ?> stateKeySelector;
	private TypeInformation<?> stateKeyType;
  
	public OneInputTransformation(
			Transformation<IN> input,
			String name,
			OneInputStreamOperator<IN, OUT> operator, // 比如StreamMap
			TypeInformation<OUT> outputType,
			int parallelism) {
		this(input, name, SimpleOperatorFactory.of(operator), outputType, parallelism);
	}  
}   
回到样例代码，DataStream.keyBy会返回一个KeyedStream。KeyedStream. timeWindow会返回一个WindowedStream。同时内部把各种 Transformation 注册到了 Env 中。

WindowedStream内部对应WindowedOperator。WindowedStream却不是Stream的子类! 而是把 KeyedStream 包含在内作为一个成员变量。

// 这个居然不是Stream的子类! 而是把 KeyedStream 包含在内作为一个成员变量。
@Public
public class WindowedStream<T, K, W extends Window> {
	private final KeyedStream<T, K> input; // 这里包含了DataStream。
	private final WindowAssigner<? super T, W> windowAssigner;
	private Trigger<? super T, ? super W> trigger;
	private Evictor<? super T, ? super W> evictor;
	private long allowedLateness = 0L;

  // reduce, fold等函数也是类似操作。
  private <R> SingleOutputStreamOperator<R> apply(InternalWindowFunction<Iterable<T>, R, K, W> function, TypeInformation<R> resultType, Function originalFunction) {

		final String opName = generateOperatorName(windowAssigner, trigger, evictor, originalFunction, null);
		KeySelector<T, K> keySel = input.getKeySelector();
		WindowOperator<K, T, Iterable<T>, R, W> operator;

		ListStateDescriptor<T> stateDesc = new ListStateDescriptor<>("window-contents",
				input.getType().createSerializer(getExecutionEnvironment().getConfig()));

      // 这里直接生成了 WindowOperator
			operator =
				new WindowOperator<>(windowAssigner,
					windowAssigner.getWindowSerializer(getExecutionEnvironment().getConfig()),
					keySel,
					input.getKeyType().createSerializer(getExecutionEnvironment().getConfig()),
					stateDesc,
					function,
					trigger,
					allowedLateness,
					lateDataOutputTag);
		}

		return input.transform(opName, resultType, operator);
}
在生成了程序逻辑之后，Env里面就有了 一系列 transformation（每个transformation里面记录了自己对应的物理 operator，比如StreamMap，WindowOperator），这个是后面生成计算图的基础。

当调用env.execute时，通过StreamGraphGenerator.generate遍历其中的transformation集合构造出StreamGraph。

2. 生成计算图
我们这里重点介绍StreamGraph以及如何生成，JobGraph，ExecutionGraph只是简介。

StreamGraph代表程序的拓扑结构，是从用户代码直接生成的图。StreamOperator是具体的物理算子。

一个很重要的点是，把 SourceStreamTask / OneInputStreamTask 添加到StreamNode上，作为 jobVertexClass，这个是真实计算的部分。

StreamOperator是一个接口。StreamOperator 是 数据流操作符的基础接口，该接口的具体实现子类中，会有保存用户自定义数据处理逻辑的函数的属性，负责对userFunction的调用，以及调用时传入所需参数，比如在StreamSource这个类中，在调用SourceFunction的run方法时，会构建一个SourceContext的具体实例，作为入参，用于run方法中，进行数据的转发；

StreamOperator
PublicEvolving
public interface StreamOperator<OUT> extends CheckpointListener, KeyContext, Disposable, Serializable {
}
AbstractStreamOperator
AbstractStreamOperator抽象类实现了StreamOperator。在AbstractStreamOperator中有一些重要的成员变量，总体来说可以分为几类，一类是运行时相关的，一类是状态相关的，一类是配置相关的，一类是时间相关的，还有一类是监控相关的。

@PublicEvolving
public abstract class AbstractStreamOperator<OUT>
		implements StreamOperator<OUT>, SetupableStreamOperator<OUT>, Serializable {
	protected ChainingStrategy chainingStrategy = ChainingStrategy.HEAD;
	private transient StreamTask<?, ?> container;
	protected transient StreamConfig config;
	protected transient Output<StreamRecord<OUT>> output;
	private transient StreamingRuntimeContext runtimeContext;
	public void processWatermark(Watermark mark) throws Exception {
		if (timeServiceManager != null) {
			timeServiceManager.advanceWatermark(mark); //第一步处理watermark
		}
		output.emitWatermark(mark);//第二步，将watermark发送到下游
	}  
}
AbstractUdfStreamOperator
AbstractUdfStreamOperator抽象类继承了AbstractStreamOperator，对其部分方法做了增强，多了一个成员变量UserFunction。提供了一些通用功能，比如把context赋给算子，保存快照等等。此外还实现了OutputTypeConfigurable接口的setOutputType方法对输出数据的类型做了设置。

@PublicEvolving
public abstract class AbstractUdfStreamOperator<OUT, F extends Function>
		extends AbstractStreamOperator<OUT>
		implements OutputTypeConfigurable<OUT> {
	protected final F userFunction;/** The user function. */
}
KeyedProcessOperator & WindowOperator。
KeyedStream，WindowedStream分别对应KeyedProcessOperator，WindowOperator。

@Internal
public class WindowOperator<K, IN, ACC, OUT, W extends Window>
	extends AbstractUdfStreamOperator<OUT, InternalWindowFunction<ACC, OUT, K, W>>
	implements OneInputStreamOperator<IN, OUT>, Triggerable<K, W> {
	protected final WindowAssigner<? super IN, W> windowAssigner;
	private final KeySelector<IN, K> keySelector;
	private final Trigger<? super IN, ? super W> trigger;
	private final StateDescriptor<? extends AppendingState<IN, ACC>, ?> windowStateDescriptor;
	protected final TypeSerializer<K> keySerializer;
	protected final TypeSerializer<W> windowSerializer;  
}

@Internal
public class KeyedProcessOperator<K, IN, OUT>
		extends AbstractUdfStreamOperator<OUT, KeyedProcessFunction<K, IN, OUT>>
		implements OneInputStreamOperator<IN, OUT>, Triggerable<K, VoidNamespace> {
	private transient TimestampedCollector<OUT> collector;
	private transient ContextImpl context;
	private transient OnTimerContextImpl onTimerContext;
  
	@Override
	public void open() throws Exception {
		super.open();
		collector = new TimestampedCollector<>(output);
		InternalTimerService<VoidNamespace> internalTimerService =
				getInternalTimerService("user-timers", VoidNamespaceSerializer.INSTANCE, this);
		TimerService timerService = new SimpleTimerService(internalTimerService);
		context = new ContextImpl(userFunction, timerService);
		onTimerContext = new OnTimerContextImpl(userFunction, timerService);
	}

	@Override
	public void onEventTime(InternalTimer<K, VoidNamespace> timer) throws Exception {
		collector.setAbsoluteTimestamp(timer.getTimestamp());
		invokeUserFunction(TimeDomain.EVENT_TIME, timer);
	}

	@Override
	public void onProcessingTime(InternalTimer<K, VoidNamespace> timer) throws Exception {
		collector.eraseTimestamp();
		invokeUserFunction(TimeDomain.PROCESSING_TIME, timer);
	}

	@Override
	public void processElement(StreamRecord<IN> element) throws Exception {
		collector.setTimestamp(element);
		context.element = element;
		userFunction.processElement(element.getValue(), context, collector);
		context.element = null;
	}

	private void invokeUserFunction(
			TimeDomain timeDomain,
			InternalTimer<K, VoidNamespace> timer) throws Exception {
		onTimerContext.timeDomain = timeDomain;
		onTimerContext.timer = timer;
		userFunction.onTimer(timer.getTimestamp(), onTimerContext, collector);
		onTimerContext.timeDomain = null;
		onTimerContext.timer = null;
	}  
}
OneInputStreamOperator & TwoInputStreamOperator
承接输入数据并进行处理的算子就是OneInputStreamOperator、TwoInputStreamOperator等。 这两个接口非常类似，本质上就是处理流上存在的三种元素StreamRecord，Watermark和LatencyMarker。一个用作单流输入，一个用作双流输入。除了StreamSource以外的所有Stream算子都必须实现并且只能实现其中一个接口。

@PublicEvolving
public interface OneInputStreamOperator<IN, OUT> extends StreamOperator<OUT> {
	void processElement(StreamRecord<IN> element) throws Exception;
	void processWatermark(Watermark mark) throws Exception;
	void processLatencyMarker(LatencyMarker latencyMarker) throws Exception;
}
StreamMap & StreamFlatMap
map，filter等常用操作都是OneInputStreamOperator。下面给出StreamMap，StreamFlatMap作为具体例子。

// 用StreamMap里做个实际算子的例子@
Internal
public class StreamMap<IN, OUT>
		extends AbstractUdfStreamOperator<OUT, MapFunction<IN, OUT>>
		implements OneInputStreamOperator<IN, OUT> {

	private static final long serialVersionUID = 1L;

	public StreamMap(MapFunction<IN, OUT> mapper) {
		super(mapper);
		chainingStrategy = ChainingStrategy.ALWAYS;
	}

	@Override
	public void processElement(StreamRecord<IN> element) throws Exception {
		output.collect(element.replace(userFunction.map(element.getValue())));
	}
}

// 用StreamFlatMap里做个实际算子的例子
@Internal
public class StreamFlatMap<IN, OUT>
		extends AbstractUdfStreamOperator<OUT, FlatMapFunction<IN, OUT>>
		implements OneInputStreamOperator<IN, OUT> {

	private transient TimestampedCollector<OUT> collector;

	public StreamFlatMap(FlatMapFunction<IN, OUT> flatMapper) {
		super(flatMapper);
		chainingStrategy = ChainingStrategy.ALWAYS;
	}

	@Override
	public void open() throws Exception {
		super.open();
		collector = new TimestampedCollector<>(output);
	}

	@Override
	public void processElement(StreamRecord<IN> element) throws Exception {
		collector.setTimestamp(element);
		userFunction.flatMap(element.getValue(), collector);
	}
}
生成StreamGraph
程序执行即env.execute("Java WordCount from SocketTextStream Example")这行代码的时候，就会生成StreamGraph。代表程序的拓扑结构，是从用户代码直接生成的图。

StreamGraph生成函数分析
实际生成StreamGraph的入口是StreamGraphGenerator.generate(env, transformations) 。其中的transformations是一个list，里面记录的就是我们在transform方法中放进来的算子。最终会调用 transformXXX 来对具体的Transformation进行转换。

@Internal
public class StreamGraphGenerator {
	private final List<Transformation<?>> transformations;
	private StreamGraph streamGraph;

	public StreamGraph generate() {
		//注意，StreamGraph的生成是从sink开始的
		streamGraph = new StreamGraph(executionConfig, checkpointConfig, savepointRestoreSettings);

		for (Transformation<?> transformation: transformations) {
			transform(transformation);
		}

		final StreamGraph builtStreamGraph = streamGraph;
		return builtStreamGraph;
	}	

	private Collection<Integer> transform(Transformation<?> transform) {
		//这个方法的核心逻辑就是判断传入的steamOperator是哪种类型，并执行相应的操作，详情见下面那一大堆if-else
		//这里对操作符的类型进行判断，并以此调用相应的处理逻辑.简而言之，处理的核心无非是递归的将该节点和节点的上游节点加入图
		Collection<Integer> transformedIds;
		if (transform instanceof OneInputTransformation<?, ?>) {
			transformedIds = transformOneInputTransform((OneInputTransformation<?, ?>) transform);
		} else if (transform instanceof TwoInputTransformation<?, ?, ?>) {
			transformedIds = transformTwoInputTransform((TwoInputTransformation<?, ?, ?>) transform);
    }
        .......
  }

  //因为map，filter等常用操作都是OneInputStreamOperator,我们就来看看StreamGraphGenerator.transformOneInputTransform((OneInputTransformation<?, ?>) transform)方法。
  //该函数首先会对该transform的上游transform进行递归转换，确保上游的都已经完成了转化。然后通过transform构造出StreamNode，最后与上游的transform进行连接，构造出StreamNode。

	private <IN, OUT> Collection<Integer> transformOneInputTransform(OneInputTransformation<IN, OUT> transform) {
		//就是递归处理节点，为当前节点和它的依赖节点建立边，处理边之类的，把节点加到图里。
		Collection<Integer> inputIds = transform(transform.getInput());
		String slotSharingGroup = determineSlotSharingGroup(transform.getSlotSharingGroup(), inputIds);
    // 这里添加Operator到streamGraph上。
		streamGraph.addOperator(transform.getId(),
				slotSharingGroup,
				transform.getCoLocationGroupKey(),
				transform.getOperatorFactory(),
				transform.getInputType(),
				transform.getOutputType(),
				transform.getName());

		if (transform.getStateKeySelector() != null) {
			TypeSerializer<?> keySerializer = transform.getStateKeyType().createSerializer(executionConfig);
			streamGraph.setOneInputStateKey(transform.getId(), transform.getStateKeySelector(), keySerializer);
		}

		int parallelism = transform.getParallelism() != ExecutionConfig.PARALLELISM_DEFAULT ?
			transform.getParallelism() : executionConfig.getParallelism();
		streamGraph.setParallelism(transform.getId(), parallelism);
		streamGraph.setMaxParallelism(transform.getId(), transform.getMaxParallelism());

		for (Integer inputId: inputIds) {
			streamGraph.addEdge(inputId, transform.getId(), 0);
		}
		return Collections.singleton(transform.getId());
	}
}
streamGraph.addOperator
在之前的生成图代码中，有streamGraph.addOperator，我们具体看看实现。

这里重要的是把 SourceStreamTask / OneInputStreamTask 添加到StreamNode上，作为 jobVertexClass。

@Internal
public class StreamGraph implements Pipeline {
  
  public <IN, OUT> void addOperator(
        Integer vertexID,
        @Nullable String slotSharingGroup,
        @Nullable String coLocationGroup,
        StreamOperatorFactory<OUT> operatorFactory,
        TypeInformation<IN> inTypeInfo,
        TypeInformation<OUT> outTypeInfo,
        String operatorName) {

      // 这里添加了 OneInputStreamTask/SourceStreamTask，这个是日后真实运行的地方。
      if (operatorFactory.isStreamSource()) {
        addNode(vertexID, slotSharingGroup, coLocationGroup, SourceStreamTask.class, operatorFactory, operatorName);
      } else {
        addNode(vertexID, slotSharingGroup, coLocationGroup, OneInputStreamTask.class, operatorFactory, operatorName);
      }
  }

	protected StreamNode addNode(Integer vertexID,
		@Nullable String slotSharingGroup,
		@Nullable String coLocationGroup,
		Class<? extends AbstractInvokable> vertexClass, // 这里是OneInputStreamTask...
		StreamOperatorFactory<?> operatorFactory,
		String operatorName) {

		StreamNode vertex = new StreamNode(
			vertexID,
			slotSharingGroup,
			coLocationGroup,
			operatorFactory,
			operatorName,
			new ArrayList<OutputSelector<?>>(),
			vertexClass);

		streamNodes.put(vertexID, vertex);
		return vertex;
	}
}
关键类StreamNode
@Internal
public class StreamNode implements Serializable {
	private transient StreamOperatorFactory<?> operatorFactory;
	private List<OutputSelector<?>> outputSelectors;
	private List<StreamEdge> inEdges = new ArrayList<StreamEdge>();
	private List<StreamEdge> outEdges = new ArrayList<StreamEdge>();
	private final Class<? extends AbstractInvokable> jobVertexClass; // OneInputStreamTask
  
	@VisibleForTesting
	public StreamNode(
			Integer id,
			@Nullable String slotSharingGroup,
			@Nullable String coLocationGroup,
			StreamOperator<?> operator,
			String operatorName,
			List<OutputSelector<?>> outputSelector,
			Class<? extends AbstractInvokable> jobVertexClass) {
		this(id, slotSharingGroup, coLocationGroup, SimpleOperatorFactory.of(operator),
				operatorName, outputSelector, jobVertexClass);
	}  
  
	public Class<? extends AbstractInvokable> getJobVertexClass() {
		return jobVertexClass;
	}  
}
3. Task之间数据交换机制
Flink中的数据交换构建在如下两条设计原则之上：

数据交换的控制流（例如，为实例化交换而进行的消息传输）是接收端初始化的，这非常像最初的MapReduce。
数据交换的数据流（例如，在网络上最终传输的数据）被抽象成一个叫做IntermediateResult的概念，它是可插拔的。这意味着系统基于相同的实现逻辑可以既支持流数据，又支持批处理数据的传输。
数据在task之间传输整体过程
第一步必然是准备一个ResultPartition；
通知JobMaster；
JobMaster通知下游节点；如果下游节点尚未部署，则部署之；
下游节点向上游请求数据
开始传输数据
数据在task之间具体传输
描述了数据从生产者传输到消费者的完整生命周期。

数据在task之间传递有如下几步：

数据在本operator处理完后，通过Collector收集，这些记录被传给RecordWriter对象。每条记录都要选择一个下游节点，所以要经过ChannelSelector。一个ChannelSelector选择一个或者多个序列化器来处理记录。如果记录在broadcast中，它们将被传递给每一个序列化器。如果记录是基于hash分区的，ChannelSelector将会计算记录的hash值，然后选择合适的序列化器。
每个channel都有一个serializer，序列化器将record数据记录序列化成二进制的表示形式。然后将它们放到大小合适的buffer中（记录也可以被切割到多个buffer中）。
接下来数据被写入ResultPartition下的各个subPartition （ResultSubpartition - RS，用于为特定的消费者收集buffer数据）里，此时该数据已经存入DirectBuffer（MemorySegment）。既然首个buffer进来了，RS就对消费者变成可访问的状态了（注意，这个行为实现了一个streaming shuffle），然后它通知JobManager。
JobManager查找RS的消费者，然后通知TaskManager一个数据块已经可以访问了。通知TM2的消息会被发送到InputChannel，该inputchannel被认为是接收这个buffer的，接着通知RS2可以初始化一个网络传输了。然后，RS2通过TM1的网络栈请求该buffer，然后双方基于netty准备进行数据传输。网络连接是在TaskManager（而非特定的task）之间长时间存在的。
单独的线程控制数据的flush速度，一旦触发flush，则通过Netty的nio通道向对端写入。
对端的netty client接收到数据，decode出来，把数据拷贝到buffer里，然后通知InputChannel
一旦buffer被TM2接收，它会穿过一个类似的对象栈，起始于InputChannel（接收端 等价于IRPQ）,进入InputGate（它包含多个IC），最终进入一个RecordDeserializer，它用于从buffer中还原成类型化的记录，然后将其传递给接收task。
有可用的数据时，下游算子从阻塞醒来。从InputChannel取出buffer，再解序列化成record，交给算子执行用户代码。
4. 数据源的逻辑——StreamSource与时间模型
SourceFunction是所有stream source的根接口。

StreamSource抽象了一个数据源，并且指定了一些如何处理数据的模式。StreamSource是用来开启整个流的算子。SourceFunction定义了两个接口方法：

run ： 启动一个source，即对接一个外部数据源然后emit元素形成stream（大部分情况下会通过在该方法里运行一个while循环的形式来产生stream）。
cancel ： 取消一个source，也即将run中的循环emit元素的行为终止。

@Public
public interface SourceFunction<T> extends Function, Serializable {
	void run(SourceContext<T> ctx) throws Exception;
	void cancel();
	@Public // Interface might be extended in the future with additional methods.
  //SourceContex则是用来进行数据发送的接口。
	interface SourceContext<T> {
      void collect(T element);
      @PublicEvolving
      void collectWithTimestamp(T element, long timestamp);
      @PublicEvolving
      void emitWatermark(Watermark mark);
      @PublicEvolving
      void markAsTemporarilyIdle();
      Object getCheckpointLock();
      void close();
	}  
}

public class StreamSource<OUT, SRC extends SourceFunction<OUT>>
		extends AbstractUdfStreamOperator<OUT, SRC> implements StreamOperator<OUT> {
			//读到数据后，把数据交给collect方法，collect方法负责把数据交到合适的位置（如发布为br变量，或者交给下个operator，或者通过网络发出去）
	private transient SourceFunction.SourceContext<OUT> ctx;
	private transient volatile boolean canceledOrStopped = false;
	private transient volatile boolean hasSentMaxWatermark = false;
  
	public void run(final Object lockingObject,
			final StreamStatusMaintainer streamStatusMaintainer,
			final Output<StreamRecord<OUT>> collector,
			final OperatorChain<?, ?> operatorChain) throws Exception {
			userFunction.run(ctx);    
  }
}
SocketTextStreamFunction
回到实例代码，env.socketTextStream(hostname, port)就是生成了SocketTextStreamFunction。

run方法的逻辑如上，逻辑很清晰，就是从指定的hostname和port持续不断的读取数据，按行分隔符划分成一个个字符串，然后转发到下游。

cancel方法的实现如下，就是将运行状态的标识isRunning属性设置为false，并根据需要关闭当前socket。

@PublicEvolving
public class SocketTextStreamFunction implements SourceFunction<String> {
	private final String hostname;
	private final int port;
	private final String delimiter;
	private final long maxNumRetries;
	private final long delayBetweenRetries;
	private transient Socket currentSocket;
	private volatile boolean isRunning = true;

  public SocketTextStreamFunction(String hostname, int port, String delimiter, long maxNumRetries) {
		this(hostname, port, delimiter, maxNumRetries, DEFAULT_CONNECTION_RETRY_SLEEP);
	}

	public void run(SourceContext<String> ctx) throws Exception {
   final StringBuilder buffer = new StringBuilder();
   long attempt = 0;
   /** 这里是第一层循环，只要当前处于运行状态，该循环就不会退出，会一直循环 */
   while (isRunning) {
      try (Socket socket = new Socket()) {
         /** 对指定的hostname和port，建立Socket连接，并构建一个BufferedReader，用来从Socket中读取数据 */
         currentSocket = socket;
         LOG.info("Connecting to server socket " + hostname + ':' + port);
         socket.connect(new InetSocketAddress(hostname, port), CONNECTION_TIMEOUT_TIME);
         BufferedReader reader = new BufferedReader(new InputStreamReader(socket.getInputStream()));
         char[] cbuf = new char[8192];
         int bytesRead;
         /** 这里是第二层循环，对运行状态进行了双重校验，同时对从Socket中读取的字节数进行判断 */
         while (isRunning && (bytesRead = reader.read(cbuf)) != -1) {
            buffer.append(cbuf, 0, bytesRead);
            int delimPos;
            /** 这里是第三层循环，就是对从Socket中读取到的数据，按行分隔符进行分割，并将每行数据作为一个整体字符串向下游转发 */
            while (buffer.length() >= delimiter.length() && (delimPos = buffer.indexOf(delimiter)) != -1) {
               String record = buffer.substring(0, delimPos);
               if (delimiter.equals("\n") && record.endsWith("\r")) {
                  record = record.substring(0, record.length() - 1);
               }
               /** 用入参ctx，进行数据的转发 */
               ctx.collect(record);
               buffer.delete(0, delimPos + delimiter.length());
            }
         }
      }
      /** 如果由于遇到EOF字符，导致从循环中退出，则根据运行状态，以及设置的最大重试尝试次数，决定是否进行 sleep and retry，或者直接退出循环 */
      if (isRunning) {
         attempt++;
         if (maxNumRetries == -1 || attempt < maxNumRetries) {
            LOG.warn("Lost connection to server socket. Retrying in " + delayBetweenRetries + " msecs...");
            Thread.sleep(delayBetweenRetries);
         }
         else {
            break;
         }
      }
   }
   /** 在最外层的循环都退出后，最后检查下缓存中是否还有数据，如果有，则向下游转发 */
   if (buffer.length() > 0) {
      ctx.collect(buffer.toString());
   }
	}
  
	public void cancel() {
   isRunning = false;
   Socket theSocket = this.currentSocket;
   /** 如果当前socket不为null，则进行关闭操作 */
   if (theSocket != null) {
      IOUtils.closeSocket(theSocket);
   }
	}  
}
5. StreamTask
回到实例代码，filter，map是在StreamTask中执行，可以看看StreamTask等具体定义。

@Internal
public abstract class StreamTask<OUT, OP extends StreamOperator<OUT>>
		extends AbstractInvokable
		implements AsyncExceptionHandler {
	private final StreamTaskActionExecutor actionExecutor;

  /**
	 * The input processor. Initialized in {@link #init()} method.
	 */
	@Nullable
	protected StreamInputProcessor inputProcessor; // 这个是处理关键。

	/** the head operator that consumes the input streams of this task. */
	protected OP headOperator;

	/** The chain of operators executed by this task. */
	protected OperatorChain<OUT, OP> operatorChain;

	/** The configuration of this streaming task. */
	protected final StreamConfig configuration;

	/** Our state backend. We use this to create checkpoint streams and a keyed state backend. */
	protected StateBackend stateBackend;

	/** The external storage where checkpoint data is persisted. */
	private CheckpointStorageWorkerView checkpointStorage;

	/**
	 * The internal {@link TimerService} used to define the current
	 * processing time (default = {@code System.currentTimeMillis()}) and
	 * register timers for tasks to be executed in the future.
	 */
	protected TimerService timerService;

	private final Thread.UncaughtExceptionHandler uncaughtExceptionHandler;

	/** The map of user-defined accumulators of this task. */
	private final Map<String, Accumulator<?, ?>> accumulatorMap;

	/** The currently active background materialization threads. */
	private final CloseableRegistry cancelables = new CloseableRegistry();

	private final StreamTaskAsyncExceptionHandler asyncExceptionHandler;

	/**
	 * Flag to mark the task "in operation", in which case check needs to be initialized to true,
	 * so that early cancel() before invoke() behaves correctly.
	 */
	private volatile boolean isRunning;

	/** Flag to mark this task as canceled. */
	private volatile boolean canceled;

	private boolean disposedOperators;

	/** Thread pool for async snapshot workers. */
	private ExecutorService asyncOperationsThreadPool;

	private final RecordWriterDelegate<SerializationDelegate<StreamRecord<OUT>>> recordWriter;

	protected final MailboxProcessor mailboxProcessor;

	private Long syncSavepointId = null;  
  
	@Override
	public final void invoke() throws Exception {
		try {
			beforeInvoke();

			// final check to exit early before starting to run
			if (canceled) {
				throw new CancelTaskException();
			}

			// let the task do its work
			isRunning = true;
			runMailboxLoop(); //MailboxProcessor.runMailboxLoop会调用StreamTask.processInput

			// if this left the run() method cleanly despite the fact that this was canceled,
			// make sure the "clean shutdown" is not attempted
			if (canceled) {
				throw new CancelTaskException();
			}

			afterInvoke();
		}
		finally {
			cleanUpInvoke();
		}
	} 
  
	protected void processInput(MailboxDefaultAction.Controller controller) throws Exception {
		InputStatus status = inputProcessor.processInput(); // 这里会具体从source读取数据。
		if (status == InputStatus.MORE_AVAILABLE && recordWriter.isAvailable()) {
			return;
		}
		if (status == InputStatus.END_OF_INPUT) {
			controller.allActionsCompleted();
			return;
		}
    //具体执行操作。
		CompletableFuture<?> jointFuture = getInputOutputJointFuture(status);
		MailboxDefaultAction.Suspension suspendedDefaultAction = controller.suspendDefaultAction();
		jointFuture.thenRun(suspendedDefaultAction::resume);
	}  
}
前面提到，Task对象在执行过程中，把执行的任务交给了StreamTask这个类去执行。在我们的wordcount例子中，实际初始化的是OneInputStreamTask的对象。那么这个对象是如何执行用户的代码的呢？

它做的如下：

首先，初始化 initialize-operator-states()。

然后 open-operators() 方法。

最后调用 StreamTask#runMailboxLoop，便开始处理Source端消费的数据，并流入下游算子处理。

具体来说，就是把任务直接交给了InputProcessor去执行processInput方法。这是一个StreamInputProcessor的实例，该processor的任务就是处理输入的数据，包括用户数据、watermark和checkpoint数据等。

具体到OneInputStreamTask，OneInputStreamTask.inputProcessor 是 StreamOneInputProcessor 类型，它把input, output聚合在一起。input是StreamTaskNetworkInput类型。output是StreamTaskNetworkOutput类型。

具体代码如下

@Internal
public class OneInputStreamTask<IN, OUT> extends StreamTask<OUT, OneInputStreamOperator<IN, OUT>> {
  //这是OneInputStreamTask的init方法，从configs里面获取StreamOperator信息，生成自己的inputProcessor。
	@Override
	public void init() throws Exception {
		StreamConfig configuration = getConfiguration();
		int numberOfInputs = configuration.getNumberOfInputs();
		if (numberOfInputs > 0) {
			CheckpointedInputGate inputGate = createCheckpointedInputGate();
			DataOutput<IN> output = createDataOutput(); //  这里生成了 StreamTaskNetworkOutput
			StreamTaskInput<IN> input = createTaskInput(inputGate, output);
			inputProcessor = new StreamOneInputProcessor<>( // 这里把input, output通过Processor配置到了一起。
				input,
				output,
				operatorChain);
		}
		headOperator.getMetricGroup().gauge(MetricNames.IO_CURRENT_INPUT_WATERMARK, this.inputWatermarkGauge);
	}
  
	private StreamTaskInput<IN> createTaskInput(CheckpointedInputGate inputGate, DataOutput<IN> output) {
		int numberOfInputChannels = inputGate.getNumberOfInputChannels();
		StatusWatermarkValve statusWatermarkValve = new StatusWatermarkValve(numberOfInputChannels, output);

		TypeSerializer<IN> inSerializer = configuration.getTypeSerializerIn1(getUserCodeClassLoader());
		return new StreamTaskNetworkInput<>(
			inputGate,
			inSerializer,
			getEnvironment().getIOManager(),
			statusWatermarkValve,
			0);
	}  
  
	/**
	 * The network data output implementation used for processing stream elements
	 * from {@link StreamTaskNetworkInput} in one input processor.
	 */
	private static class StreamTaskNetworkOutput<IN> extends AbstractDataOutput<IN> {

		private final OneInputStreamOperator<IN, ?> operator;
		private final WatermarkGauge watermarkGauge;
		private final Counter numRecordsIn;

		private StreamTaskNetworkOutput(
				OneInputStreamOperator<IN, ?> operator, // 这个就是注册的Operator
				StreamStatusMaintainer streamStatusMaintainer,
				WatermarkGauge watermarkGauge,
				Counter numRecordsIn) {
			super(streamStatusMaintainer);

			this.operator = checkNotNull(operator);
			this.watermarkGauge = checkNotNull(watermarkGauge);
			this.numRecordsIn = checkNotNull(numRecordsIn);
		}

		@Override
		public void emitRecord(StreamRecord<IN> record) throws Exception {
			numRecordsIn.inc();
			operator.setKeyContextElement1(record);
			operator.processElement(record);
		}

		@Override
		public void emitWatermark(Watermark watermark) throws Exception {
			watermarkGauge.setCurrentWatermark(watermark.getTimestamp());
			operator.processWatermark(watermark); // 这里就进入了processWatermark具体处理，比如WindowOperator的
		}

		@Override
		public void emitLatencyMarker(LatencyMarker latencyMarker) throws Exception {
			operator.processLatencyMarker(latencyMarker);
		}
	}  
}

@Internal
public interface StreamInputProcessor extends AvailabilityProvider, Closeable {
	InputStatus processInput() throws Exception;
}

@Internal
public final class StreamOneInputProcessor<IN> implements StreamInputProcessor {
	@Override
	public InputStatus processInput() throws Exception {
		InputStatus status = input.emitNext(output);  // 这里是开始从输入source读取一个record。input, output分别是 StreamTaskNetworkInput，StreamTaskNetworkOutput。
		if (status == InputStatus.END_OF_INPUT) {
			operatorChain.endHeadOperatorInput(1);
		}
		return status;
	}
}

@Internal
public final class StreamTaskNetworkInput<T> implements StreamTaskInput<T> {
  
	@Override
	public InputStatus emitNext(DataOutput<T> output) throws Exception {

		while (true) {
			// get the stream element from the deserializer
			if (currentRecordDeserializer != null) {
				DeserializationResult result = currentRecordDeserializer.getNextRecord(deserializationDelegate);
				if (result.isBufferConsumed()) {
					currentRecordDeserializer.getCurrentBuffer().recycleBuffer();
					currentRecordDeserializer = null;
				}

				if (result.isFullRecord()) {
					processElement(deserializationDelegate.getInstance(), output); //具体处理record
					return InputStatus.MORE_AVAILABLE;
				}
			}

			Optional<BufferOrEvent> bufferOrEvent = checkpointedInputGate.pollNext();
			if (bufferOrEvent.isPresent()) {
				processBufferOrEvent(bufferOrEvent.get());
			} else {
				if (checkpointedInputGate.isFinished()) {
					checkState(checkpointedInputGate.getAvailableFuture().isDone(), "Finished BarrierHandler should be available");
					if (!checkpointedInputGate.isEmpty()) {
						throw new IllegalStateException("Trailing data in checkpoint barrier handler.");
					}
					return InputStatus.END_OF_INPUT;
				}
				return InputStatus.NOTHING_AVAILABLE;
			}
		}
	}

  // 根据record类型，来处理record还是watermark
	private void processElement(StreamElement recordOrMark, DataOutput<T> output) throws Exception {
		if (recordOrMark.isRecord()){
			output.emitRecord(recordOrMark.asRecord()); // 调用 StreamTaskNetworkOutput，最终调用到operator.processElement(record);
		} else if (recordOrMark.isWatermark()) {
			statusWatermarkValve.inputWatermark(recordOrMark.asWatermark(), lastChannel);
		} else if (recordOrMark.isLatencyMarker()) {
			output.emitLatencyMarker(recordOrMark.asLatencyMarker());
		} else if (recordOrMark.isStreamStatus()) {
			statusWatermarkValve.inputStreamStatus(recordOrMark.asStreamStatus(), lastChannel);
		} else {
			throw new UnsupportedOperationException("Unknown type of StreamElement");
		}
	}
}

@PublicEvolving
public abstract class AbstractStreamOperator<OUT>
		implements StreamOperator<OUT>, SetupableStreamOperator<OUT>, Serializable {
  	protected transient InternalTimeServiceManager<?> timeServiceManager;
	public void processWatermark(Watermark mark) throws Exception {
		if (timeServiceManager != null) {
			timeServiceManager.advanceWatermark(mark);
		}
		output.emitWatermark(mark);
	}  
}

@Internal
public class InternalTimeServiceManager<K> {
	private final Map<String, InternalTimerServiceImpl<K, ?>> timerServices;  
	public void advanceWatermark(Watermark watermark) throws Exception {
		for (InternalTimerServiceImpl<?, ?> service : timerServices.values()) {
			service.advanceWatermark(watermark.getTimestamp());
		}
	}
}  

public class InternalTimerServiceImpl<K, N> implements InternalTimerService<N> {
	private final ProcessingTimeService processingTimeService;
	private final KeyContext keyContext;
	private final KeyGroupedInternalPriorityQueue<TimerHeapInternalTimer<K, N>> processingTimeTimersQueue;
	private final KeyGroupedInternalPriorityQueue<TimerHeapInternalTimer<K, N>> eventTimeTimersQueue;
	private final KeyGroupRange localKeyGroupRange;
	private final int localKeyGroupRangeStartIdx;  
	public void advanceWatermark(long time) throws Exception {
		currentWatermark = time;
		InternalTimer<K, N> timer;
		while ((timer = eventTimeTimersQueue.peek()) != null && timer.getTimestamp() <= time) {
			eventTimeTimersQueue.poll();
			keyContext.setCurrentKey(timer.getKey());
			triggerTarget.onEventTime(timer);
		}
	}  
}
上面的代码中，StreamTaskNetworkOutput.emitRecord中的operator.processElement(record);才是真正处理用户逻辑的代码。

StatusWatermarkValve就是用来处理watermark的。

@Internal
public class StatusWatermarkValve {
	private final DataOutput output;
	
	public void inputWatermark(Watermark watermark, int channelIndex) throws Exception {
		// ignore the input watermark if its input channel, or all input channels are idle (i.e. overall the valve is idle).
		if (lastOutputStreamStatus.isActive() && channelStatuses[channelIndex].streamStatus.isActive()) {
			long watermarkMillis = watermark.getTimestamp();

			// if the input watermark's value is less than the last received watermark for its input channel, ignore it also.
			if (watermarkMillis > channelStatuses[channelIndex].watermark) {
				channelStatuses[channelIndex].watermark = watermarkMillis;

				// previously unaligned input channels are now aligned if its watermark has caught up
				if (!channelStatuses[channelIndex].isWatermarkAligned && watermarkMillis >= lastOutputWatermark) {
					channelStatuses[channelIndex].isWatermarkAligned = true;
				}

				// now, attempt to find a new min watermark across all aligned channels
				findAndOutputNewMinWatermarkAcrossAlignedChannels();
			}
		}
	}	
	
	private void findAndOutputNewMinWatermarkAcrossAlignedChannels() throws Exception {
		long newMinWatermark = Long.MAX_VALUE;
		boolean hasAlignedChannels = false;

		// determine new overall watermark by considering only watermark-aligned channels across all channels
		for (InputChannelStatus channelStatus : channelStatuses) {
			if (channelStatus.isWatermarkAligned) {
				hasAlignedChannels = true;
				newMinWatermark = Math.min(channelStatus.watermark, newMinWatermark);
			}
		}

		// we acknowledge and output the new overall watermark if it really is aggregated
		// from some remaining aligned channel, and is also larger than the last output watermark
		if (hasAlignedChannels && newMinWatermark > lastOutputWatermark) {
			lastOutputWatermark = newMinWatermark;
			output.emitWatermark(new Watermark(lastOutputWatermark)); // 这里会最终emit watermark
		}
	}	
}
6. Watermarks的生成
而Watermark的产生是在Apache Flink的Source节点 或 Watermark生成器计算产生(如Apache Flink内置的 Periodic Watermark实现)

There are two ways to assign timestamps and generate Watermarks:

Directly in the data stream source 自定义数据源设置 Timestamp/Watermark
Via a TimestampAssigner / WatermarkGenerator 在数据流中设置 Timestamp/Watermark。
自定义数据源设置 Timestamp/Watermark
自定义的数据源类需要继承并实现 SourceFunction[T] 接口，其中 run 方法是定义数据生产的地方：

//自定义的数据源为自定义类型MyType
class MySource extends SourceFunction[MyType]{

    //重写run方法，定义数据生产的逻辑
    override def run(ctx: SourceContext[MyType]): Unit = {
        while (/* condition */) {
            val next: MyType = getNext()
            //设置timestamp从MyType的哪个字段获取(eventTimestamp)
            ctx.collectWithTimestamp(next, next.eventTimestamp)
    
            if (next.hasWatermarkTime) {
                //设置watermark从MyType的那个方法获取(getWatermarkTime)
                ctx.emitWatermark(new Watermark(next.getWatermarkTime))
            }
        }
    }
}
在数据流中设置 Timestamp/Watermark
在数据流中，可以设置 stream 的 Timestamp Assigner ，该 Assigner 将会接收一个 stream，并生产一个带 Timestamp和Watermark 的新 stream。

Flink通过水位线分配器（TimestampsAndPeriodicWatermarksOperator和TimestampsAndPunctuatedWatermarksOperator这两个算子）向事件流中注入水位线。元素在streaming dataflow引擎中流动到WindowOperator时，会被分为两拨，分别是普通事件和水位线。

回到实例代码，assignTimestampsAndWatermarks 就是生成一个TimestampsAndPeriodicWatermarksOperator。

TimestampsAndPeriodicWatermarksOperator的具体处理 Watermark代码如下。其中processWatermark具体是阻断上游水位线，这样下游就只能用自身产生的水位线了。

public class TimestampsAndPeriodicWatermarksOperator<T>
		extends AbstractUdfStreamOperator<T, AssignerWithPeriodicWatermarks<T>>
		implements OneInputStreamOperator<T, T>, ProcessingTimeCallback {
	private transient long watermarkInterval;
	private transient long currentWatermark;		

  //可以看到在processElement会调用AssignerWithPeriodicWatermarks.extractTimestamp提取event time, 然后更新StreamRecord的时间。
	@Override
	public void processElement(StreamRecord<T> element) throws Exception {
		final long newTimestamp = userFunction.extractTimestamp(element.getValue(),
				element.hasTimestamp() ? element.getTimestamp() : Long.MIN_VALUE);

		output.collect(element.replace(element.getValue(), newTimestamp));
	}

	@Override
	public void onProcessingTime(long timestamp) throws Exception {
		// register next timer
		Watermark newWatermark = userFunction.getCurrentWatermark(); //定时调用用户自定义的getCurrentWatermark
		if (newWatermark != null && newWatermark.getTimestamp() > currentWatermark) {
			currentWatermark = newWatermark.getTimestamp();
			// emit watermark
			output.emitWatermark(newWatermark);
		}

		long now = getProcessingTimeService().getCurrentProcessingTime();
		getProcessingTimeService().registerTimer(now + watermarkInterval, this);
	}

	@Override
	public void processWatermark(Watermark mark) throws Exception {
		// if we receive a Long.MAX_VALUE watermark we forward it since it is used
		// to signal the end of input and to not block watermark progress downstream
		if (mark.getTimestamp() == Long.MAX_VALUE && currentWatermark != Long.MAX_VALUE) {
			currentWatermark = Long.MAX_VALUE;
			output.emitWatermark(mark);
		}
	}  
}	
7. WindowOperator的实现
最后的 .keyBy(0) .timeWindow(Time.seconds(10)) 是由 WindowOperator处理。

Flink通过水位线分配器（TimestampsAndPeriodicWatermarksOperator和TimestampsAndPunctuatedWatermarksOperator这两个算子）向事件流中注入水位线。元素在streaming dataflow引擎中流动到WindowOperator时，会被分为两拨，分别是普通事件和水位线。

如果是普通的事件，则会调用processElement方法进行处理，在processElement方法中，首先会利用窗口分配器为当前接收到的元素分配窗口，接着会调用触发器的onElement方法进行逐元素触发。对于时间相关的触发器，通常会注册事件时间或者处理时间定时器，这些定时器会被存储在WindowOperator的处理时间定时器队列和水位线定时器队列中，如果触发的结果是FIRE，则对窗口进行计算。

如果是水位线（事件时间场景），则方法processWatermark将会被调用，它将会处理水位线定时器队列中的定时器。如果时间戳满足条件，则利用触发器的onEventTime方法进行处理。processWatermark 用来处理上游发送过来的watermark，可以认为不做任何处理，下游的watermark只与其上游最近的生成方式相关。

WindowOperator内部有触发器上下文对象接口的实现——Context，它主要提供了三种类型的方法：

提供状态存储与访问；
定时器的注册与删除；
窗口触发器process系列方法的包装；
在注册定时器时，会新建定时器对象并将其加入到定时器队列中。等到时间相关的处理方法（processWatermark和trigger）被触发调用，则会从定时器队列中消费定时器对象并调用窗口触发器，然后根据触发结果来判断是否触动窗口的计算。

@Internal
public class WindowOperator<K, IN, ACC, OUT, W extends Window>
	extends AbstractUdfStreamOperator<OUT, InternalWindowFunction<ACC, OUT, K, W>>
	implements OneInputStreamOperator<IN, OUT>, Triggerable<K, W> {

  protected final WindowAssigner<? super IN, W> windowAssigner;
	protected transient TimestampedCollector<OUT> timestampedCollector;
	protected transient Context triggerContext = new Context(null, null); //触发器上下文对象
	protected transient WindowContext processContext;
	protected transient WindowAssigner.WindowAssignerContext windowAssignerContext;
无论是windowOperator还是KeyedProcessOperator都持有InternalTimerService具体实现的对象，通过这个对象用户可以注册EventTime及ProcessTime的timer，当watermark 越过这些timer的时候，调用回调函数执行一定的操作。

window operator通过WindowAssigner和Trigger来实现它的逻辑。当一个element到达时，通过KeySelector先assign一个key，并且通过WindowAssigner assign若干个windows（指定element分配到哪个window去），这样这个element会被放入若干个pane。一个pane会存放所有相同key和相同window的elements。

比如 SlidingEventTimeWindows 的实现。

* public class SlidingEventTimeWindows extends WindowAssigner<Object, TimeWindow>
  
	Collection<TimeWindow> assignWindows(Object element, long timestamp, ...) {
			List<TimeWindow> windows = new ArrayList<>((int) (size / slide));
			long lastStart = TimeWindow.getWindowStartWithOffset(timestamp, offset, slide);
			for (long start = lastStart;
				start > timestamp - size;
				start -= slide) {
        //可以看到这里会assign多个TimeWindow，因为是slide
				windows.add(new TimeWindow(start, start + size));
			}
			return windows;
	}
再比如 TumblingProcessingTimeWindows

public class TumblingProcessingTimeWindows extends WindowAssigner<Object, TimeWindow> {

Collection<TimeWindow> assignWindows(Object element, long timestamp, ...) {
      final long now = context.getCurrentProcessingTime();
      long start = now - (now % size);
      //很简单，分配一个TimeWindow
      return Collections.singletonList(new TimeWindow(start, start + size)); 
}
processWatermark
首先看看处理Watermark

public void processWatermark(Watermark mark) throws Exception {
    //定义一个标识，表示是否仍有定时器满足触发条件   
    boolean fire;   
    do {
        //从水位线定时器队列中查找队首的一个定时器，注意此处并不是出队（注意跟remove方法的区别）      
        Timer<k, w=""> timer = watermarkTimersQueue.peek();      
        //如果定时器存在，且其时间戳戳不大于水位线的时间戳
        //（注意理解条件是：不大于，水位线用于表示小于该时间戳的元素都已到达，所以所有不大于水位线的触发时间戳都该被触发）
        if (timer != null && timer.timestamp <= mark.getTimestamp()) {
            //置标识为真，表示找到满足触发条件的定时器         
            fire = true;         
            //将该元素从队首出队
            watermarkTimers.remove(timer);         
            watermarkTimersQueue.remove();
            //构建新的上下文         
            context.key = timer.key;         
            context.window = timer.window;         
            setKeyContext(timer.key);         
            //窗口所使用的状态存储类型为可追加的状态存储
            AppendingState<in, acc=""> windowState;         
            MergingWindowSet<w> mergingWindows = null;         
            //如果分配器是合并分配器（比如会话窗口）
            if (windowAssigner instanceof MergingWindowAssigner) {
                //获得合并窗口帮助类MergingWindowSet的实例            
                mergingWindows = getMergingWindowSet();            
                //获得当前窗口对应的状态窗口（状态窗口对应着状态后端存储的命名空间）
                W stateWindow = mergingWindows.getStateWindow(context.window);            
                //如果没有对应的状态窗口，则跳过本次循环
                if (stateWindow == null) {                              
                    continue;            
                }
                //获得当前窗口对应的状态表示            
                windowState = getPartitionedState(stateWindow, 
                    windowSerializer, windowStateDescriptor);         
            } else {
                //如果不是合并分配器，则直接获取窗口对应的状态表示            
                windowState = getPartitionedState(context.window, 
                    windowSerializer, windowStateDescriptor);         
            }
            //从窗口状态表示中获得窗口中所有的元素         
            ACC contents = windowState.get();         
            if (contents == null) {            
                // if we have no state, there is nothing to do            
                continue;         
            }
            //通过上下文对象调用窗口触发器的事件时间处理方法并获得触发结果对象
            TriggerResult triggerResult = context.onEventTime(timer.timestamp);         
            //如果触发的结果是FIRE（触动窗口计算），则调用fire方法进行窗口计算
            if (triggerResult.isFire()) {            
                fire(context.window, contents);         
            }
            //而如果触动的结果是清理窗口，或者事件时间等于窗口的清理时间（通常为窗口的maxTimestamp属性）         
            if (triggerResult.isPurge() || 
                (windowAssigner.isEventTime() 
                    && isCleanupTime(context.window, timer.timestamp))) {
                //清理窗口及元素            
                cleanup(context.window, windowState, mergingWindows);         
            }      
        } else {
            //队列中没有符合条件的定时器，置标识为否，终止循环         
            fire = false;      
        }   
    } while (fire);   
    //向下游发射水位线,把waterMark传递下去
    output.emitWatermark(mark);   
    //更新currentWaterMark, 将当前算子的水位线属性用新水位线的时间戳覆盖
    this.currentWatermark = mark.getTimestamp();
}
以上方法虽然冗长但流程还算清晰，其中的fire方法用于对窗口进行计算，它会调用内部窗口函数（即InternalWindowFunction，它包装了WindowFunction）的apply方法。

processElement
处理element到达的逻辑，将当前的element的value加到对应的window中，触发onElement

public void processElement(StreamRecord<IN> element) throws Exception {
    Collection<W> elementWindows = windowAssigner.assignWindows(  //通过WindowAssigner为element分配一系列windows
        element.getValue(), element.getTimestamp(), windowAssignerContext);

    final K key = (K) getStateBackend().getCurrentKey();

    if (windowAssigner instanceof MergingWindowAssigner) { //如果是MergingWindow
        //.......
    } else { //如果是普通window
        for (W window: elementWindows) {

            // drop if the window is already late
            if (isLate(window)) { //late data的处理，默认是丢弃  
                continue;
            }

            AppendingState<IN, ACC> windowState = getPartitionedState( //从backend中取出该window的状态，就是buffer的element
                window, windowSerializer, windowStateDescriptor);
            windowState.add(element.getValue()); //把当前的element加入buffer state

            context.key = key;
            context.window = window; //context的设计相当tricky和晦涩

            TriggerResult triggerResult = context.onElement(element); //触发onElment，得到triggerResult

            if (triggerResult.isFire()) { //对triggerResult做各种处理
                ACC contents = windowState.get();
                if (contents == null) {
                    continue;
                }
                fire(window, contents); //如果fire，真正去计算窗口中的elements
            }

            if (triggerResult.isPurge()) {
                cleanup(window, windowState, null); //purge，即去cleanup elements
            } else {
                registerCleanupTimer(window);
            }
        }
    }
}
判断是否是late data的逻辑

protected boolean isLate(W window) {
    return (windowAssigner.isEventTime() && (cleanupTime(window) <= currentWatermark));
}
而isCleanupTime和cleanup这对方法主要涉及到窗口的清理。如果当前窗口是时间窗口，且窗口的时间到达了清理时间，则会进行清理窗口清理。那么清理时间如何判断呢？Flink是通过窗口的最大时间戳属性结合允许延迟的时间联合计算的

private long cleanupTime(W window) {
    //清理时间被预置为窗口的最大时间戳加上允许的延迟事件   
    long cleanupTime = window.maxTimestamp() + allowedLateness;
    //如果窗口为非时间窗口（其maxTimestamp属性值为Long.MAX_VALUE），则其加上允许延迟的时间，
    //会造成Long溢出，从而会变成负数，导致cleanupTime < window.maxTimestamp 条件成立，
    //则直接将清理时间设置为Long.MAX_VALUE   
    return cleanupTime >= window.maxTimestamp() ? cleanupTime : Long.MAX_VALUE;
}
trigger
这个是用来触发onProcessingTime，这个需要依赖系统时间的定时器来触发，逻辑和processWatermark基本等同，只是触发条件不一样

@Override
public void trigger(long time) throws Exception {
    boolean fire;

    //Remove information about the triggering task
    processingTimeTimerFutures.remove(time);
    processingTimeTimerTimestamps.remove(time, processingTimeTimerTimestamps.count(time));

    do {
        Timer<K, W> timer = processingTimeTimersQueue.peek();
        if (timer != null && timer.timestamp <= time) {
            fire = true;

            processingTimeTimers.remove(timer);
            processingTimeTimersQueue.remove();

            context.key = timer.key;
            context.window = timer.window;
            setKeyContext(timer.key);

            AppendingState<IN, ACC> windowState;
            MergingWindowSet<W> mergingWindows = null;

            if (windowAssigner instanceof MergingWindowAssigner) {
                mergingWindows = getMergingWindowSet();
                W stateWindow = mergingWindows.getStateWindow(context.window);
                if (stateWindow == null) {
                    // then the window is already purged and this is a cleanup
                    // timer set due to allowed lateness that has nothing to clean,
                    // so it is safe to just ignore
                    continue;
                }
                windowState = getPartitionedState(stateWindow, windowSerializer, windowStateDescriptor);
            } else {
                windowState = getPartitionedState(context.window, windowSerializer, windowStateDescriptor);
            }

            ACC contents = windowState.get();
            if (contents == null) {
                // if we have no state, there is nothing to do
                continue;
            }

            TriggerResult triggerResult = context.onProcessingTime(timer.timestamp);
            if (triggerResult.isFire()) {
                fire(context.window, contents);
            }

            if (triggerResult.isPurge() || (!windowAssigner.isEventTime() && isCleanupTime(context.window, timer.timestamp))) {
                cleanup(context.window, windowState, mergingWindows);
            }

        } else {
            fire = false;
        }
    } while (fire);
}
0x08 参考
Flink - watermark

Stream指南七 理解事件时间与Watermarks

Flink运行时之流处理程序生成流图

Flink原理与实现：如何生成ExecutionGraph及物理执行图

Flink timer注册与watermark触发

Apache Flink源码解析 （四）Stream Operator

Flink流处理之窗口算子分析

Flink 原理与实现：如何生成 StreamGraph

Flink中Watermark定时生成源码分析

追源索骥：透过源码看懂Flink核心框架的执行流程

Flink运行时之流处理程序生成流图

Apache Flink 进阶（六）：Flink 作业执行深度解析

调试Windows和事件时间

Flink最佳实践（二）Flink流式计算系统

Streaming System 第三章：Watermarks

Apache Flink源码解析之stream-source

Flink源码系列——Flink中一个简单的数据处理功能的实现过程

Flink中task之间的数据交换机制

Flink task之间的数据交换

[Flink架构（二）- Flink中的数据传输]()

Flink的数据抽象及数据交换过程

聊聊flink的Execution Plan Visualization

Flink 原理与实现：如何生成 StreamGraph

Flink源码系列——获取StreamGraph的过程

Flink源码系列——Flink中一个简单的数据处理功能的实现过程

Flink源码解读系列1——分析一个简单Flink程序的执行过程

Flink timer注册与watermark触发[转载自网易云音乐实时计算平台经典实践知乎专栏]

[Flink – process watermark](https://cnblogs.com/fxjwind/p/7657058.html)

Flink流计算编程--Flink中allowedLateness详细介绍及思考

「Spark-2.2.0」Structured Streaming - Watermarking操作详解

Flink window机制

Flink – window operator

flink的window计算、watermark、allowedLateness、trigger

Apache Flink源码解析 （四）Stream Operator

Flink - watermark生成

Flink入门教程--Task Lifecycle(任务的生命周期简介)

Flink 原理与实现：Operator Chain原理

Flink算子的生命周期

Flink原理（三）——Task（任务）、Operator Chain（算子链）和Slot（资源）

Flink – Stream Task执行过程

Flink Slot详解与Job Execution Graph优化

★★★★★★关于生活和技术的思考★★★★★★
微信公众账号：罗西的思考
如果您想及时得到个人撰写文章的消息推送，或者想看看个人推荐的技术资料，可以扫描下面二维码（或者长按识别二维码）关注个人公众号）。

分类: 002_大数据, 005_源码分析, 201_Flink
好文要顶 关注我 收藏该文  
罗西的思考
关注 - 0
粉丝 - 8
+加关注
00
« 上一篇： [白话解析] Flink的Watermark机制
» 下一篇： [白话解析]以水浒传为例学习隐马尔可夫模型
posted @ 2020-02-22 17:13  罗西的思考  阅读(506)  评论(0)  编辑  收藏
刷新评论刷新页面返回顶部
注册用户登录后才能发表评论，请 登录 或 注册， 访问 网站首页。
【推荐】了解你才能更懂你，博客园首发问卷调查，助力社区新升级
【推荐】超50万行VC++源码: 大型组态工控、电力仿真CAD与GIS源码库
【推荐】深度回顾！30篇好文，解析历年双十一背后的阿里技术秘籍
公告
★关于生活和技术的思考★
欢迎关注公众号,您将会得到及时的文章推送信息：

昵称： 罗西的思考
园龄： 8个月
粉丝： 8
关注： 0
+加关注
<	2020年6月	>
日	一	二	三	四	五	六
31	1	2	3	4	5	6
7	8	9	10	11	12	13
14	15	16	17	18	19	20
21	22	23	24	25	26	27
28	29	30	1	2	3	4
5	6	7	8	9	10	11
搜索
 
 
常用链接
我的随笔
我的评论
我的参与
最新评论
我的标签
随笔分类
001_机器学习(20)
002_大数据(25)
003_白话解析(17)
004_项目记录(7)
005_源码分析(17)
006_Android(1)
007_梁山好汉说IT(18)
100_Java(2)
101_Lua(3)
103_Python(1)
200_Alink(8)
201_Flink(10)
202_Ionic(1)
203_Redis(1)
204_Hadoop(1)
205_OpenResty(1)
206_Oozie(1)
随笔档案
2020年6月(16)
2020年5月(6)
2020年4月(4)
2020年3月(5)
2020年2月(3)
2020年1月(6)
2019年12月(1)
2019年11月(5)
2019年10月(1)
最新评论
1. Re:Alink漫谈(八) : 二分类评估 AUC、K-S、PRC、Precision、Recall、LiftChart 如何实现
感谢分享！！！


--柠檬电影观看
2. Re:Alink漫谈(一) : 从KMeans算法实现不同看Alink设计思想
关注了，也在调研Alink，期望后文


--Lemon丶
3. Re:[白话解析] 深入浅出朴素贝叶斯模型原理及应用
good article.
--拥剑公子
4. Re:[梁山好汉说IT] 以水浒传为例讲解贝叶斯定理
不明觉厉
--xing.org1^
阅读排行榜
1. [白话解析] Flink的Watermark机制(1436)
2. [白话解析] 深入浅出最大熵模型(1106)
3. [源码分析] 从FlatMap用法到Flink的内部实现(531)
4. [源码分析] 从源码入手看 Flink Watermark 之传播过程(505)
5. [白话解析] 深入浅出朴素贝叶斯模型原理及应用(338)
评论排行榜
1. [白话解析] 深入浅出贝叶斯定理(1)
2. [白话解析] 深入浅出朴素贝叶斯模型原理及应用(1)
3. Alink漫谈(一) : 从KMeans算法实现不同看Alink设计思想(1)
4. Alink漫谈(八) : 二分类评估 AUC、K-S、PRC、Precision、Recall、LiftChart 如何实现(1)
推荐排行榜
1. [白话解析]以水浒传为例学习隐马尔可夫模型(2)
2. [源码分析] 从实例和源码入手看 Flink 之广播 Broadcast(1)
3. [白话解析] 通俗解析集成学习之bagging，boosting & 随机森林(1)
4. Alink漫谈(一) : 从KMeans算法实现不同看Alink设计思想(1)
5. [Alink漫谈之三] AllReduce通信模型(1)
Copyright © 2020 罗西的思考
Powered by .NET Core on Kubernetes
