>问题：线上一个慢sql扫了两千多万行
>![在这里插入图片描述](https://img-blog.csdnimg.cn/20201210115036920.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)

排查思路：怀疑where后的条件没走索引，explain sql后发现type=all,possiblee_key为null，确实没走索引
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201210120146611.png)
show index from table发现表根本没建索引，仅pk
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020121012032989.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
所以需要根据where条件创建索引，创建之后开始走索引了，扫的行数降到了300w行
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201210120445792.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)

