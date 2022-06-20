@[TOC](基于metric监控反压)
# 反压影响
反压除了会导致数据产出变慢，还会影响到两项指标: checkpoint 时长和 state 大小
- 前者是因为 checkpoint barrier 是不会越过普通数据的，数据处理被阻塞也会导致 checkpoint barrier 流经整个数据管道的时长变长，因而 checkpoint 总体时间（End to End Duration）变长。
- 后者是因为为保证 EOS（Exactly-Once-Semantics，准确一次），对于有两个以上输入管道的 Operator，checkpoint barrier 需要对齐（Alignment），接受到较快的输入管道的 barrier 后，它后面数据会被缓存起来但不处理，直到较慢的输入管道的 barrier 也到达，这些被缓存的数据会被放到state 里面，导致 checkpoint 变大。

这两个影响对于生产环境的作业来说是十分危险的，因为 checkpoint 是保证数据一致性的关键，checkpoint 时间变长有可能导致 checkpoint 超时失败，而 state 大小同样可能拖慢 checkpoint 甚至导致 OOM （使用 Heap-based StateBackend）或者物理内存使用超出容器资源（使用 RocksDBStateBackend）的稳定性问题。
# 定位反压节点
要解决反压首先要做的是定位到造成反压的节点，这主要有两种办法:
- 通过 Flink Web UI 自带的反压监控面板；
- 通过 Flink Task Metrics。
前者比较容易上手，适合简单分析，后者则提供了更加丰富的信息，适合用于监控系统。因为反压会向上游传导，这两种方式都要求我们从 Source 节点到 Sink 的逐一排查，直到找到造成反压的根源原因。下面分别介绍这两种办法。
## 反压监控面板
Flink Web UI 的反压监控提供了 SubTask 级别的反压监控，原理是通过周期性对 Task 线程的栈信息采样，得到线程被阻塞在请求 Buffer（意味着被下游队列阻塞）的频率来判断该节点是否处于反压状态。默认配置下，这个频率在 0.1 以下则为 OK，0.1 至 0.5 为 LOW，而超过 0.5 则为 HIGH。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021040216194263.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
如果处于反压状态，那么有两种可能性：

- 该节点的发送速率跟不上它的产生数据速率。这一般会发生在一条输入多条输出的 Operator（比如 flatmap）。
- 下游的节点接受速率较慢，通过反压机制限制了该节点的发送速率。

如果是第一种状况，那么该节点则为反压的根源节点，它是从 Source Task 到 Sink Task 的第一个出现反压的节点。如果是第二种情况，则需要继续排查下游节点。

>总体来看，如果我们找到第一个出现反压的节点，那么反压根源要么是就这个节点，要么是它紧接着的下游节点。

那么如果区分这两种状态呢？很遗憾只通过反压面板是无法直接判断的，我们还需要结合 Metrics 或者其他监控手段来定位。此外如果作业的节点数很多或者并行度很大，由于要采集所有 Task 的栈信息，反压面板的压力也会很大甚至不可用。
## Task Metrics

