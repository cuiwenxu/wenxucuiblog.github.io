@[TOC](flink cdc实践)
# 概念&原理
## flink cdc工作原理
启动MySQL CDC源时，它将获取一个全局读取锁（Flush table with read lock,带有读取锁的FLUSH TABLES），该锁将阻止其他数据库的写入。 然后，它读取当前binlog位置以及数据库和表的schema。 之后，将释放全局读取锁。 然后，它扫描数据库表并从先前记录的位置读取binlog。 Flink将定期执行检查点以记录二进制日志位置。 在进行故障转移的情况下，作业将重新启动并从受检查点的二进制日志位置恢复。 因此，它保证了仅一次的语义。
==需要注意的是：如果未授予MySQL用户RELOAD权限，则MySQL CDC源将改为使用表级锁，并使用此方法执行快照。 这会长时间阻止写入。==
