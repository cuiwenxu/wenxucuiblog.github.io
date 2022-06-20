![在这里插入图片描述](https://img-blog.csdnimg.cn/20210207145345581.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)



维表的每个变化都会触发事实表的计算（使用普通的inner join，事实表select from当天，作出一个视图，两个视图进行join），把join的结果写到kafka upsert table表中，然后消费kafka，统一sink到table中。

