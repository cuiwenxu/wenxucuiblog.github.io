
|  变量名| 描述 |默认值|
|--|--|--|
|  hive.partition.pruning| 分区裁剪检查，查询分区表时如果不指定分区编译器是否报错（A strict value for this variable indicates that an error is thrown by the compiler in case no partition predicate is provided on a partitioned table. This is used to protect against a user inadvertently issuing a query against all the partitions of the table.） |  nonstrict | 
| hive.map.aggr | map断是否做聚合（Determines whether the map side aggregation is on or not.） |  true | 
| hive.default.fileformat | 创建表的默认存储格式（Default file format for CREATE TABLE statement. Options are TextFile, SequenceFile, RCFile, and Orc.） | TextFile  | 
|hive.merge.mapfiles  | 在仅有map的job结束后是否合并小文件（Merge small files at the end of a map-only job.） |  true | 
| hive.merge.mapredfiles | mapred job结束后是否做小文件合并（Merge small files at the end of a map-reduce job.） | false  | 
| hive.merge.size.per.task | 任务结束后合并文件的大小（Size of merged files at the end of the job.） | 256000000（大约256M）  | 
| hive.merge.smallfiles.avgsize | 当任务结束后输出文件小于该值时，hive会附带启动一个mr job将小文件合并成大文件，这个只有在hive.merge.mapfiles=true(map only任务)和hive.merge.mapredfiles=true(mapred任务)时才生效（When the average output file size of a job is less than this number, Hive will start an additional map-reduce job to merge the output files into bigger files. This is only done for map-only jobs if hive.merge.mapfiles is true, and for map-reduce jobs if hive.merge.mapredfiles is true.） |  16000000（大约16M） | 
| hive.querylog.location | hive query log的存放目录，hive的每个session都会在该目录下创建一个文件，如果这个变量设置为空，则hive的query log不会生成（Directory where structured hive query logs are created. One file per session is created in this directory. If this variable set to empty string structured log will not be created.） |  /tmp/<user.name> | 

参考自：https://cwiki.apache.org/confluence/display/Hive/AdminManual+Configuration

