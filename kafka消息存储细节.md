##### Kafka部分名词解释如下：

Broker：消息中间件处理结点，一个Kafka节点就是一个broker，多个broker可以组成一个Kafka集群。
Topic：一类消息，例如page view日志、click日志等都可以以topic的形式存在，Kafka集群能够同时负责多个topic的分发。
Partition：topic物理上的分组，一个topic可以分为多个partition，每个partition是一个有序的队列。
Segment：partition物理上由多个segment组成
offset：每个partition都由一系列有序的、不可变的消息组成，这些消息被连续的追加到partition中。partition中的每个消息都有一个连续的序列号叫做offset,用于partition唯一标识一条消息.

- topic、partition、segment的关系
一个topic可以有多个partition，partition分为多个segment，其中partition对应物理结构上的文件夹。partition的命名为topic-parition序号。segement对应.index(偏移量索引文件，记录offset和文件磁盘地址偏移量).timeindex(时间戳索引文件，记录时间戳和offset的映射)和.log(日志数据文件)，下面分别介绍这三种文件
##### 日志数据文件
Kafka将生产者发送给它的消息数据内容保存至日志数据文件中，该文件以该段的基准偏移量左补齐0命名，文件后缀为“.log”。分区中的每条message由offset来表示它在这个分区中的偏移量，这个offset并不是该Message在分区中实际存储位置，而是逻辑上的一个值（Kafka中用8字节长度来记录这个偏移量），但它却唯一确定了分区中一条Message的逻辑位置，同一个分区下的消息偏移量按照顺序递增（这个可以类比下数据库的自增主键）。另外，从dump出来的日志数据文件的字符值中可以看到消息体的各个字段的内容值。
##### 偏移量索引文件
如果消息的消费者每次fetch都需要从1G大小（默认值）的日志数据文件中来查找对应偏移量的消息，那么效率一定非常低，在定位到分段后还需要顺序比对才能找到。Kafka在设计数据存储时，为了提高查找消息的效率，故而为分段后的每个日志数据文件均使用<font color="red">稀疏索引的方式建立索引</font>，这样子既节省空间又能通过索引快速定位到日志数据文件中的消息内容。偏移量索引文件和数据文件一样也同样也以该段的基准偏移量左补齐0命名，文件后缀为“.index”。下图为index文件内容的截图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191231194433919.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
从上图可以看出，index文件中的索引并不是连续的，每隔一定的字节数建立了一条索引，可以通过“index.interval.bytes”设置索引的跨度(默认的跨度为4096bytes)。
从上面dump出来的偏移量索引内容可以看出，索引条目用于将偏移量映射成为消息在日志数据文件中的实际物理位置，每个索引条目由offset和position组成，每个索引条目可以唯一确定在各个分区数据文件的一条消息。
有了偏移量索引文件，通过它，Kafka就能够根据指定的偏移量快速定位到消息的实际物理位置。具体的做法是，根据指定的偏移量，使用二分法查询定位出该偏移量对应的消息所在的分段索引文件和日志数据文件。然后通过二分查找法，继续查找出小于等于指定偏移量的最大偏移量，同时也得出了对应的position（实际物理位置），根据该物理位置在分段的日志数据文件中顺序扫描查找偏移量与指定偏移量相等的消息。下面是Kafka中分段的日志数据文件和偏移量索引文件的对应映射关系图（其中也说明了如何按照起始偏移量来定位到日志数据文件中的具体消息）。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191231193549147.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
##### 时间戳索引文件
从上面一节的分区目录中，我们还可以看到存在一些以“.timeindex”的时间戳索引文件。这种类型的索引文件是Kafka从0.10.1.1版本开始引入的的一个基于时间戳的索引文件，它们的命名方式与对应的日志数据文件和偏移量索引文件名基本一样，唯一不同的就是后缀名。从上面dump出来的该种类型的时间戳索引文件的内容来看，每一条索引条目都对应了一个8字节长度的时间戳字段和一个4字节长度的偏移量字段，其中时间戳字段记录的是该LogSegment到目前为止的最大时间戳，后面对应的偏移量即为此时插入新消息的偏移量。
另外，时间戳索引文件的时间戳类型与日志数据文件中的时间类型是一致的，索引条目中的时间戳值及偏移量与日志数据文件中对应的字段值相同（ps：Kafka也提供了通过时间戳索引来访问消息的方法）。

以上三种文件可以使用下面命令查看
```
bin/kafka-run-class.sh kafka.tools.DumpLogSegments --files /tmp/kafka-logs/test-0/00000000000000000000.timeindex --print-data-log > 00000000000000000000_timeindex.txt
```

