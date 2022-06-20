@[TOC](kafka消息可靠性还有准确性保障)
# 可靠性&准确性定义
首先弄明白定义
- `不丢数据`即为可靠
- `不丢&不重(恰好一次)`即为准确性

# kafka怎么保证不丢数据
kafka是多副本的，producer发送给leader副本的消息会被同步到follower副本上，并且kafka会将leader副本和follower尽量分布在不同的机器上，只有当所有的副本都丢失，数据才有可能丢失。
# kafka怎么保证恰好一次
## 生产者->broker(幂等)
通过幂等可实现不丢不重，在producer客户端设置`enable.idempotence=true`即可开启幂等。具体实现为给每一条消息分配一个sequence number，同时broker端在内存会为每一个<producer_id,分区>都维护一个sn_old,只有当producer发送的消息的sn_new=sn_old+1时，broker才会接受此消息。
## 生产者->broker->消费者(事务)
可以通过事务，将消费->转换->生产放到一个事务中，实现原子性操作。具体实现是
生产者定义transaction_id，用于识别不同的事务，代码如下
```java
Properties properties=new Properties();
properties.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG,"自定义transaction_id")
KafkaProducer producer=new KafkaProducer();
producer.initTransaction();
producer.beginTransaction();
try {
	ProducerRecord record1=new ProducerRecord();
	producer.send();
	producer.commitTransaction();
} catch (){
	producer.abortTransaction();
}
```




