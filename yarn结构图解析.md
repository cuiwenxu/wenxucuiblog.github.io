yarn通过两类长期运行的守护进程提供自己的核心服务：管理集群上资源使用的资源管理器(resource manager)、运行在集群中所有节点上且能够启动和监控容器(container)的节点管理器(node manager)。容器用于执行特定应用程序的进程，每个容器都有资源限制(内存、CPU等)。图4-2描述了YARN是如何运行一个应用的。
为了在YARN上运行一个应用，首先，客户端联系资源管理器，要求它运行一个application master进程(下图中的步骤1)。然后，资源管理器找到一个能够在容器中启动application master的节点管理器(步骤2a和2b)。准确地说，application master一旦运行起来后能做些什么依赖于应用本身。有可能是在所处的容器中简单地运行一个计算，并将结果返回给客户端；或是向资源管理器请求更多的容器(步骤3)，以用于运行一个分布式计算(步骤4a和4b)。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210308184314995.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
>(ResourceManager和NodeManager)与(ApplicationMaster和Container)的联系与区别
>ResourceManager和NodeManager对应yarn的两个守护进程，是yarn为其上运行的各应用的基础服务。而ApplicationMaster和Container则是具体应用的概念。

flink job 对应到yarn上分别是什么
spark job？
