# flink消费kafka细节

Apache kafka connector提供对Kafka服务的事件流的访问。Flink提供了特殊的Kafka连接器，用于从Kafka主题读写数据。 Flink Kafka Consumer与Flink的检查点机制集成在一起，以提供一次精确的处理语义。 为此，Flink不仅仅依赖于Kafka的消费者群体偏移量跟踪，还内部跟踪和检查这些偏移量。

请为您的用例和环境选择一个包（Maven项目ID）和类名。 对于大多数用户来说，FlinkKafkaConsumer08（flink-connector-kafka的一部分）是合适的。

| Maven Dependency                | Supported since | Consumer and Producer Class name            | Kafka version | Notes                                                        |
| :------------------------------ | :-------------- | :------------------------------------------ | :------------ | :----------------------------------------------------------- |
| flink-connector-kafka-0.8_2.11  | 1.0.0           | FlinkKafkaConsumer08 FlinkKafkaProducer08   | 0.8.x         | Uses the [SimpleConsumer](https://cwiki.apache.org/confluence/display/KAFKA/0.8.0+SimpleConsumer+Example) API of Kafka internally. Offsets are committed to ZK by Flink. |
| flink-connector-kafka-0.9_2.11  | 1.0.0           | FlinkKafkaConsumer09 FlinkKafkaProducer09   | 0.9.x         | Uses the new [Consumer API](http://kafka.apache.org/documentation.html#newconsumerapi) Kafka. |
| flink-connector-kafka-0.10_2.11 | 1.2.0           | FlinkKafkaConsumer010 FlinkKafkaProducer010 | 0.10.x        | This connector supports [Kafka messages with timestamps](https://cwiki.apache.org/confluence/display/KAFKA/KIP-32+-+Add+timestamps+to+Kafka+message) both for producing and consuming. |
| flink-connector-kafka-0.11_2.11 | 1.4.0           | FlinkKafkaConsumer011 FlinkKafkaProducer011 | 0.11.x        | Since 0.11.x Kafka does not support scala 2.10. This connector supports [Kafka transactional messaging](https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging) to provide exactly once semantic for the producer. |
| flink-connector-kafka_2.11      | 1.7.0           | FlinkKafkaConsumer FlinkKafkaProducer       | >= 1.0.0      | This universal Kafka connector attempts to track the latest version of the Kafka client. The version of the client it uses may change between Flink releases. Starting with Flink 1.9 release, it uses the Kafka 2.2.0 client. Modern Kafka clients are backwards compatible with broker versions 0.10.0 or later. However for Kafka 0.11.x and 0.10.x versions, we recommend using dedicated flink-connector-kafka-0.11_2.11 and flink-connector-kafka-0.10_2.11 respectively. |

在创建kafka consumer时，需要指定一些参数

```java
Properties properties = new Properties();
// kafka broker地址，用逗号隔开
properties.setProperty("bootstrap.servers", "localhost:9092");
// zookeeper机器地址，仅在Kafka 0.8用到
properties.setProperty("zookeeper.connect", "localhost:2181");
// kafka消费者的group.id
properties.setProperty("group.id", "test");
DataStream<String> stream = env
	.addSource(new FlinkKafkaConsumer08<>("topic", new SimpleStringSchema(), properties));
```



### flink消费kafka时的容错

启用flink检查点之后，flink会定期checkpoint offset，万一作业失败，Flink将把流式程序恢复到最新检查点的状态，并从存储在检查点的偏移量开始重新使用Kafka的记录。

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.enableCheckpointing(5000); // checkpoint every 5000 msecs
```

### flink worker和kafka partition对应关系

partition会分配给flink并行的task，当task比partition数量多时，会有task进程闲置
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191210141710109.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)

当kafka的partition比flink task多时，一个task会分配到多个partition

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019121014173421.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
### flink如何保证kafka的恰好一次处理

flink kafka consumer和flink的检查点机制紧密集成，flink每次从kafka队列读到新数据都会更新offset到state中，flink kafka consumer是一个stateful operator，其state存的就是offset。

### 从Kafka主题阅读时，Flink如何处理背压？

如果下游无法以相同的速度处理所有传入数据，则像Flink这样的流媒体系统必须能够减慢flink kafka consumer消费的速度。这就是所谓的反压处理。 Flink的Kafka consumer自带处理backpressure的机制：一旦下游无法跟上Kafka消费的速度，Flink就会放慢来自Kafka的消息的消费，从而减少来自代理的请求。由于代理将所有消息都保留在磁盘上，因此它们也可以提供过去的消息。一旦操作员再次加速，Flink将全速消耗累积的消息。这种行为使Kafka非常适合作为流源和Flink之间的缓冲区，因为它为负载高峰时的事件提供了持久的缓冲区。

### kafka生产者API的使用


