##### FROM_UNIXTIME(`timestamp`/1000)
```sql
SELECT FROM_UNIXTIME(`timestamp`/1000),substr(FROM_UNIXTIME(`timestamp`/1000),1,10),`timestamp`
FROM face_event
WHERE site_id='dyc_tianjin_nk'
AND substr(FROM_UNIXTIME(`timestamp`/1000),1,10)='2020-02-26'
limit 100
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200302180248568.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
