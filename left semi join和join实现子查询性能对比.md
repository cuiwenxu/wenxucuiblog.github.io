@[TOC](left semi join详解)
# 实现原理
**hive从[ 0.13](https://issues.apache.org/jira/browse/HIVE-784) 版本实现了in/exist自查询，实现方式是通过left semi join，left semi jion具体实现是在右表（子查询）中先对on的关键字做group by ，然后再做join操作**

```sql
select a.*
from
(
select brand,store_id
from dw.dim_store_info_current
) a 
left semi join
(
select brand
from dw.dim_brand_business_info_current
) b 
on a.brand=b.brand
```

下图为explain的截图

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-5bDfcZf0-1581079185868)(left_semi_join_explain.png)\]](https://img-blog.csdnimg.cn/20200209194814377.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
# left semi join和join实现子查询性能对比
**因为多了一步group by,所以当子查询的表重复记录较少的情况下，用join实现子查询性能更高**
# left semi join和left join区别
- 区别一：
left join遇到右表重复时，结果集会产生多条记录。而left semi join因为是先做group by，所以即使右表有重复数据，得到的结果集也不会重复。
- 区别二：
left join可以取到右表的所有字段，但是left semi join因为是先做group by，所以只能取到右表的关联字段（即group by的维度字段）


