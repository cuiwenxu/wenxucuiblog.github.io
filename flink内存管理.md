### 概述
flink==积极==的内存管理体现在：
- 定义MemorySegment——不直接使用JVM的byte，而是将jvm的堆内内存&堆外内存统一用MemorySegment管理，这样的好处在于：1、可以使用堆外内存；2、将jvm堆内存分成固定大小的page后，可以配合序列化和类型信息系统(type infomation system)在二进制基础上快速操作(而不是先反序列化为对象然后再操作)
- 在JVM 内存结构规划上，Flink 也作了相应的调整：将JVM堆分为三个区域，并且将MemoryManager 和 Network Buffers 两个实现了显式内存管理的子系统分配到老年代，而留给用户代码的 Free 区域分配到新生代。显而易见，因为所有常驻型数据都以二进制的形式存在 Flink 的MemoryManager中，这些MemorySegment一直呆在老年代而不会被GC回收。其他的数据对象基本上是由用户代码生成的短生命周期对象，这部分对象可以被 Minor GC 快速回收。只要用户不去创建大量类似缓存的常驻型对象，那么老年代的大小是不会变的，Major GC也就永远不会发生。从而有效地降低了垃圾回收的压力。另外，这里的内存块还可以是堆外内存，这可以使得 JVM 内存更小，从而加速垃圾回收。

### 细节
##### Taskmanager将JVM堆分为三个区域：
- 网络缓冲区(network buffer)：网络堆栈使用若干32 KB的buffer来缓冲网络传输的记录。 在TaskManager启动时分配。 默认情况下，使用2048个缓冲区(64M)，但可以通过`taskmanager.network.numberOfBuffers`进行调整。
- 内存管理器池(MemoryManager pool)：大量的buffer(32 KiBytes)，所有运行时需要缓冲记录时都会使用它们。 记录以序列化形式存储在这些块中。内存管理器在启动时分配这些缓冲区。
- 剩余堆(remaining heap)：堆的这一部分留给用户代码和TaskManager的数据结构。 由于这些数据结构非常小，因此该内存大部分可供用户代码使用。   
![flink 内存区域划分](https://img-blog.csdnimg.cn/20200902110152302.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70#pic_center)
在分配network buffer和Memory Manager pool缓冲区时，JVM通常执行一个或多个完整的垃圾回收。
这为TaskManager的启动增加了一些时间，但在以后执行任务时节省了垃圾回收时间。__网络缓冲区和内存管理器缓冲区在TaskManager的整个生命周期中都有效。它们存在于JVM的终身代(tenured generation)，属于长期存在的不会被收集的对象。__

NOTE：
- 可以通过“ taskmanager.network.bufferSizeInBytes”来调整缓冲区的大小，但是对于大多数设置而言，32K似乎是一个不错的大小。
- 关于如何统一网络缓冲池和内存管理器区域的想法。有想法可以添加一种模式，以使由MemoryManager延迟分配内(在需要时分配)。 这减少了TaskManager的启动时间，但稍后在实际分配缓冲区时将导致更多垃圾回收。

##### Memory Segments
__Flink将其所有内存表示为Memory Segment的集合__。 该段代表一个内存区域(默认为32 KiBytes，即flink内存三类区域的最小单位)，提供以偏移量访问数据的方法(获取和放置long，int，bytes，在内存段和数组之间复制...)。
您可以将其视为专门用于Flink的java.nio.ByteBuffer的版本(请参见下文，为什么我们不使用java.nio.ByteBuffer)。
__每当Flink将记录存储在某处时，它实际上会将其序列化为一个或多个内存段。 系统可以将记录对应的“指针”存储在另一个数据结构中(通常也构建为内存段的集合)。这意味着Flink依赖于有效的序列化(类似于C语言用指针来管理内存)，该序列可识别页面和跨页面的中断记录。为了实现类似C的内存管理机制，Flink必须有配套的数据类型和序列化机制。__

![record序列化后在memory segment中的存储方式](https://img-blog.csdnimg.cn/20200902111755709.png#pic_center)

##### 内存段与字节缓冲区
为什么Flink不直接使用java.nio.ByteBuffer？与ByteBuffer相比，MemorySegment有一些好处：
- MemorySegment使用sun.misc.Unsafe方法，因此像long这样的类型的获取成本低很多（无需移位）。
- MemorySegment只有绝对的get / put方法，并且所有类型都具有绝对方法，这使其线程安全。 java.nio.ByteBuffer则没有字节数组和其他ByteBuffer的绝对get / put，迫使您调用需要锁定或放弃线程安全性的相对方法。
- 由于我们只有一个MemorySegment的最终实现类，因此JIT可以对方法进行了更好的虚拟化和内联处理，而ByteBuffer至少存在5种不同的实现。

#### 对垃圾收集的影响
这种使用内存的机制对Flink的垃圾回收行为具有良好的影响。
Flink不会将任何记录保存为对象，而是将它们序列化后存储在长期存在的缓冲区中。那意味着实际上没有长期有效的记录——记录只存在于通过用户功能传递并序列化为内存段。寿命长的对象就是内存段本身，它们永远不会被垃圾回收。

__运行Flink任务时，JVM将在“新一代”(对应于flink三类内存中的剩余堆内存，用于存放user function和taskmanager数据结构)中的短期对象上执行许多垃圾回收。这些垃圾收集通常代价很小。终身代垃圾收集(长期且昂贵，对应于flink三类内存中的MemoryManager pool和network buffer)很少发生。__

 ![JVM内存和flink内存对应关系](https://img-blog.csdnimg.cn/20200902145923116.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70#pic_center)
为了证明 Flink 内存管理和序列化器的优势，Flink 官方对 Object-on-Heap (直接 Java 对象存储)、Flink-Serialized (内建序列化器 + 显式内存管理)和 Kryo-Serialized (Kryo 序列化器 + 显式内存管理)三种方案进行了 GC 表现的对比测试。

测试方法是对一千万个 Tuple2<Integer, String> 对象进行排序，其中 Integer 字段的值是均匀分布的，String 字段是长度为12的字符串并服从长尾分布。测试的作业跑在 heap 为900MB的 JVM 内，这恰好是排序一千万个对象所需要的最低内存。

测试在 JVM GC 上的表现如下图:
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200902205524524.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70#pic_center)


显而易见，使用显式内存管理可以显著地减少 GC 频率。在 Object-on-Heap 的测试中，GC 频繁地被触发并导致 CPU 峰值达到90%。在测试中使用的8核机器上单线程的作业最多只占用12.5%的 CPU ，机器花费在 GC 的成本显然超过了实际运行作业的成本。而在另外两个依赖显式内存管理和序列化的测试中，GC 很少被触发，CPU 使用率也一直稳定在较低的水平。
### 具体的调优参数 
有两个参数可供调优
- 1、jvm级别的参数 `-XX:NewRatio`
__当OldGen(老年代)堆的大小与network buffer和memory manager pool的总大小匹配时效果最佳__。此比率由JVM选项-XX:NewRatio控制，该选项定义了与NewGen与OldGen倍数关系。默认情况下，Flink使用OldGen两倍于NewGen的设置(-XX:NewRatio = 2，较新的GC上的JVM默认设置)，而MemoryManager和NetworkBuffers使用70％的堆。这应该会使MemoryManager pool重叠(因为按照-XX:NewRatio=2，network buffer和MemoryManager pool约占66.6%)。
- 2、Flink的参数` taskmanager.memory.fraction`或`taskmanager.memory.size`
Flink管理的内存量可以通过两种方式配置
    + 相对值(默认模式)：在该模式下，MemoryManager将评估在启动所有其他TaskManager服务之后剩余多少堆空间。然后它将分配该空间的特定比例（默认为0.7）作为flink管理的内存页面。 比例可以通过	` taskmanager.memory.fraction`指定。
    + 绝对值：在flink-conf.yaml中指定`taskmanager.memory.size`时，MemoryManager在启动时将分配那么多的内存作为flink管理页。


