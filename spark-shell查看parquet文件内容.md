1、进入spark shell

`./spark-shell`



2、执行以下操作读取parquet文件

```java
val sqlContext = new org.apache.spark.sql.SQLContext(sc)

val parquetFile = sqlContext.parquetFile("/user/hive/warehouse/ods.db/ods_mall_lite_pc_v3/brand=tgdc/city=beijing/site=ydg/dt=2020-04-21/process_date=2020-04-22/part-00001-8013515e-4871-4f32-93bd-1d46af59d590.c000.snappy.parquet")
```

3、打印具体内容

`parquetFile.take(200).foreach(println)`


