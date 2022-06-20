A manifest is an immutable Avro file that lists data files or delete files, along with each file’s partition data tuple, metrics, and tracking information. One or more manifest files are used to store a snapshot, which tracks all of the files in a table at some point in time. Manifests are tracked by a manifest list for each table snapshot.

A manifest is a valid Iceberg data file: files must use valid Iceberg formats, schemas, and column projection.

A manifest may store either data files or delete files, but not both because manifests that contain delete files are scanned first during job planning. Whether a manifest is a data manifest or a delete manifest is stored in manifest metadata.

A manifest stores files for a single partition spec. When a table’s partition spec changes, old files remain in the older manifest and newer files are written to a new manifest. This is required because a manifest file’s schema is based on its partition spec (see below). The partition spec of each manifest is also used to transform predicates on the table’s data rows into predicates on partition values that are used during job planning to select files from a manifest.

A manifest file must store the partition spec and other metadata as properties in the Avro file’s key-value metadata:

清单是一个不可变的 Avro 文件，它列出了数据文件或删除文件，以及每个文件的分区数据元组、指标和跟踪信息。一个或多个清单文件用于存储快照，该快照在某个时间点跟踪表中的所有文件。清单由每个表快照的清单列表跟踪。

清单是一个有效的 Iceberg 数据文件：文件必须使用有效的 Iceberg 格式、模式和列投影。

清单可以存储数据文件或删除文件，但不能同时存储两者，因为在作业计划期间首先扫描包含删除文件的清单。清单是数据清单还是删除清单都存储在清单元数据中。

清单存储单个分区规范的文件。当表的分区规范更改时，旧文件保留在旧清单中，而新文件将写入新清单。这是必需的，因为清单文件的架构基于其分区规范（见下文）。每个清单的分区规范还用于将表数据行上的谓词转换为分区值上的谓词，这些值在作业规划期间用于从清单中选择文件。

清单文件必须将分区规范和其他元数据作为属性存储在 Avro 文件的键值元数据中：
