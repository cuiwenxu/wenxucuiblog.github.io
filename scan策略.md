Scans are planned by reading the manifest files for the current snapshot. Deleted entries in data and delete manifests are not used in a scan.

Manifests that contain no matching files, determined using either file counts or partition summaries, may be skipped.

For each manifest, scan predicates, which filter data rows, are converted to partition predicates, which filter data and delete files. These partition predicates are used to select the data and delete files in the manifest. This conversion uses the partition spec used to write the manifest file.

Scan predicates are converted to partition predicates using an inclusive projection: if a scan predicate matches a row, then the partition predicate must match that row’s partition. This is called inclusive [1] because rows that do not match the scan predicate may be included in the scan by the partition predicate.

For example, an events table with a timestamp column named ts that is partitioned by ts_day=day(ts) is queried by users with ranges over the timestamp column: ts > X. The inclusive projection is ts_day >= day(X), which is used to select files that may have matching rows. Note that, in most cases, timestamps just before X will be included in the scan because the file contains rows that match the predicate and rows that do not match the predicate.

Scan predicates are also used to filter data and delete files using column bounds and counts that are stored by field id in manifests. The same filter logic can be used for both data and delete files because both store metrics of the rows either inserted or deleted. If metrics show that a delete file has no rows that match a scan predicate, it may be ignored just as a data file would be ignored [2].

Data files that match the query filter must be read by the scan.

Delete files that match the query filter must be applied to data files at read time, limited by the scope of the delete file using the following rules.

扫描是通过读取当前快照的清单文件来计划的。在扫描中不会使用数据和删除清单中的已删除条目。

可以跳过使用文件计数或分区摘要确定的不包含匹配文件的清单。

对于每个清单，用于筛选数据行的扫描谓词被转换为用于筛选数据和删除文件的分区谓词。这些分区谓词用于选择清单中的数据和删除文件。此转换使用用于编写清单文件的分区规范。

扫描谓词使用包含式投影转换为分区谓词:如果扫描谓词匹配一行，则分区谓词必须匹配该行的分区。这称为包容性[1]，因为不匹配扫描谓词的行可能会被分区谓词包含在扫描中。

例如，一个带有时间戳列ts的事件表被分区为ts_day=day(ts)，用户使用时间戳列的范围查询该事件表:ts > X。包含投影是ts_day >= day(X)，它用于选择可能有匹配行的文件。注意，在大多数情况下，X之前的时间戳将包含在扫描中，因为文件包含与谓词匹配的行和与谓词不匹配的行。

扫描谓词还用于使用列边界和计数(按清单中的字段id存储)筛选数据和删除文件。数据和删除文件可以使用相同的筛选逻辑，因为两者都存储插入或删除行的度量。如果指标显示删除文件没有匹配扫描谓词的行，那么它可能会被忽略，就像数据文件被忽略[2]一样。

匹配查询筛选器的数据文件必须被扫描读取。

匹配查询筛选器的删除文件必须在读取时应用于数据文件，受到使用以下规则的删除文件范围的限制。
