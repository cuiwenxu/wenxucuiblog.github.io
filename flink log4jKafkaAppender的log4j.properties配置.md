```java
# This affects logging for both user code and Flink
log4j.rootLogger=INFO,kafka,console


# Uncomment this if you want to _only_ change Flink's logging
#log4j.logger.org.apache.flink=INFO

# The following lines keep the log level of common libraries/connectors on
# log level INFO. The root logger does not override this. You have to manually
# change the log levels here.
log4j.logger.org.apache.kafka=WARN,stdout,Error
log4j.logger.org.apache.hadoop=INFO

log4j.appender.console=org.apache.log4j.ConsoleAppender
log4j.appender.console.target=System.INFO
log4j.appender.console.layout=org.apache.hadoop.log.Log4Json


# log send to kafka
log4j.appender.kafka=org.apache.kafka.log4jappender.KafkaLog4jAppender
log4j.appender.kafka.brokerList=
log4j.appender.kafka.topic=log4j_log_test
log4j.appender.kafka.compressionType=none
log4j.appender.kafka.requiredNumAcks=0
log4j.appender.kafka.syncSend=false
log4j.appender.kafka.layout=org.apache.hadoop.log.Log4Json
log4j.appender.kafka.level=INFO

```


