###### 问题：
>hi、
flink程序日志上报到kafka可以使用现成的KafkaLog4jAppender，但是这个现成的类没办法在上报的message中携带app_id，没有app_id就没办法识别是哪个app的日志，请问怎样才能在上报的日志中添加app_id

###### 答案：
>我们的获取逻辑是通过自定义KafkaLog4jAppender，自定义appender通过解析当前系统路径(因为flink每个taskmanager会自己定义一个带有applicationId的路径，然后里面会放各种jar包，包括我自定义的appender)，获取之后通过MDC.put(),给日志加一列appId，在appder里面把日志上报到外部的日志系统

###### 实现
log4j.properties文件配置
```java
log4j.rootLogger=INFO,kafka,console
# Uncomment this if you want to _only_ change Flink's logging
#log4j.logger.org.apache.flink=INFO
# The following lines keep the log level of common libraries/connectors on
# log level INFO. The root logger does not override this. You have to manually
# change the log levels here.
log4j.logger.org.apache.kafka=WARN
log4j.logger.org.apache.hadoop=INFO
log4j.appender.console=org.apache.log4j.ConsoleAppender
log4j.appender.console.target=System.INFO
log4j.appender.console.layout=org.apache.log4j.PatternLayout
#log4j.appender.console.layout.ConversionPattern=%d{yy/MM/dd HH:mm:ss} %p %c{2}: %m%n
log4j.appender.console.layout.ConversionPattern={"debug_level":"%p","debug_timestamp":"%d{ISO8601}","debug_thread":"%t","debug_file":"%F", "debug_line":"%L","debug_message":"%m"}%n
# log send to kafka
#log4j.appender.kafka=org.apache.kafka.log4jappender.KafkaLog4jAppender
#使用自定义appender
log4j.appender.kafka=com.aibee.di.util.KafkaAppender
log4j.appender.kafka.brokerList=xxx
log4j.appender.kafka.topic=flink-log-test
log4j.appender.kafka.compressionType=none
log4j.appender.kafka.requiredNumAcks=0
log4j.appender.kafka.syncSend=false
log4j.appender.kafka.layout=org.apache.log4j.PatternLayout
#log4j.appender.kafka.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} %-5p %c{1}:%L - %m%n
log4j.appender.kafka.layout.ConversionPattern={"debug_level":"%p","debug_timestamp":"%d{ISO8601}","debug_thread":"%t","debug_file":"%F", "debug_line":"%L","debug_message":"%m","log_path":"%X{log_path}","app_id":"%X{app_id}"}%n
```
将如下自定义的log4j appender 打包，放在flink 的lib目录下面，任务启动就可以在kafka查看到日志信息
```java
package com.xxx.xx.util;

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;
import org.apache.kafka.common.config.ConfigException;
import org.apache.kafka.common.serialization.ByteArraySerializer;
import org.apache.log4j.AppenderSkeleton;
import org.apache.log4j.helpers.LogLog;
import org.apache.log4j.spi.LoggingEvent;

import java.nio.charset.StandardCharsets;
import java.util.Date;
import java.util.Properties;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Future;

import static org.apache.kafka.clients.producer.ProducerConfig.COMPRESSION_TYPE_CONFIG;
import static org.apache.kafka.clients.producer.ProducerConfig.ACKS_CONFIG;
import static org.apache.kafka.clients.producer.ProducerConfig.RETRIES_CONFIG;
import static org.apache.kafka.clients.producer.ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG;
import static org.apache.kafka.clients.producer.ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG;
import static org.apache.kafka.clients.CommonClientConfigs.BOOTSTRAP_SERVERS_CONFIG;
import static org.apache.kafka.clients.CommonClientConfigs.SECURITY_PROTOCOL_CONFIG;
import static org.apache.kafka.common.config.SslConfigs.SSL_TRUSTSTORE_LOCATION_CONFIG;
import static org.apache.kafka.common.config.SslConfigs.SSL_TRUSTSTORE_PASSWORD_CONFIG;
import static org.apache.kafka.common.config.SslConfigs.SSL_KEYSTORE_TYPE_CONFIG;
import static org.apache.kafka.common.config.SslConfigs.SSL_KEYSTORE_LOCATION_CONFIG;
import static org.apache.kafka.common.config.SslConfigs.SSL_KEYSTORE_PASSWORD_CONFIG;
import static org.apache.kafka.common.config.SaslConfigs.SASL_KERBEROS_SERVICE_NAME;

/**
 * A log4j appender that produces log messages to Kafka
 */
public class KafkaAppender extends AppenderSkeleton {

    private String brokerList;
    private String topic;
    private String compressionType;
    private String securityProtocol;
    private String sslTruststoreLocation;
    private String sslTruststorePassword;
    private String sslKeystoreType;
    private String sslKeystoreLocation;
    private String sslKeystorePassword;
    private String saslKerberosServiceName;
    private String clientJaasConfPath;
    private String kerb5ConfPath;
    private String logPath;

    private int retries;
    private int requiredNumAcks = Integer.MAX_VALUE;
    private boolean syncSend;
    private Producer<byte[], byte[]> producer;

    public Producer<byte[], byte[]> getProducer() {
        return producer;
    }

    public String getBrokerList() {
        return brokerList;
    }

    public void setBrokerList(String brokerList) {
        this.brokerList = brokerList;
    }

    public int getRequiredNumAcks() {
        return requiredNumAcks;
    }

    public void setRequiredNumAcks(int requiredNumAcks) {
        this.requiredNumAcks = requiredNumAcks;
    }

    public int getRetries() {
        return retries;
    }

    public void setRetries(int retries) {
        this.retries = retries;
    }

    public String getCompressionType() {
        return compressionType;
    }

    public void setCompressionType(String compressionType) {
        this.compressionType = compressionType;
    }

    public String getTopic() {
        return topic;
    }

    public void setTopic(String topic) {
        this.topic = topic;
    }

    public boolean getSyncSend() {
        return syncSend;
    }

    public void setSyncSend(boolean syncSend) {
        this.syncSend = syncSend;
    }

    public String getSslTruststorePassword() {
        return sslTruststorePassword;
    }

    public String getSslTruststoreLocation() {
        return sslTruststoreLocation;
    }

    public String getSecurityProtocol() {
        return securityProtocol;
    }

    public void setSecurityProtocol(String securityProtocol) {
        this.securityProtocol = securityProtocol;
    }

    public void setSslTruststoreLocation(String sslTruststoreLocation) {
        this.sslTruststoreLocation = sslTruststoreLocation;
    }

    public void setSslTruststorePassword(String sslTruststorePassword) {
        this.sslTruststorePassword = sslTruststorePassword;
    }

    public void setSslKeystorePassword(String sslKeystorePassword) {
        this.sslKeystorePassword = sslKeystorePassword;
    }

    public void setSslKeystoreType(String sslKeystoreType) {
        this.sslKeystoreType = sslKeystoreType;
    }

    public void setSslKeystoreLocation(String sslKeystoreLocation) {
        this.sslKeystoreLocation = sslKeystoreLocation;
    }

    public void setSaslKerberosServiceName(String saslKerberosServiceName) {
        this.saslKerberosServiceName = saslKerberosServiceName;
    }

    public void setClientJaasConfPath(String clientJaasConfPath) {
        this.clientJaasConfPath = clientJaasConfPath;
    }

    public void setKerb5ConfPath(String kerb5ConfPath) {
        this.kerb5ConfPath = kerb5ConfPath;
    }

    public String getSslKeystoreLocation() {
        return sslKeystoreLocation;
    }

    public String getSslKeystoreType() {
        return sslKeystoreType;
    }

    public String getSslKeystorePassword() {
        return sslKeystorePassword;
    }

    public String getSaslKerberosServiceName() {
        return saslKerberosServiceName;
    }

    public String getClientJaasConfPath() {
        return clientJaasConfPath;
    }

    public String getKerb5ConfPath() {
        return kerb5ConfPath;
    }

    @Override
    public void activateOptions() {
        this.logPath = System.getProperty("user.dir");
        // check for config parameter validity
        Properties props = new Properties();
        if (brokerList != null)
            props.put(BOOTSTRAP_SERVERS_CONFIG, brokerList);
        if (props.isEmpty())
            throw new ConfigException("The bootstrap servers property should be specified");
        if (topic == null)
            throw new ConfigException("Topic must be specified by the Kafka log4j appender");
        if (compressionType != null)
            props.put(COMPRESSION_TYPE_CONFIG, compressionType);
        if (requiredNumAcks != Integer.MAX_VALUE)
            props.put(ACKS_CONFIG, Integer.toString(requiredNumAcks));
        if (retries > 0)
            props.put(RETRIES_CONFIG, retries);
        if (securityProtocol != null) {
            props.put(SECURITY_PROTOCOL_CONFIG, securityProtocol);
        }
        if (securityProtocol != null && securityProtocol.contains("SSL") && sslTruststoreLocation != null &&
                sslTruststorePassword != null) {
            props.put(SSL_TRUSTSTORE_LOCATION_CONFIG, sslTruststoreLocation);
            props.put(SSL_TRUSTSTORE_PASSWORD_CONFIG, sslTruststorePassword);

            if (sslKeystoreType != null && sslKeystoreLocation != null &&
                    sslKeystorePassword != null) {
                props.put(SSL_KEYSTORE_TYPE_CONFIG, sslKeystoreType);
                props.put(SSL_KEYSTORE_LOCATION_CONFIG, sslKeystoreLocation);
                props.put(SSL_KEYSTORE_PASSWORD_CONFIG, sslKeystorePassword);
            }
        }
        if (securityProtocol != null && securityProtocol.contains("SASL") && saslKerberosServiceName != null && clientJaasConfPath != null) {
            props.put(SASL_KERBEROS_SERVICE_NAME, saslKerberosServiceName);
            System.setProperty("java.security.auth.login.config", clientJaasConfPath);
            if (kerb5ConfPath != null) {
                System.setProperty("java.security.krb5.conf", kerb5ConfPath);
            }
        }

        props.put(KEY_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class.getName());
        props.put(VALUE_SERIALIZER_CLASS_CONFIG, ByteArraySerializer.class.getName());
        this.producer = getKafkaProducer(props);
        LogLog.debug("Kafka producer connected to " + brokerList);
        LogLog.debug("Logging for topic: " + topic);
    }

    protected Producer<byte[], byte[]> getKafkaProducer(Properties props) {
        return new KafkaProducer<>(props);
    }

    @Override
    protected void append(LoggingEvent event) {
        event.setProperty("log_path", logPath);
        String message = subAppend(event);
        message = message + logPath;
        LogLog.debug("[" + new Date(event.getTimeStamp()) + "]" + message);
        Future<RecordMetadata> response = producer.send(
                new ProducerRecord<byte[], byte[]>(topic, message.getBytes(StandardCharsets.UTF_8)));
        if (syncSend) {
            try {
                response.get();
            } catch (InterruptedException | ExecutionException ex) {
                throw new RuntimeException(ex);
            }
        }
    }

    private String subAppend(LoggingEvent event) {
        return (this.layout == null) ? event.getRenderedMessage() : this.layout.format(event);
    }

    @Override
    public void close() {
        if (!this.closed) {
            this.closed = true;
            producer.close();
        }
    }

    @Override
    public boolean requiresLayout() {
        return true;
    }
}
```

附上问题链接：
http://apache-flink.147419.n8.nabble.com/flink1-11-td5389.html
