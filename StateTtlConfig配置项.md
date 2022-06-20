```java
StateTtlConfig ttlConfig = StateTtlConfig
                            .newBuilder(org.apache.flink.api.common.time.Time.seconds(1))
                            .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
                            .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
                            .cleanupFullSnapshot()
                            .build();
 ```
                            
- TTL：
>通过`newBuilder(org.apache.flink.api.common.time.Time.seconds(1))`设置

表示状态的过期时间，是一个 org.apache.flink.api.common.time.Time 对象。一旦设置了 TTL，那么如果上次访问的时间戳 + TTL 超过了当前时间，则表明状态过期了（这是一个简化的说法，严谨的定义请参考 org.apache.flink.runtime.state.ttl.TtlUtils 类中关于 expired 的实现） 。
- UpdateType：
>通过`setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)`设置

表示状态时间戳的更新的时机，是一个 Enum 对象。如果设置为 Disabled，则表明不更新时间戳；如果设置为 OnCreateAndWrite，则表明当状态创建或每次写入时都会更新时间戳；如果设置为 OnReadAndWrite，则除了在状态创建和写入时更新时间戳外，读取也会更新状态的时间戳。
- StateVisibility：
>通过`setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)`设置

表示对已过期但还未被清理掉的状态如何处理，也是 Enum 对象。如果设置为 ReturnExpiredIfNotCleanedUp，那么即使这个状态的时间戳表明它已经过期了，但是只要还未被真正清理掉，就会被返回给调用方；如果设置为 NeverReturnExpired，那么一旦这个状态过期了，那么永远不会被返回给调用方，只会返回空状态，避免了过期状态带来的干扰。
TimeCharacteristic 以及 TtlTimeCharacteristic：表示 State TTL 功能所适用的时间模式，仍然是 Enum 对象。前者已经被标记为 Deprecated（废弃），推荐新代码采用新的 TtlTimeCharacteristic 参数。截止到 Flink 1.8，==只支持 ProcessingTime 一种时间模式==，对 EventTime 模式的 State TTL 支持还在开发中。
- CleanupStrategies：
>通过`cleanupIncrementally() 增量清理 .cleanupFullSnapshot()全量清理`来设置

表示过期对象的清理策略，目前来说有三种 Enum 值。当设置为 FULL_STATE_SCAN_SNAPSHOT 时，对应的是 EmptyCleanupStrategy 类，表示对过期状态不做主动清理，当执行完整快照（Snapshot / Checkpoint）时，会生成一个较小的状态文件，但本地状态并不会减小。唯有当作业重启并从上一个快照点恢复后，本地状态才会实际减小，因此可能仍然不能解决内存压力的问题。为了应对这个问题，Flink 还提供了增量清理的枚举值，分别是针对 Heap StateBackend 的 INCREMENTAL_CLEANUP（对应 IncrementalCleanupStrategy 类），以及对 RocksDB StateBackend 有效的 ROCKSDB_COMPACTION_FILTER（对应 RocksdbCompactFilterCleanupStrategy 类）.  对于增量清理功能，Flink 可以被配置为每读取若干条记录就执行一次清理操作，而且可以指定每次要清理多少条失效记录；对于 RocksDB 的状态清理，则是通过 JNI 来调用 C++ 语言编写的 FlinkCompactionFilter 来实现，底层是通过 RocksDB 提供的后台 Compaction 操作来实现对失效状态过滤的。              
