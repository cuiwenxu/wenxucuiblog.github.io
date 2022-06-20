@[TOC](spark job调度)
# 跨application的调度
在集群上运行时，每个Spark应用程序都会获得一组独立的执行器JVM，它们仅运行该应用程序的任务并存储数据。 如果多个用户需要共享您的集群，则有不同的选项来管理分配，具体取决于集群管理器。

所有群集管理器上可用的最简单的选项是资源的静态分区。 使用这种方法，将为每个应用程序分配最大的可用资源，并在整个使用期间保留这些资源。 这是Spark的standalone模式和YARN模式以及粗粒度的Mesos模式中使用的方法。 可以根据集群类型如下配置资源分配：

- standalone模式：默认情况下，提交到独立模式集群的应用程序将以FIFO（先进先出）顺序运行，并且每个应用程序将尝试使用所有可用节点。您可以通过在应用程序中设置`spark.cores.max`配置属性来限制应用程序使用的节点数，或者通过`spark.deploy.defaultCores`更改未设置此应用程序的应用程序的默认值。最后，除了控制core的个数之外，每个应用程序的`spark.executor.memory`设置还控制其内存使用情况。
- Mesos：要在Mesos上使用静态分区，请在独立模式下将`spark.mesos.coarse`配置属性设置为true，并可以选择将`spark.cores.max`设置为限制每个应用程序的资源共享。您还应该设置`spark.executor.memory`来控制执行者的内存。
- YARN：Spark YARN客户端的`--num-executors`选项控制它将在群集上分配多少executor（spark.executor.instances作为配置属性），而`--executor-memory`（spark.executor.memory配置属性）和`--executor-cores`（spark.executor.cores配置属性）控制每个executor的资源。有关更多信息，请参见YARN Spark属性。
##  动态资源分配

Spark提供了一种机制，可以根据工作负载动态调整应用程序占用的资源。 这意味着，如果不再使用资源，您的应用程序可以将资源返还给群集，并在以后有需求时再次请求它们。 如果多个应用程序共享您的Spark集群中的资源，则此功能特别有用。

默认情况下，此功能是禁用的，但是在所有粗粒度群集管理器（即standalone模式，YARN模式和Mesos粗粒度模式）上均可用。
### 配置
使用此功能有两个要求。首先，您的应用程序必须将`spark.dynamicAllocation.enabled`设置为true。其次，您必须在同一集群中的每个工作节点上设置一个外部shuffle服务，并在应用程序中将`spark.shuffle.service.enabled`设置为true。开启外部shuffle的目的是允许executor被移除而无需删除其shuffle文件（下面将详细介绍）。设置此服务的方式因群集管理器而异：

- standalone模式下，只需将`spark.shuffle.service.enabled`设置为true即可启动您的工作程序。

- Mesos粗粒度模式下，在将`spark.shuffle.service.enabled`设置为true的所有辅助节点上运行
> $ SPARK_HOME/sbin/start-mesos-shuffle-service.sh。

- 在YARN模式下，请按照[此处](https://spark.apache.org/docs/latest/running-on-yarn.html#configuring-the-external-shuffle-service)的说明进行操作。

所有其他相关配置都是可选的，并且在`spark.dynamicAllocation.*`和`spark.shuffle.service.*`名称空间下。有关更多详细信息，请参见[配置页面](https://spark.apache.org/docs/latest/configuration.html#dynamic-allocation)。
### 资源分配策略
从较高的层次上讲，Spark应该在不再使用执行者时放弃他们，并在需要它们时收购他们。 由于没有明确的方法来预测即将被删除的执行程序是否将在不久的将来运行任务，或者即将被添加的新执行程序实际上将处于空闲状态，因此我们需要一组启发式方法来确定 何时删除并要求执行人。
#### 请求策略
#### 移除策略
# 单application内的调度
一个application分为若干job，默认情况下，Spark的调度程序以FIFO方式运行作业。每个job又分为若干stages（例如，map和reduce阶段），第一个job在所有可用资源上都具有最高优先级，第一个job完成后，则第二个作业具有最高优先级，依此类推。如果队列头部job不需要使用整个集群，那么后面的job可以立即开始运行，但是如果队列开头的job很大，则以后的job可能会大大延迟。

从Spark 0.8开始，还可以配置job之间的公平共享。在公平共享下，Spark以“轮询”的方式在job之间分配资源，以便所有作业都获得大致相等的集群资源份额。这意味着在运行长job时提交的短作业可以立即开始接收资源，并且仍然获得良好的响应时间，而无需等待长作业完成。此模式最适合多用户场景。

要启用fair scheduler，只需在配置SparkContext时将spark.scheduler.mode属性设置为FAIR即可：
```scala
val conf = new SparkConf().setMaster(...).setAppName(...)
conf.set("spark.scheduler.mode", "FAIR")
val sc = new SparkContext(conf)
```
## fair scheduler pool（公平调度池）
fair scheduler模式还支持将job分组到池中，并为每个池设置不同的调度选项（例如权重）。 例如，这对于创建用于更重要的作业的“高优先级”池或将每个用户的作业分组在一起并为用户提供相等的份额而不管他们有多少并发作业。 该方法以Hadoop Fair Scheduler为蓝本。

不加干预情况下，新提交的作业将进入默认池，但是可以通过在提交线程的SparkContext中将`spark.scheduler.pool`属性添加到SparkContext中来设置作业的池。 这样做如下：
```scala
// Assuming sc is your SparkContext variable
sc.setLocalProperty("spark.scheduler.pool", "pool1")
```
设置此本地属性后，在此线程内提交的所有作业（通过在该线程中调用RDD.save，计数，收集等）都将使用该池。 该设置是按线程设置的，以使一个线程轻松代表同一用户运行多个作业。 如果您想清除与线程关联的池，只需调用：
```scala
sc.setLocalProperty("spark.scheduler.pool", null)
```
## 池的默认行为
默认情况下，每个池都获得集群的相等份额，在每个池中，作业以FIFO顺序运行。 例如，如果您为每个用户创建一个池，则意味着每个用户将获得集群的平等份额，并且每个用户的查询将按顺序运行。
## 配置池属性
特定池的属性也可以通过配置文件进行修改。每个池支持三个属性：
- scheduleMode：可以是FIFO或FAIR，以控制池中的作业是彼此排在后面（默认），还是公平地共享池的资源。
- weight(权重)：控制相对于其他池的池在集群中的份额。默认情况下，所有池的权重均为1。例如，如果给特定池赋予权重2，则它将获得比其他活动池多2倍的资源。设置较高的权重（例如1000）还可以在池之间实现优先级-本质上，权重1000的池总是在作业处于活动状态时首先启动任务。
- minShare：除了权重以外，还可以为每个池分配管理员希望拥有的最小份额（作为CPU核心数）。公平的调度程序总是尝试满足所有活动池的最小份额，然后根据权重重新分配额外的资源。因此，==minShare属性可以是确保池始终快速获取特定数量的资源（例如10个核心）的一种方式==，默认情况下，每个池的minShare为0。


可以通过创建类似于`conf/fairscheduler.xml.template`的XML文件，或者在类路径上放置名为`fairscheduler.xml`的文件，还可以在SparkConf中设置`spark.scheduler.allocation.file`属性来设置池属性。
>conf.set("spark.scheduler.allocation.file", "/path/to/file")

XML文件的格式只是每个池的<pool>元素，其中的各种设置都有不同的元素。 例如：
```xml
<?xml version="1.0"?>
<allocations>
  <pool name="production">
    <schedulingMode>FAIR</schedulingMode>
    <weight>1</weight>
    <minShare>2</minShare>
  </pool>
  <pool name="test">
    <schedulingMode>FIFO</schedulingMode>
    <weight>2</weight>
    <minShare>3</minShare>
  </pool>
</allocations>
```
## 使用JDBC连接进行调度
要为JDBC客户端会话设置Fair Scheduler池，用户可以设置`spark.sql.thriftserver.scheduler.pool`变量：
> SET spark.sql.thriftserver.scheduler.pool=accounting;



