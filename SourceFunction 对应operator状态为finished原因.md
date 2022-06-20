SourceFunction具体的实现类的run方法完成后会调用`OperatorChain.endInput`方法，将operator状态标记为finished
下图代码来自org.apache.flink.streaming.api.operators.StreamSource
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200721220125723.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)



