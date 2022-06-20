@[TOC](自定义table source&sink)
# 概览
在大多数情况下，开发者不需要从头开始创建新的connector，而是想要略微修改现有的connector或hook。 在少数情况下，开发者想创建专门的连接器。

本节针对两种用例都提供帮助。 它说明了表connector的一般体系结构，从API中的声明到将在集群上执行的运行时代码。

从SQL到可运行的代码转换分为三个阶段
- metadata
此阶段DDL sql语句会被转换成CatalogTable
- planning
在planning阶段，DynamicTableSourceFactory和DynamicTableSinkFactory两个工厂类会根据CatalogTable解析出DynamicTableSource和DynamicTableSink。
默认情况下，使用Java的服务提供者接口（SPI）发现DynamicTableSourceFactory和DynamicTableSinkFactory的实例。连接器选项（例如示例中的'connector'='kafka'）必须对应于有效的工厂标识符。
- runtime
逻辑计划完成后，planner将从表连接器获取运行时实现。 运行时逻辑是在Flink的核心连接器接口（例如InputFormat或SourceFunction）中实现的。
这些接口通过另一个抽象级别分组为ScanRuntimeProvider，LookupRuntimeProvider和SinkRuntimeProvider的子类。
例如，OutputFormatProvider（提供org.apache.flink.api.common.io.OutputFormat）和SinkFunctionProvider（提供org.apache.flink.streaming.api.functions.sink.SinkFunction）都是planner可以使用的SinkRuntimeProvider的具体实例。 

下图为DDL SQL转换为运行时代码的过程
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210129135525152.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
# 完整的栗子
在这个栗子中，我们将定义一个从socket读取数据的TableSource。我们需要实现的功能点包含

- 创建可以解析和验证SQL中配置项的工厂类
- 实现table connector
- 使用flink提供的工具类，如data structure converters(数据结构转换器)和FactoryUtil

表源使用一个简单的单线程SourceFunction打开一个socket，以监听传入的字节。 原始字节可解码为行。 传入数据的格式要求为changelog标志为第一列。

需要解析的SQL为
```sql
CREATE TABLE UserScores (name STRING, score INT)
WITH (
  'connector' = 'socket',
  'hostname' = 'localhost',
  'port' = '9999',
  'byte-delimiter' = '10',
  'format' = 'changelog-csv',
  'changelog-csv.column-delimiter' = '|'
);

```
我们使用以下命令制造测试数据
```bash
> nc -lk 9999
INSERT|Alice|12
INSERT|Bob|5
DELETE|Alice|12
INSERT|Alice|18
```
## 定义TableSourceFactory
本节概述了如何将catalog转换为具体的connector实例，所有的工厂类都已添加到META-INF/services路径下。
### SocketDynamicTableFactory
SocketDynamicTableFactory将catalog转换为TableSource。 由于TableSource需要解码格式，因此为了方便起见，我们使用提供的FactoryUtil查找格式。


