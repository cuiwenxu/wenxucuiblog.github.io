火焰图是一个可视化的，有效地表示如下问题的答案，如:

- 哪些方法目前正在消耗CPU资源?
- 一种方法的消费与其他方法相比如何?
- 堆栈上的哪一系列调用导致执行一个特定的方法?
![在这里插入图片描述](https://img-blog.csdnimg.cn/e5ecd7db3266409fbf27b20b7423e111.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
火焰图是通过多次采样stack trace来生成的。每个方法调用都由一个条形图表示，条形图的长度与它在样本中出现的次数成正比。

从Flink 1.13开始，Flink本身就支持Flame Graphs。为了生成一个火焰图，导航到一个正在运行的作业的作业图，选择一个感兴趣的operator，在菜单中右键单击火焰图选项卡:
![在这里插入图片描述](https://img-blog.csdnimg.cn/9666d46529384b28b8afed356c4c4210.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
>任何测量过程本身都会不可避免地影响被测量对象(参见双分裂实验)。CPU stack trace采样也不例外。为了防止对生产环境的意外影响，Flame Graphs目前作为一个可选择的功能。要启用它，你需要在conf/flink-conf.yaml中设置rest.flamgraph.enabled: true。我们建议在开发和预生产环境中启用它，但您应该将它作为生产环境中的试验性特性。

除了cpu上的火焰图，也可以使用非cpu和混合可视化，可以使用面板顶部的选择器进行切换:
![在这里插入图片描述](https://img-blog.csdnimg.cn/46df79ee0279487c86b9d27bf3d33528.png)
Off-CPU Flame Graph显示了在样本中发现的阻塞调用。区别如下:

- On-CPU: Thread.State in [RUNNABLE, NEW]
- Off-CPU: Thread.State in [TIMED_WAITING, WAITING, BLOCKED]

![在这里插入图片描述](https://img-blog.csdnimg.cn/d42bd4c3c3fe4204b4bab3bb7cf96f91.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)

