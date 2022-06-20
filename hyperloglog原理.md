
# 思路概述
抛硬币的概率->字符串取hash之后也是01序列，也可看做抛硬币的结果->抛几次（集合中有几个元素）会出现这种前导0->不精确，所以分桶->前导0的个数直接求平均偏差大，所以求调和平均->细节微调（分桶数、constant值等）

![在这里插入图片描述](https://img-blog.csdnimg.cn/b711d33b54fb48ca895df12c4902e22e.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)

