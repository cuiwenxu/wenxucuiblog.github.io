@[TOC](Insufficient number of network buffers)
# 报错日志&截图
![在这里插入图片描述](https://img-blog.csdnimg.cn/18dccbad311d49cb8512c4c7c071d3d1.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)

# 问题原因

task在申请MemorySegment时获取不到所需的数量，抛出IOException，detail message为Insufficient number of network buffers

抛出异常的方法为NetworkBufferPool.tryRedistributeBuffers

private void tryRedistributeBuffers(int numberOfSegmentsToRequest) throws IOException {
        assert Thread.holdsLock(factoryLock);

        if (numTotalRequiredBuffers + numberOfSegmentsToRequest > totalNumberOfMemorySegments) {
            throw new IOException(
                    String.format(
                            "Insufficient number of network buffers: "
                                    + "required %d, but only %d available. %s.",
                            numberOfSegmentsToRequest,
                            totalNumberOfMemorySegments - numTotalRequiredBuffers,
                            getConfigDescription()));
        }

        this.numTotalRequiredBuffers += numberOfSegmentsToRequest;

        try {
            redistributeBuffers();
        } catch (Throwable t) {
            this.numTotalRequiredBuffers -= numberOfSegmentsToRequest;

            redistributeBuffers();
            ExceptionUtils.rethrow(t);
        }
}

该异常常出现在调整flink 任务并发度之后或者设置unchainedOperator=true之后，任务所需network buffer数量增加导致。

# 解决方案

调整flink-conf.yarm中控制network buffer大小的参数，1.12参数列表如下

参数     | 含义|默认值
-------- | ----- | -----
containerized.heap-cutoff-ratio  | JVM cut-off 部分使用的内存 | 0.25
taskmanager.network.memory.fraction |  Network Buffer 使用的内存 | 0.1
taskmanager.memory.segment-size | Network Buffer 的大小 | 32kb
taskmanager.memory.network.min |  Network Buffer 的最小值 | 64mb
taskmanager.memory.network.max | Network Buffer 的最大值 | 64mb

不同版本中的参数名称和默认值可能不一致，以上述默认参数为例，假如我们有一个 2g 的 TaskManager，那么各部分对应的内存数值为：
>JVM cut-off 内存 = 2g * containerized.heap-cutoff-ratio
JVM heap 内存 = (2g - JVM cut-off 内存) * (1 - taskmanager.network.memory.fraction)
JVM non-heap 内存 = 2g - JVM heap 内存
Network Buffer 内存 = (2g - JVM cut-off 内存) * taskmanager.network.memory.fraction
Network Segments 个数 = Network Buffer 内存 / taskmanager.memory.segment-size

 那么，那么如何确定参数调整为多少呢？有两种方案

## 方案一：通过任务拓扑计算所需network buffer大小：

TaskManager所需的network buffer的数量需要为发送端+接收端之和，其中：

- 发送端共享一个 LocalBufferPool，总大小为 subpartitions + 1，subpartition 是这个 task 数据流向下游 task 的通道，如果和下游连接方式是 HASH 或者 REBALANCE，那么 subpartition 数量为下游 Task 数。

- 接收端的 Network Buffer 数量 = channel数 * taskmanager.network.memory.buffers-per-channel + taskmanager.network.memory.floating-buffers-per-gate，channel 数是接收上游数据的通道，如果和下游连接方式是 HASH 或者 REBALANCE，那么 channels 数量为上游 Task 数。

参数     | 含义|默认值
-------- | ----- | -----
taskmanager.network.memory.buffers-per-channel  | 每个 channel 独享的 buffer 数 | 2
taskmanager.network.memory.floating-buffers-per-gate |  Floating Buffer 数量 | 8


将 TaskManager 上所有 Task 的 Network Buffer 数相加，即可得到 TaskManager 内实际需要的 Network Buffer 数量。然后调整taskmanager.network.memory.fraction

## 方案二：根据经验直接调整flink-conf.yaml文件中的taskmanager.memory.network.fraction参数

在不了解任务拓扑的情况下，根据经验值调整，并观察作业运行情况



>network buffer数量计算方式引用
http://www.liaojiayi.com/flink-network-buffer/




