@[TOC](jvm垃圾回收算法&垃圾回收器)
# 代垃圾收集器特点
- 所有新生代垃圾收集器，都使用复制算法，都会发生stop-the-world。由于绝大多数对象的生命周期通常比较短，在新生代被回收的可能性很大，新生代的垃圾回收通常可以回收大部分对象，因此采用复制算法效率更高。
- parnew和parallel scavenge区别
ParNew 与 Parallel Scavenge的一个主要区别是，ParNew可以与CMS进行搭配使用。
"ParNew" is a stop-the-world, copying collector which uses multiple GC threads.
It differs from "Parallel Scavenge" in that it has enhancements that make it usable with CMS. For example, "ParNew" does the synchronization needed so that it can run during the concurrent phases of CMS.
- 除了CMS，其他的老年代垃圾收集器GC时都是stop-the-world，都会在清理垃圾之后进行压缩整理。
- G1回收的范围是整个Java堆(包括新生代，老年代)，而前六种收集器回收的范围仅限于新生代或老年代。G1收集器是JDK1.7提供的一个新收集器，G1收集器基于“标记-整理”算法实现，也就是说不会产生内存碎片。
# jvm默认的垃圾回收器
1.8中老年代使用PS MarkSweep(类似Serial Old)，年轻代使用PS Scavenge
>注：在Parallel Scavenge收集器架构中本身有PS MarkSweep收集器来进行老年代收集，但由于PS MarkSweep与Serial Old实现非常接近，因此官方的许多资料都直接以Serial Old代替PS MarkSweep进行讲解。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210321232620189.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
# 几种比较推荐的垃圾回收器组合
- 1、基于低停顿时间的垃圾收集器
-XX:+UseConcMarkSweepGC（该参数隐式启用-XX:+UseParNewGC）
- 2、基于吞吐量优先的垃圾收集器
-XX:+UseParallelOldGC（该参数隐式启用-XX:+UseParallelGC）


