本节详细介绍Iceberg如何进行行级删除。`v1中不支持行级删除。`

行级删除文件是有效的Iceberg数据文件:文件必须使用有效的Iceberg格式、schema和column projecttion。建议使用表的默认文件格式写入删除文件。

行级删除文件由manifest文件（清单）跟踪。删除文件使用一组相互独立的清单，但清单的schema是相同的。

position delete（位置删除）和equality delete（相等删除）都允许用delete对已删除的行值进行编码。这可用于重建表的更改流。
# row delete files(基于位置的delete files)
>注意理解这里的delete files定义：delete files是记录被删除的行的文件，而不是指被删除的文件，delete files类似于一种字典，用于记录被删除的行

delete files通过文件和位置（这两个坐标）标识被删除的行。

基于位置的delete files存储file_position_delete这种结构体，该结构体包含以下字段:

Field id, name	| Type	| Description
-------- | -------- | --------
2147483546 file_path	| string	| Full URI of a data file with FS scheme. This must match the file_path of the target data file in a manifest entry
2147483545 pos	| long	| Ordinal position of a deleted row in the target data file identified by file_path, starting at 0
2147483544 row	| required struct<...> 	| Deleted row values. Omit the column when not storing deleted rows.


delete files的行必须按file_path和position排序，以便在扫描时优化过滤行。

- 按file_path排序允许按列式存储格式的文件下推过滤器。
- 按position排序允许在扫描时过滤行，以避免在内存中保留删除的内容（浪费内存）。
# equality delete files(基于列值相等的delete files)
equality delete files通过一个或多个列值标识数据文件集合中已删除的行，并且可以选择包含已删除行的附加列。

A data row is deleted if its values are equal to all delete columns for any row in an equality delete file that applies to the row’s data file (see Scan Planning).

Each row of the delete file produces one equality predicate that matches any row where the delete columns are equal. Multiple columns can be thought of as an AND of equality predicates. A null value in a delete column matches a row if the row’s value is null, equivalent to col IS NULL.

