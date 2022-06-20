
一、Flume简单介绍
Flume提供一个分布式的，可靠的，对大数据量的日志进行高效收集、聚集、移动的服务

1.1 FLume的基本组成
Flume的组件主要包括三个:Source、Channel、Sink,其前后关系如图1所示。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200114210100238.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
图1
1.2 Source
Source用来从数据源接收数据,并将数据写入到Channel中。
Source分很多种，现在公司内用到的有两个:
（1）KafkaSource:其用于从Kafka中获取数据
（2）TailDirSource:其可以读取本地磁盘文件中的数据，TailDirSource会在本地维护一个position文件，根据文件名字和文件的inode来记录所消费到的文件中的偏移量来记录数据消费到了哪个位置。当Flume挂掉重启时，会从当前偏移量继续消费。(注意:经过vim编辑过的文件inode会变更)
（3）AvroSource:这个Source会监听当前服务器的指定的端口，数据发送到这个端口就会被AvroSource收集

1.3 Channel
Channel是一个被动组件，Source将数据写入到Channel,Sink从Channel中读取数据，其就像一个队列。
Channel有两种:
（1）FileChannel:其会将进入Channel的数据持久化到磁盘上，当flume挂掉时，保证数据不会丢失
（2）MemoryChannel:数据在Channel中只在内存中进行传输，虽然性能上相比FileChannel较高，但是当flume挂掉时，而Channel中还有数据没有被消费完时，这部分数据将会丢失。

1.4 Sink
Sink是一个agent的输出组件，其将数据输送到目的地。
（1）HdfsSink:数据目的地是hdfs的Sink
（2）AvroSink:数据目的地是指定端口号的Sink

1.5 三个组件的对应关系(类型)
数据源于Source:1对1
Source与Channel:1对多
Channel与Sink:1对多
Sink与数据目的地:1对1

二、Flume HA
Flume HA不是完全的HA,其不依赖于第三方组件如zookeeper
2.1 Flume HA存在的必要性
现在flume的应用场景有：flume采集ng日志数据到hdfs、flume采集kafka数据到hdfs、flume采集ng日志数据到kafka。那么，当flume采集数据的时候由于不可预测的原因突然挂掉怎么办，所以，为了保障数据采集链路的正常运行,搭建flume的HA的架构是非常有必要的。

2.2 FLume如何可以做到HA来提高稳定性
2.2.1 如果数据源是本地文件?
如果要做到HA的话,FLume就要采用双层架构,如图2所示
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200114210205145.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
图2
这个架构的关键在于第一层Flume的sink上。在第一层的的每个Flume的Channel上都会有两个Sink，这两个Sink消费的Channel的数据是负载均衡的，这两个Sink会分别向第二层的两个Collect发送数据。当第二层的FLume有其中一个挂了的时候，这个时候相应要给这个FLume发送数据的Sink就不会再发送数据了，数据都会集中在另一个Sink中，全部发送给第二层正常的Flume。
那么如果第一层的FLume有其中一个挂掉了怎么办。很遗憾，只能及时的重启它并保持正常。因为ng同样需要做负载均衡，如果第一层的FLume挂掉其中一个，那么打到对应ng日志服务器上的日志在FLume这一环节就会缺失。
2.2.2 如果数据源是Kafka?
对于数据源是Kafka的情况，有两种部署方式，第一种就是如上面提到的双层架构，那么第二种可以利用Kafka的消费者本身的特性。相比于第一种，第二种无疑更加简洁。
消费Kafka的数据的客服端成为Kafka的消费者，Kafka的消费者有一个重要的特性，对于同组的消费者来说，其消费的相同topic下的数据是互斥的，可以简单理解为，如果有同组的两个消费者去消费同一个topic的数据，在特定情况下(topic的分区是偶数个)，这两个消费者会平分消费topic中的数据。架构图如图3所示
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200114210239771.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
图3
同时启动多个FLume去消费Kafka的数据，当有其中一个Flume挂掉时，数据的消费权会重新分配而保证数据可以继续正常消费。

2.3 如此架构对数据有什么影响
(1)消费效率:由于每一层的FLume都是多个FLume在消费数据，所以消费数据的效率会提高。
(2)消费出的数据是否会有丢失或者重复:
1、对于数据源是Kafka的Flume来说,由于每当启动或者关闭Flume的时候,topic的分区都会重新分配，而由于数据是批量进入到Channel中，只有当数据完全进入到Channel中数据才会被标记为被消费，所以当数据有部分进入到Channel中而未被提交时发生了数据重新分配，那么这部分已经进入到Channel中的数据而未被提交的数据很有可能会重新被分配给另一个FLume，这部分数据就会重新被消费。最终的数据就会发生重复，但是由于时间差非常小，所以重复的数据可以忽略。
2、Flume自身用Channel的事务保证了数据不会丢失，其保证数据至少会被发送一次。Source在向Channel中put数据和Sink从Channel中take数据时，都是按照batch走的，那么当put或者take数据时就会启动事务，只有当数据完全put成功时事务才会被提交，不然就会回滚。同样的，只有当Sink确认数据安全到达目的地时，事务才会被提交，不然会回滚。

三、Q&A

Q:Flume数据一定会重复吗?
A:这个要视情况而定，由于Flume保证数据不丢失的机制，导致了当遇到特殊情况时数据会重新发送，比如下游存储系统(hdfs)运行过慢，这时就会有重复数据产生。

Q:Flume数据会丢失吗?
A:只要Flume不发生故障性异常，那么数据就不会丢失。如果说因为数据量过于庞大而造成丢失，这种情况还暂时没碰到过。
