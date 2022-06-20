>以下内容基于Flink 1.12.0

Flink支持基于processing-time动态join hive表，基于processing-time的动态join 始终join hive表的最新版本，Flink支持分区表和Hive非分区表的temporal join，对于分区表，Flink支持自动跟踪Hive表的最新分区。
>注意：Flink 暂时不支持基于event-time的temporal join

Temporal Join The Latest Partition

对于持续变化的分区表，我们可以把它视作无界流，那么分区就对应作动态表的版本。在processing-time语义下，Flink join支持自动跟踪Hive表的最新分区。最新版本可通过‘streaming-source.partition-order’ 选项进行配置。
以下演示显示了经典的业务场景，维度表来自Hive，并且每天通过batch pipeline作业或Flink作业进行更新，kafka流为在线业务数据或日志，并且需要与维度结合表来丰富流。



客户端读hive表有jar包依赖冲突，暂时hang住，在命令行调试。
尝试将cdc的数据sink到hdfs，无分区的hive表可自动加载hdfs上的数据



