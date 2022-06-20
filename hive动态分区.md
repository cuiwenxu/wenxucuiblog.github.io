通过下面命令开启动态分区
```
set hive.exec.dynamic.partition=true;
set hive.exec.dynamic.partition.mode=nonstrict;
```
执行下面语句，`需要注意的是，在SELECT子句的最后两个字段，必须对应前面PARTITION (brand,city,site,dt)中指定的分区字段，包括顺序<`
insert overwrite table test.ods_store_user_events partition (brand,city,site,dt) 
select pid,gender,age,image_url,event_type,start_time,end_time,region_id,region_type,in_door_id,out_door_id,staff_id,price,sku,brand,city,site,dt
from test.ods_store_user_events_no_part;
