SLA（Service-Level Agreement），也就是服务等级协议，指的是系统服务提供者（Provider）对客户（Customer）的一个服务承诺。这是衡量一个大型分布式系统是否“健康”的常见方法。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210614224221759.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
在开发设计系统服务的时候，无论面对的客户是公司外部的个人、商业用户，还是公司内的不同业务部门，我们都应该对自己所设计的系统服务有一个定义好的 SLA。因为 SLA 是一种服务承诺，所以指标可以多种多样。常见的SLA 指标如：可用性、准确性、系统容量和延迟。

实时计算最常用的SLA指标是延迟，即统计端到端的数据延迟。
