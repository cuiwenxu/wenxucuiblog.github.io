>基于kafka的实时数仓可以达到秒级别延迟（多层，如果是单层可达到ms级别延迟），但是kafka的成本太高，如果要做到近实时的数仓，可用iceberg+hdfs替代kafka。

以上这段是很多公司用iceberg替换kafka的原因，通过下面两个问题问清楚成本高在哪


>Q1:存放同样大小1pb的数据，kafka成本为什么比hdfs高？

>A1:kafka是按消息队列设计的，为了满足低延迟，而采用了pagecache层（非常激进的使用内存，因为一旦读数据miss cache，会产生磁盘读操作，这种情况下速度会大幅度降低），所以用于搭建kafka集群的机器内存一般比较大，所以成本较高


>Q2:如果对延迟要求放宽，存放同样大小1pb的数据，kafka成本能否比hdfs低或持平？


>A2:首先这是一个工程设计的问题，kafka设计角色是一个消息队列，而hdfs是大数据量存储系统，两者针对各自应用场景都做了很多优化，比如从hdfs读数据会从多个块同时读，而kafka不会，所以即使对延迟要求放宽，存放同样大小1pb的数据，同样读取速度要求下kafka成本还是高于hdfs


