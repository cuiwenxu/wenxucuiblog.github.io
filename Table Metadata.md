# Table Metadata
表元数据存储为JSON。每个表元数据更改都会创建一个由原子操作提交的新表元数据文件。此操作用于确保表元数据的新版本替换它所基于的版本。这将生成表版本的线性历史记录，并确保并发写操作不会丢失。

用于提交元数据的原子操作取决于如何跟踪表，该规范没有对其进行标准化。
## Table Metadata Fields
表元数据由以下字段组成:
v1     | v2 | 字段 | 描述
-------- | ----- | -----| -----
 required | required | format-version | 格式的整数版本号。目前，它总是1。如果表的版本高于支持的版本，则会抛出异常
 optional | required | table-uuid | 标识表的UUID，在创建表时生成。如果刷新元数据后，表的UUID与预期的UUID不匹配，则会抛出异常。
 required | required | location | 表的基本位置。writers使用它来确定在何处存储datafile（数据文件）、manifest（清单文件）和metadata（表元数据文件）
 无 | required | last-sequence-number | 表中分配的最大序号，单调递增的long型变量，用于跟踪表中快照的顺序。
 required | required | last-updated-ms | 从上一次更新表的unix时间戳(以毫秒为单位)。每个表元数据文件应该在写入之前更新这个字段
 required | required | last-column-id | 一个整数;为表分配的最大列ID
 required | 无 | schema | 表的当前schema。(已弃用:使用schema和current-schema-id代替)
 optional | required | schemas | schema列表，存储格式为带有schema id的对象
 optional | required | current-schema-id | 表的当前schema ID
 required | 无 | partition-spec | 表的当前分区字段。请注意，writer使用它对数据进行分区，但在读取时不使用，因为读取使用存储在清单文件中的字段。(废弃的:使用partition-spec和default-spec-id代替)
 optional | required | partition-specs | 分区字段列表
 optional | required | default-spec-id | writer默认使用的当前分区字段ID
 optional | required | last-partition-id | An integer; the highest assigned partition field ID across all partition specs for the table. This is used to ensure partition fields are always assigned an unused ID when evolving specs.
 optional | optional | properties | 表属性的字符串到字符串的映射。这些属性用于控制读写，而不是用于任意元数据。例如,commit.retry.Num-retries用于控制提交重试次数。
 optional | optional | current-snapshot-id | long ID of the current table snapshot
 optional | optional | snapshots | 有效快照的列表。有效快照是指其数据文件都真实存在于文件系统中，不能有被删除的
 optional | optional | snapshot-log | A list (optional) of timestamp and snapshot ID pairs that encodes changes to the current snapshot for the table. Each time the current-snapshot-id is changed, a new entry should be added with the last-updated-ms and the new current-snapshot-id. When snapshots are expired from the list of valid snapshots, all entries before a snapshot that has expired should be removed.
  optional | optional | metadata-log | A list (optional) of timestamp and metadata file location pairs that encodes changes to the previous metadata files for the table. Each time a new metadata file is created, a new entry of the previous metadata file location should be added to the list. Tables can be configured to remove oldest metadata log entries and keep a fixed-size log of the most recent entries after a commit.
  optional | required | sort-orders | A list of sort orders, stored as full sort order objects.
   optional | required | default-sort-order-id | Default sort order id of the table. Note that this could be used by writers, but is not used when reading because reads use the specs stored in manifest files.
 



