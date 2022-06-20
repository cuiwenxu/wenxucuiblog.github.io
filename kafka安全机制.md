==SSL或SASL概念普及，SASL全称Simple Authentication and Security Layer，是一种用来扩充C/S模式验证能力的机制。SSL(Secure Sockets Layer 安全套接字协议),及其继任者传输层安全（Transport Layer Security，TLS）是为网络通信提供安全及数据完整性的一种安全协议。==
在0.9.0.0版中，Kafka社区添加了SSL或SASL对来自客户端（生产者和消费者），其他代理和工具的代理的连接进行身份验证。Kafka支持以下SASL机制：
  - SASL/GSSAPI (Kerberos) - 从version 0.9.0.0开始
  - SASL/PLAIN - 从version 0.10.0.0开始
  - SASL/SCRAM-SHA-256 and SASL/SCRAM-SHA-512 - 从version 0.10.2.0开始
  - SASL/OAUTHBEARER - 从version 2.0开始
##### kafka client配置
仅新的Kafka Producer和Consumer使用SSL，不支持旧的API。 对于生产者和消费者，SSL的配置将相同。如果代理中不需要客户端身份验证，则以下是最小配置示例：
```java
 security.protocol=SSL
 ssl.truststore.location=/var/private/ssl/client.truststore.jks
 ssl.truststore.password=test1234
```
注意：ssl.truststore.password在技术上是可选的，但强烈建议使用。 如果未设置密码，则仍然可以访问信任库，但是将禁用完整性检查。 如果需要客户端身份验证，则kafka集群必须创建密钥库，client还必须配置以下内容：
```java
ssl.keystore.location=/var/private/ssl/client.keystore.jks
ssl.keystore.password=test1234
ssl.key.password=test1234
```

