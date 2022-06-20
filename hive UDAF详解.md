hive udaf的编写主要用到两个类，GenericUDAFEvaluator(udaf求值器)和AbstractGenericUDAFResolver(udaf分解器或翻译为解决器)
AbstractGenericUDAFResolver的主要作用是获取求值器，类结构如下，
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200304201621916.png)
GenericUDAFEvaluator的作用是求值，其方法分别对应于mapreduce不同的阶段，类结构如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200304202208129.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
其中
- Mode是个enum类，用于代表UDAF在mapreduce的各个阶段。
- AggregationBuffer用于存放聚集的中间结果
- 剩下的方法对应mapreduce的各阶段
>Mode各值的含义
```java
public static enum Mode {  

    /** 

     * PARTIAL1: 这个是mapreduce的map阶段:从原始数据到部分数据聚合 

     * 将会调用iterate()和terminatePartial() 

     */  

    PARTIAL1,  

        /** 

     * PARTIAL2: 这个是mapreduce的map端的Combiner阶段，负责在map端合并map的数据::从部分数据聚合到部分数据聚合: 

     * 将会调用merge() 和 terminatePartial()  

     */  

    PARTIAL2,  

        /** 

     * FINAL: mapreduce的reduce阶段:从部分数据的聚合到完全聚合  

     * 将会调用merge()和terminate() 

     */  

    FINAL,  

        /** 

     * COMPLETE: 如果出现了这个阶段，表示mapreduce只有map，没有reduce，所以map端就直接出结果了:从原始数据直接到完全聚合 

      * 将会调用 iterate()和terminate() 

     */  

    COMPLETE  

  };  
```
>各方法分别对应的阶段
```java
// 确定各个阶段输入输出参数的数据格式ObjectInspectors  
public  ObjectInspector init(Mode m, ObjectInspector[] parameters) throws HiveException;  


// 保存数据聚集结果的类  
abstract AggregationBuffer getNewAggregationBuffer() throws HiveException;  


// 重置聚集结果  
public void reset(AggregationBuffer agg) throws HiveException;  

  
// map阶段，迭代处理输入sql传过来的列数据  
public void iterate(AggregationBuffer agg, Object[] parameters) throws HiveException;  

 
// map与combiner结束返回结果，得到部分数据聚集结果  
public Object terminatePartial(AggregationBuffer agg) throws HiveException;  

  
// combiner合并map返回的结果，还有reducer合并mapper或combiner返回的结果。  
public void merge(AggregationBuffer agg, Object partial) throws HiveException;  

  
// reducer阶段，输出最终结果  
public Object terminate(AggregationBuffer agg) throws HiveException;  
```
一张图说明Mode和各方法的对应关系
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200305113926949.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
代码见https://github.com/cuiwenxu/hive-extension-examples
