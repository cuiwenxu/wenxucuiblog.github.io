此页面提供了如何配置和调优大状态应用程序的指南。

# 概述
Flink应用要想大规模可靠运行，必须满足两个条件:
- 应用程序需要能够可靠地使用检查点
- 在出现故障后，资源需要足够跟上输入数据流

第一部分讨论了如何在规模上获得良好的执行检查点。最后一节解释了一些关于规划使用多少资源的最佳实践。
Monitoring State and Checkpoints #
The easiest way to monitor checkpoint behavior is via the UI’s checkpoint section. The documentation for checkpoint monitoring shows how to access the available checkpoint metrics.

The two numbers (both exposed via Task level metrics and in the web interface) that are of particular interest when scaling up checkpoints are:

The time until operators receive their first checkpoint barrier When the time to trigger the checkpoint is constantly very high, it means that the checkpoint barriers need a long time to travel from the source to the operators. That typically indicates that the system is operating under a constant backpressure.

The alignment duration, which is defined as the time between receiving first and the last checkpoint barrier. During unaligned exactly-once checkpoints and at-least-once checkpoints subtasks are processing all of the data from the upstream subtasks without any interruptions. However with aligned exatcly-once checkpoints, the channels that have already received a checkpoint barrier are blocked from sending further data until all of the remaining channels catch up and receive theirs checkpoint barriers (alignment time).

Both of those values should ideally be low - higher amounts means that checkpoint barriers traveling through the job graph slowly, due to some back-pressure (not enough resources to process the incoming records). This can also be observed via increased end-to-end latency of processed records. Note that those numbers can be occasionally high in the presence of a transient backpressure, data skew, or network issues.

Unaligned checkpoints can be used to speed up the propagation time of the checkpoint barriers. However please note, that this does not solve the underlying problem that’s causing the backpressure in the first place (and end-to-end records latency will remain high).
# 监控状态和检查点
监视检查点行为的最简单方法是通过UI的checkpoint部分。

当checkpoint大小调整时，有两个数字特别有趣:

当触发检查点的时间一直很高时，意味着checkpoint barrier从source到operator需要很长时间。这通常表明系统在恒定的反压力下运行。

对齐持续时间，定义为接收第一个检查点和最后一个检查点屏障之间的时间。在未对齐一次检查点和至少一次检查点期间，子任务将处理来自上游子任务的所有数据，而不会有任何中断。但是，如果使用精确对齐的检查点，那么已经接收到检查点屏障的通道将被阻止发送进一步的数据，直到所有剩余的通道赶上并接收到它们的检查点屏障(对齐时间)。

理想情况下，这两个值都应该是较低的—较高的值意味着检查点壁垒在作业图中移动得很慢，这是因为一些回压(没有足够的资源来处理传入的记录)。这也可以通过处理记录的端到端延迟的增加来观察到。请注意，在出现瞬态反压、数据倾斜或网络问题时，这些数字偶尔会很高。

未对齐的检查点可用于加快检查点壁垒的传播时间。但是请注意，这并不能首先解决导致回压的潜在问题(并且端到端记录延迟仍然很高)。

