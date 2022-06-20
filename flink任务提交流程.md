flink任务提交流程图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020071320544167.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
Flink本身也是具有ResourceManager和TaskManager的，这里虽然是on Yarn模式，但Flink本身也是拥有一套资源管理架构，虽然各个组件的名字一样，但这里yarn只是一个资源的提供者，若是standalone模式，资源的提供者就是物理机或者虚拟机了
