@[TOC](synchronized锁升级过程)
# 偏向锁获取
偏向锁获取流程如下图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210319110935299.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
当线程访问同步块并获取锁时处理流程如下：
- 1、检查 mark word 的线程 id 。
- 2、如果为空则设置 CAS 替换当前线程 id。如果替换成功则获取锁成功，如果失败则撤销偏向锁。
- 3、如果不为空则检查线程id为是否为本线程。如果是则获取锁成功，如果失败则撤销偏向锁。

持有偏向锁的线程以后每次进入这个锁相关的同步块时，只需比对一下 mark word 的线程 id 是否为本线程，如果是则获取锁成功。

如果发生线程竞争发生 2、3 步失败的情况则需要撤销偏向锁。




