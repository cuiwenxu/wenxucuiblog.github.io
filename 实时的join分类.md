# 实时的join分类
## 最直观的想法，无界变有界：Window Join
### 原理
利用 Flink 时间窗口机制
只考虑在同一个时间窗口内的数据进行关联
### 存在的问题
  - 1.窗口边界上的数据关联失败， 比如：10:59 的曝光，11:01 的点击，互相关联不上
     
  - 2.时效性差，窗口结束才触发计算和下发
  
![在这里插入图片描述](https://img-blog.csdnimg.cn/8b2c9996259c41688ebb6480f5daf1d4.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bem5p6X5Y-z5p2OMDI=,size_20,color_FFFFFF,t_70,g_se,x_16)
## 准确性优先：Interval Join
### 原理
利用 Flink 的 Watermark 机制，当前侧的点，关联对侧的时间区间
### 存在的问题
- 1.时效性较差，Outer 补 null 的数据要等区间结束才下发
- 2.状态过期逻辑需额外处理（在过期时补 null 下发）

![在这里插入图片描述](https://img-blog.csdnimg.cn/922081964b5d4ee48fdd5fa77cbcc290.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bem5p6X5Y-z5p2OMDI=,size_20,color_FFFFFF,t_70,g_se,x_16)
## 时效性优先：Regular Join（Streaming Join） 
### 原理
利用 Flink Table 的回撤重发机制
- 1.每条数据与对侧当前已到达的所有数据关联，得到的都是“当下”的正确结果
### 存在的问题
- 1.准确”是暂时的，下发的并不是最终结果
- 2.回撤重发机制导致数据量放大
- 3.依赖全局的状态清理策略（TTL）

![在这里插入图片描述](https://img-blog.csdnimg.cn/62aa641087a349b3a61ddfafea71a4c1.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bem5p6X5Y-z5p2OMDI=,size_20,color_FFFFFF,t_70,g_se,x_16)
## 综合比较
- 没有完美方案
- 不同场景下需要做不同取舍
- 不同的取舍下有不同的优化方案

![在这里插入图片描述](https://img-blog.csdnimg.cn/1209bbdc214e4db589296a63d2a95b48.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bem5p6X5Y-z5p2OMDI=,size_20,color_FFFFFF,t_70,g_se,x_16)

