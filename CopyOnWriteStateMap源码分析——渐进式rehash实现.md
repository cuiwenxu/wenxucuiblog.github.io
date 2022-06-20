# 需求场景
渐进式 rehash 策略表示 CopyOnWriteStateMap 中当前有一个 hash 表对外服务，但是当前 hash 表中元素太多需要扩容了，需要将数据迁移到一个容量更大的 hash 表中。

Java 的 HashMap 在扩容时会一下子将旧 hash 表中所有数据都移动到大 hash 表中，这样的策略存在的问题是如果 HashMap 当前存储了 1 G 的数据，那么瞬间需要将 1 G 的数据迁移完，可能会比较耗时。此期间是无法对外提供服务的，这很影响可用性。

> 总结：表需要扩容且扩容期间还要对外提供服务

# 实现方案

而 CopyOnWriteStateMap 在扩容时，不会一下子将数据全部迁移完，而是在每次操作 CopyOnWriteStateMap 时，慢慢去迁移数据到大的 hash 表中。

例如：可以在每次 get、put 操作时，迁移 4 条数据到大 hash 表中，这样经过一段时间的 get 和 put 操作，所有的数据就能迁移完成。所以渐进式 rehash 策略，会分很多次将所有的数据迁移到新的 hash 表中。
