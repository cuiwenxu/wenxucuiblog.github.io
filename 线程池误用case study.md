# 事故过程记录
1. 研发收到压测通知，梳理性能瓶颈点时，发现线程池配置可能存在性能风险。调整线程池参数，将queueCapacity由200调整为1
2. 收到线上接口调用失败率告警
3. 开始排查问题，发现线程池报大量reject异常（java.util.concurrent.RejectedExecutionException），导致接口调用失败率飙升![在这里插入图片描述](https://img-blog.csdnimg.cn/563945256b144cd7936bb11cac3f472b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bem5p6X5Y-z5p2OMDI=,size_20,color_FFFFFF,t_70,g_se,x_16)
5. 回滚线程池配置参数，将queueCapacity由1恢复为1
6. 报警日志逐渐减少，接口调用成功率恢复正常

# 复盘
1. 为什么要调整queueCapacity由100调整为1？
因为当时看到线程池核心线程数（256），最大线程数=2000，担心遇到流量突刺时，不增加线程数，而是将任务放入队列，队列中的任务可能会因为等待而超时。
2. 为什么会报reject异常？
该场景为CPU密集型，将线程池的等待队列调整为1后，导致并发线程数量在超过coreThreadCount后继续创建新线程，并很快使线程数量达到maxThreadCount，最终导致reject异常，下图为Java原生线程池创建线程的流程图
![在这里插入图片描述](https://img-blog.csdnimg.cn/b191b013816641f4ada7e246498a5d48.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bem5p6X5Y-z5p2OMDI=,size_20,color_FFFFFF,t_70,g_se,x_16)

# 总结
1. 线程池的细节很多，调整不当很容易踩坑，那么线程池的等待队列大小怎么调整，什么时候设置大一些，什么时候设置小？
> 可以分两类来看
> 1. CPU密集型，将queueCapacity调大一些，减少线程上下文切换时间
> 因为执行CPU密集型的任务时CPU比较繁忙，因此只需要创建和CPU核数相当的线程就好了，多了反而会造成线程上下文切换，降低任务执行效率。所以当前线程数超过核心线程数时，线程池不会增加线程，而是放在队列里等待核心线程空闲下来。
> 2. IO型，将queueCapacity调小一些，避免线程池不满但是任务却一直不执行的诡异现象
我们平时开发的Web系统通常都有大量的IO操作，比方说查询数据库、查询缓存等等。任务在执行IO操作的时候CPU就空闲了下来，这时如果增加执行任务的线程数而不是把任务暂存在队列中，就可以在单位时间内执行更多的任务，大大提高了任务执行的吞吐量。所以你看Tomcat使用的线程池就不是JDK原生的线程池，而是做了一些改造，当线程数超过coreThreadCount之后会优先创建线程，直到线程数到达maxThreadCount，这样就比较适合于Web系统大量IO操作的场景了，你在实际使用过程中也可以参考借鉴。

