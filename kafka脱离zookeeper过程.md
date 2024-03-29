当前，ApacheKafka®使用Apache ZooKeeper™来存储其元数据。诸如分区位置和主题配置之类的数据存储在Kafka之外的单独ZooKeeper集群中。在2019年，我们概述了一项计划来打破这种依赖性并将元数据管理引入Kafka本身。
以下几点导致我们不再用zookeeper来维护kafka元数据。
1、有两个系统会导致很多重复。Kafka是复制的分布式日志，其顶部是pub/sub API。 ZooKeeper是一个复制的分布式日志，顶部是文件系统API。每个都有其自己的网络通信，安全性，监视和配置方式。使用两个系统会使总复杂度翻倍。这导致不必要的陡峭学习曲线，并增加了某些配置错误导致安全漏洞的风险。
2、在外部存储元数据限制了Kafka的可扩展性。当Kafka集群启动时，或者正在选择新的控制器时，该控制器必须从ZooKeeper加载集群的完整状态。随着元数据量的增加，此加载过程的时间也随之增加。这限制了Kafka可以存储的分区数量。

最终，将元数据存储在外部会增加控制器的内存状态与外部状态不同步的可能性。控制器的活动视图（位于群集中）也可以与ZooKeeper的视图不同。
