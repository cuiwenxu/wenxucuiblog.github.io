@[TOC](hive catalog在flink中的应用)
# 概览
多年来，Hive Metastore已发展成为Hadoop生态系统的元数据中心。对于同时部署了Hive和Flink的用户，HiveCatalog使他们能够使用Hive Metastore管理Flink的元数据。
对于仅部署Flink的用户，HiveCatalog是Flink开箱即用的唯一持久目录。 如果没有持久性目录，使用Flink SQL CREATE DDL的用户必须在每个会话中重复创建像Kafka表这样的元对象，这会浪费大量时间。 HiveCatalog通过授权用户仅创建一次表和其他元对象，并在以后的跨会话中方便地进行引用和管理来填补这一空白。
# 怎样使用HiveCatalog
一旦正确配置，HiveCatalog应该即开即用。用户可以使用DDL创建Flink元对象，然后可以立即查看它们。
HiveCatalog可用于处理两种表：Hive兼容表和通用表。就存储层中的元数据和数据而言，兼容Hive的表是以兼容Hive的方式存储的表。因此，可以从Hive端查询通过Flink创建的Hive兼容表。

另一方面，通用表仅Flink程序可用。使用HiveCatalog创建通用表时，我们只是使用HMS来保留元数据。虽然这些表格对Hive可见，但Hive不能够理解元数据。因此，在Hive中使用此类表会导致未定义的行为。

Flink使用属性`is_generic`来表示一个表是Hive兼容表还是通用表。使用HiveCatalog创建表格时，默认情况下将其视为通用表格。如果要创建与Hive兼容的表，请确保在表属性中将`is_generic`设置为false。

如上所述，不应在Hive中使用通用表。在Hive CLI中，您可以为表调用`DESCRIBE FORMATTED`并通过检查`is_generic`属性来确定该表是否通用。通用表将具有`is_generic = true`。
# example
我们用一个例子从头到尾讲一遍
>准备条件：有个正在运行的Hive Metastore,搭建教程见
[hive metastore配置](https://blog.csdn.net/u011624157/article/details/112496673).
## 步骤一：设置Hive Metastore
在flink客户端conf目录下添加hive-site.xml文件，并在其中做如下配置
```xml
<configuration>
   <property>
      <name>javax.jdo.option.ConnectionURL</name>
      <value>jdbc:mysql://localhost/metastore?createDatabaseIfNotExist=true</value>
      <description>metadata is stored in a MySQL server</description>
   </property>

   <property>
      <name>javax.jdo.option.ConnectionDriverName</name>
      <value>com.mysql.jdbc.Driver</value>
      <description>MySQL JDBC driver class</description>
   </property>

   <property>
      <name>javax.jdo.option.ConnectionUserName</name>
      <value>...</value>
      <description>user name for connecting to mysql server</description>
   </property>

   <property>
      <name>javax.jdo.option.ConnectionPassword</name>
      <value>...</value>
      <description>password for connecting to mysql server</description>
   </property>

   <property>
       <name>hive.metastore.uris</name>
       <value>thrift://localhost:9083</value>
       <description>IP address (or fully-qualified domain name) and port of the metastore host</description>
   </property>

   <property>
       <name>hive.metastore.schema.verification</name>
       <value>true</value>
   </property>

</configuration>
```
## 步骤二：添加依赖和配置sql-cli-defaults.yaml
在flink客户端lib目录下添加jar包，[jar包列表](https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/table/connectors/hive/#%E4%BE%9D%E8%B5%96%E9%A1%B9)，按下面所示配置sql-cli-defaults.yaml
```yaml
execution:
    planner: blink
    type: streaming
    ...
    current-catalog: myhive  # 如果hive中没有会自动创建
    current-database: mydatabase  # hive中已存在的库
    
catalogs:
   - name: myhive
     type: hive
     hive-conf-dir: /opt/hive-conf  # contains hive-site.xml
```
上面两个步骤完成后配置便已完成，此时在flink sql client中创建的表在hive中就可以看到了
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210111215622923.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210111215708690.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)


