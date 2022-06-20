
@[TOC](flink sql转换为transformation过程解析)
# 概览
一段sql转换为transformation的过程中的对象转换如下
>sql statement->SqlNode->Operation->RelNode->Transformation

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210412162250852.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)

