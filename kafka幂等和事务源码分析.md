@[TOC](kafka幂等和事务源码分析)
自0.11.0.0之后，kafka实现了幂等性和事务，保证exactly once semantic。接下来分别介绍幂等性和事务的概念的分析源码。
# 概念及配置
## 幂等
所谓的幕等，简单地说就是对接口的多次调用所产生的结果和调用一次是一致 。生产者
在进行重试的时候有可能会重复写入消息，而使用 Kafka 幕等性功能之后就可以避免这种情
况。
开启幕等性功能的方式很简单，只需要显式地将生产者客户端参数 enable.idempotence
设置为 true 即可（这个参数的默认值为 false ），参考如下：
` properties.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG,true); `

或者
` properties.put('enable.idempotence',enable dempotence true);`

不过如果要确保幂等功能正常，还需要确保生产者客户端的三个参数配置正确
```java
//client单个connection阻塞前可持有的最大的未确认请求数量，默认为5
MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION<=5
//client发送失败后重试次数,默认为Integer.MAX_VALUE
RETRIES_CONFIG>=0
//消息可靠性确认数量，默认为1
ACKS_CONFIG=all(相当于配置acks=-1)
```
实际上使用幕等功能的时候，用户完全可以不用配置（也不建议配置）这三个参数。因为设置了ENABLE_IDEMPOTENCE_CONFIG=true之后这三个参数会自动调整。用户配置不对反而会报ConfigException。
## 事务

# 实现原理和代码分析
## 幂等
为了实现生产者的幕等性 Kafka 为此引入producer id 以下简称 PID 和序号sequence number这两个概念，每个新的生产者实例在初始化的时候都会被分配一个PID这个PID对用户而言是完全透明的,对于每个 PID,消息发送到的每一个分区都有对应的序列号，这些序列号从0开始单调递增。生产者每发送一条消息就会将<PID,分区>对应的序列号的值加1。broker 端会在内存中为每一对<PID,分区>维护一个序列号。对于收到的每一条消息，只有当它的序列号的值SN_new比 broker 端中维护的对应的序列号的值SN_old大1，即SN_new = SN_old + 1 时，broker 才会接收它。如果 SN_new < SN_old + 1,那么说明消息被重复写入,broker 可以直接将其丢弃。如果 SN_new > SN_old + 1，那么说明中间有数据尚未写入， 暗示可能有消息丢失，对应的生产者会抛出 OutOfOrderSequenceException,这个异常是个严重的异常，后续的诸如send、beginTransaction、commitTransaction方法的调用都会抛出 IllegalStateException 的异常。引入序列号来实现幂等也只是针对每一对<PID,分区>而言的，也就是说，Kafka幂等只能保证单个生产者会话中单分区的幂等。
```java
ProducerRecord<String , String> record ＝new ProducerRecord<>(topic,"key","msg") ; 
producer.send(record); 
producer.send(record); 
```
注意，上面示例中发送了两条相同的消息，不过这仅仅是指消息内容相同，但对 Kafka而言是两条不同 的消息，因为会为这两条消息分配不同的序列号 Kafka 并不会保证消息幂等。
