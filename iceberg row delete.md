# Iceberg表规范
这是Iceberg表格式的规范，用于管理分布式文件系统中变化缓慢的大型文件集合或作为表的键值存储。

## 版本1:分析数据表
版本1是当前的版本。它定义了如何使用不可变的文件格式管理大型分析表，如Parquet, Avro和ORC。

## 版本2:行级删除
Iceberg社区目前正在开发支持行级删除编码的Iceberg格式的第2版。v2规范是不完整的，在完成和采用之前可能会发生变化。本文档包含了暂定的v2格式要求，但目前没有未完成的v2规范的兼容性保证。

版本2的主要目标是提供一种对行级删除的方法。此更新可用于删除或替换不可变数据文件中的个别行，而无需重写该文件。

Delete Formats¶
This section details how to encode row-level deletes in Iceberg delete files. Row-level deletes are not supported in v1.

Row-level delete files are valid Iceberg data files: files must use valid Iceberg formats, schemas, and column projection. It is recommended that delete files are written using the table’s default file format.

Row-level delete files are tracked by manifests, like data files. A separate set of manifests is used for delete files, but the manifest schemas are identical.

Both position and equality deletes allow encoding deleted row values with a delete. This can be used to reconstruct a stream of changes to a table.

# 删除格式
本节详细介绍如何在Iceberg删除文件中对行级删除进行编码。`v1中不支持行级删除`。

行级删除文件是有效的Iceberg数据文件:文件必须使用有效的Iceberg格式、模式和列projection。建议使用表的默认文件格式写入删除文件。

行级删除文件由manifest跟踪。删除文件使用一组各自分立的manifest，但manifest的模式是相同的。

`位置删除`和`条件判等删除`都允许用delete对已删除的行值进行编码。这可用于重建表的更改流。
Position Delete Files¶
Position-based delete files identify deleted rows by file and position in one or more data files, and may optionally contain the deleted row.

A data row is deleted if there is an entry in a position delete file for the row’s file and position in the data file, starting at 0.

Position-based delete files store file_position_delete, a struct with the following fields:

Field id, name	Type	Description
2147483546 file_path	string	Full URI of a data file with FS scheme. This must match the file_path of the target data file in a manifest entry
2147483545 pos	long	Ordinal position of a deleted row in the target data file identified by file_path, starting at 0
2147483544 row	required struct<...> [1]	Deleted row values. Omit the column when not storing deleted rows.
When present in the delete file, row is required because all delete entries must include the row values.
When the deleted row column is present, its schema may be any subset of the table schema and must use field ids matching the table.

To ensure the accuracy of statistics, all delete entries must include row values, or the column must be omitted (this is why the column type is required).

The rows in the delete file must be sorted by file_path then position to optimize filtering rows while scanning.

Sorting by file_path allows filter pushdown by file in columnar storage formats.
Sorting by position allows filtering rows while scanning, to avoid keeping deletes in memory.
## 位置删除
基于位置的删除文件通过一个或多个数据文件中的文件和位置标识已删除的行，并且可以选择包含已删除的行。

如果在位置删除文件中有一行文件和数据文件中位置(从0开始)的条目，则删除数据行。

基于位置的删除文件存储file_position_delete，该结构体包含以下字段:

字段id，名称类型描述
2147483546 file_path string具有FS方案的数据文件的完整URI。这必须与清单条目中的目标数据文件的file_path匹配
2147483545 pos long目标数据文件中由file_path标识的被删除行的序号位置，从0开始
2147483544 row required struct<…>[1]删除行值。当不存储已删除的行时，省略该列。
当在删除文件中出现时，row是必需的，因为所有的删除条目都必须包括行值。
当删除的行列出现时，它的模式可以是表模式的任何子集，并且必须使用与表匹配的字段id。

为了确保统计数据的准确性，所有的删除条目必须包括行值，或者必须省略列(这就是为什么需要列类型的原因)。

删除文件中的行必须按file_path和position排序，以便在扫描时优化过滤行。

按file_path排序允许按柱状存储格式的文件下推过滤器。
按位置排序允许在扫描时过滤行，以避免在内存中保留删除。


Scan Planning¶
Scans are planned by reading the manifest files for the current snapshot. Deleted entries in data and delete manifests are not used in a scan.

Manifests that contain no matching files, determined using either file counts or partition summaries, may be skipped.

For each manifest, scan predicates, which filter data rows, are converted to partition predicates, which filter data and delete files. These partition predicates are used to select the data and delete files in the manifest. This conversion uses the partition spec used to write the manifest file.

Scan predicates are converted to partition predicates using an inclusive projection: if a scan predicate matches a row, then the partition predicate must match that row’s partition. This is called inclusive [1] because rows that do not match the scan predicate may be included in the scan by the partition predicate.

For example, an events table with a timestamp column named ts that is partitioned by ts_day=day(ts) is queried by users with ranges over the timestamp column: ts > X. The inclusive projection is ts_day >= day(X), which is used to select files that may have matching rows. Note that, in most cases, timestamps just before X will be included in the scan because the file contains rows that match the predicate and rows that do not match the predicate.

Scan predicates are also used to filter data and delete files using column bounds and counts that are stored by field id in manifests. The same filter logic can be used for both data and delete files because both store metrics of the rows either inserted or deleted. If metrics show that a delete file has no rows that match a scan predicate, it may be ignored just as a data file would be ignored [2].

Data files that match the query filter must be read by the scan.

Delete files that match the query filter must be applied to data files at read time, limited by the scope of the delete file using the following rules.

A position delete file must be applied to a data file when all of the following are true:
The data file’s sequence number is less than or equal to the delete file’s sequence number
The data file’s partition (both spec and partition values) is equal to the delete file’s partition
An equality delete file must be applied to a data file when all of the following are true:
The data file’s sequence number is strictly less than the delete’s sequence number
The data file’s partition (both spec and partition values) is equal to the delete file’s partition or the delete file’s partition spec is unpartitioned
In general, deletes are applied only to data files that are older and in the same partition, except for two special cases:

Equality delete files stored with an unpartitioned spec are applied as global deletes. Otherwise, delete files do not apply to files in other partitions.
Position delete files must be applied to data files from the same commit, when the data and delete file sequence numbers are equal. This allows deleting rows that were added in the same commit.
Notes:

An alternative, strict projection, creates a partition predicate that will match a file if all of the rows in the file must match the scan predicate. These projections are used to calculate the residual predicates for each file in a scan.
For example, if file_a has rows with id between 1 and 10 and a delete file contains rows with id between 1 and 4, a scan for id = 9 may ignore the delete file because none of the deletes can match a row that will be selected.

