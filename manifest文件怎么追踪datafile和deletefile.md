@[TOC](manifest文件怎么追踪datafile和deletefile)
# 概述
先说结论，通过manifest list和manifest这两个索引文件来追踪
# manifest 文件的两种格式
manifest是avro格式的文件，根据其命名和功能可以分为两种：
## 1：snap-{snapshot_id}-{随机字符串}.avro （manifest list文件）
- 功能：manifest list文件，用于记录manifest文件的路径                                                                                    
- 命名样例：snap-231200337564129518-1-f466c6ac-91f8-477b-8011-9e203b92e6c3.avro
- 内容样例： 

|manifest_path                                                                                                  |manifest_length|partition_spec_id|content|sequence_number|min_sequence_number|added_snapshot_id  |added_data_files_count|existing_data_files_count|deleted_data_files_count|added_rows_count|existing_rows_count|deleted_rows_count|partitions     |
|---------------------------------------------------------------------------------------------------------------|---------------|-----------------|-------|---------------|-------------------|-------------------|----------------------|-------------------------|------------------------|----------------|-------------------|------------------|---------------|
|/Users/cuiwenxu/Documents/restore/icebergfiles/testupsert/metadata/4329c3ed-a11c-4538-aae7-662fffff7919-m0.avro|6551           |0                |0      |2              |2                  |7348347678694955289|1                     |0                        |0                       |1               |0                  |0                 |[[false, a, a]]|
|/Users/cuiwenxu/Documents/restore/icebergfiles/testupsert/metadata/612d17b8-9805-4857-94b9-c769c86dcebb-m0.avro|6553           |0                |0      |1              |1                  |5534618873695917606|1                     |0                        |0                       |1               |0                  |0                 |[[false, a, a]]|
|/Users/cuiwenxu/Documents/restore/icebergfiles/testupsert/metadata/4329c3ed-a11c-4538-aae7-662fffff7919-m1.avro|6554           |0                |1      |2              |2                  |7348347678694955289|1                     |0                        |0                       |1               |0                  |0                 |[[false, a, a]]|

## 2：{随机字符串}-m{0或1}.avro（存放datafile、deletefile具体路径的文件）
- 功能：存放datafile、deletefile具体路径的文件
- 命名样例：4329c3ed-a11c-4538-aae7-662fffff7919-m0.avro
- 内容样例： 

|status|snapshot_id        |sequence_number|data_file     |                                                                                                                                                                                                                                                        
|------|-------------------|---------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|1     |7348347678694955289|null           | [2, /Users/cuiwenxu/Documents/restore/icebergfiles/testupsert/data/user_name=a/00000-4-379a5fe1-cb5c-4dea-af8e-a1eb3d3ac434-00004.parquet, PARQUET, [a], 1, 681, [[1, 52], [2, 51]], [[1, 1], [2, 1]], [[1, 0], [2, 0]], [], [[1, a], [2, 2]], [[1, a], [2, 2]],,, [1]]|


### 细节：
- 1、datafile和deletefile的路径统一都存放在data_file这一列，通过int标志位区分：
> int with meaning: 0: DATA, 1: POSITION DELETES, 2: EQUALITY DELETES
- 2、manifest list文件名中的随机字符串和manifest文件名中的随机字符串值是相同的。






 

