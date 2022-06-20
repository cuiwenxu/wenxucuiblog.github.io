@[TOC](iceberg介绍)
Apache Iceberg is an open table format for huge analytic datasets. Iceberg adds tables to Trino and Spark that use a high-performance format that works just like a SQL table.

User experience¶
Iceberg avoids unpleasant surprises. Schema evolution works and won’t inadvertently un-delete data. Users don’t need to know about partitioning to get fast queries.

Schema evolution supports add, drop, update, or rename, and has no side-effects
Hidden partitioning prevents user mistakes that cause silently incorrect results or extremely slow queries
Partition layout evolution can update the layout of a table as data volume or query patterns change
Time travel enables reproducible queries that use exactly the same table snapshot, or lets users easily examine changes
Version rollback allows users to quickly correct problems by resetting tables to a good state

Apache Iceberg是一种用于大型分析数据集的开放表格格式。Iceberg向Trino和Spark添加了使用高性能格式的表，其工作方式就像SQL表一样。

用户体验¶
冰山避免了不愉快的意外。模式演化是有效的，不会无意中恢复数据。用户不需要了解分区就可以获得快速查询。

模式演化支持添加、删除、更新或重命名，而且没有副作用
隐藏分区可以防止用户的错误，这些错误会导致无声的不正确结果或极其缓慢的查询
分区布局演变可以在数据量或查询模式改变时更新表的布局
时间旅行支持使用完全相同的表快照的可重复查询，或者允许用户轻松检查更改
版本回滚允许用户通过将表重置为良好状态来快速纠正问题

Reliability and performance¶
Iceberg was built for huge tables. Iceberg is used in production where a single table can contain tens of petabytes of data and even these huge tables can be read without a distributed SQL engine.

Scan planning is fast – a distributed SQL engine isn’t needed to read a table or find files
Advanced filtering – data files are pruned with partition and column-level stats, using table metadata
Iceberg was designed to solve correctness problems in eventually-consistent cloud object stores.

Works with any cloud store and reduces NN congestion when in HDFS, by avoiding listing and renames
Serializable isolation – table changes are atomic and readers never see partial or uncommitted changes
Multiple concurrent writers use optimistic concurrency and will retry to ensure that compatible updates succeed, even when writes conflict
可靠性和性能¶
冰山是用来装大桌子的。Iceberg用于生产环境，其中一个表可以包含数十pb的数据，甚至这些巨大的表也可以在没有分布式SQL引擎的情况下读取。

扫描计划是快速的——不需要分布式SQL引擎来读取表或查找文件
高级过滤——使用表元数据，使用分区和列级统计数据对数据文件进行修剪
Iceberg是为了解决最终一致的云对象存储中的正确性问题而设计的。

工作与任何云存储和减少网络拥塞时，在HDFS，通过避免清单和重命名
可序列化的隔离-表更改是原子的，读取器永远不会看到部分或未提交的更改
多个并发写入器使用乐观并发，并将重试以确保兼容更新成功，即使在写入冲突时也是如此

