```java
status = error
dest = err
name = PropertiesConfig

appender.console.name = ConsoleAppender
appender.console.type = CONSOLE
appender.console.layout.type = PatternLayout
appender.console.layout.pattern = %d{yyyy-MM-dd HH:mm:ss,SSS} %-5p %-60c %x - %m%n


appender.kafka.name = kafka
appender.kafka.type = kafka
appender.kafka.layout.type = PatternLayout
appender.kafka.layout.pattern = %d{yyyy-MM-dd HH:mm:ss,SSS} %-5p %-60c %x - %m%n
appender.kafka.topic=log4j_log_test
appender.kafka.property.name=bootstrap.servers
appender.kafka.property.type=Property
appender.kafka.property.value=172.16.244.42:9092
#appender.kafka.brokerList=172.16.244.42:9092

rootLogger.level = INFO
rootLogger.appenderRef.console.ref = ConsoleAppender
rootLogger.appenderRef.kafka.ref = kafka
```
