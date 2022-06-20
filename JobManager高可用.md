@[TOC](JobManager高可用)
# 概览
JobManager协调每个Flink部署。 它负责调度和资源管理。默认情况下，每个Flink集群只有一个JobManager实例。 这很容易出现单点故障（SPOF）：如果JobManager崩溃，则无法提交任何新程序，并且正在运行的程序也会失败。
使用JobManager高可用性，您可以从JobManager故障中恢复，从而消除SPOF。 您可以为独立群集和YARN群集配置高可用性。在链接: [Flink Wiki](https://cwiki.apache.org/confluence/display/FLINK/JobManager+High+Availability)中的JobManager高可用性中查看更多HA实现细节。
# 独立群集高可用性
独立集群的JobManager高可用性的总体思想是，随时可以有一个lead的JobManager，并有多个备用JobManager在leader失败的情况下接管。 这样可以确保没有单点故障，并且只要待机JobManager处于leader地位，程序就可以正常运行。 备用JobManager实例和主JobManager实例之间没有明显区别。 每个JobManager都可以充当主角色或备用角色。

例如，请考虑以下三个JobManager实例的设置：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201120143929939.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70#pic_center)
## 配置
要启用JobManager高可用性，您必须将高可用性模式设置为zookeeper，配置ZooKeeper quorum并修改conf/master文件用于记录所有的所有JobManager的hosts和web UI端口。
Flink利用ZooKeeper在所有正在运行的JobManager实例之间进行分布式协调。 ZooKeeper是独立于Flink的一项服务，该服务通过组长选举和轻量级一致状态存储提供高度可靠的分布式协调。
### 配置一、Masters File (masters)
In order to start an HA-cluster configure the masters file in conf/masters:
master file在flink安装包的conf目录下，master文件中配置所有JobManager的host地址和webui端口
```xml
jobManagerAddress1:webUIPort1
[...]
jobManagerAddressX:webUIPortX
```
JobManager默认会选取一个随机的端口用于进程间的通信，你可以通过修改`high-availability.jobmanager.port`来改变。这个key可以是单个端口(e.g. 50010)，也可以是一个范围(50000-50025)，或者是两者组合(50010,50011,50020-50025,50050-50075)。
#### 配置二、flink-conf.yaml
In order to start an HA-cluster add the following configuration keys to conf/flink-conf.yaml:
为了启动HA-cluster，需要在conf/flink-conf.yaml中做如下配置：

- high-availability mode (必要的): high-availability mode需要设置为zookeeper
>high-availability: zookeeper
- ZooKeeper quorum (必要的): ZooKeeper quorum是一组ZooKeeper server，其可提供分布式协调服务
>high-availability.zookeeper.quorum: address1:2181[,...],addressX:2181

每个addressX:port 都指向一个ZooKeeper server, 其可被flink进程访问。

- ZooKeeper root (推荐配置): ZooKeeper的根结点，在这个节点下存放所有的cluster node。
>high-availability.zookeeper.path.root: /flink

- ZooKeeper cluster-id (推荐配置): The cluster-id ZooKeeper node, under which all required coordination data for a cluster is placed.
> high-availability.cluster-id: /default_ns # important: customize per cluster

**Important**: You should not set this value manually when running a YARN cluster, a per-job YARN session, or on another cluster manager. In those cases a cluster-id is automatically being generated based on the application id. Manually setting a cluster-id overrides this behaviour in YARN. Specifying a cluster-id with the -z CLI option, in turn, overrides manual configuration. If you are running multiple Flink HA clusters on bare metal, you have to manually configure separate cluster-ids for each cluster.

- Storage directory (必要的): JobManager元数据将保留在文件系统storageDir中，并且只有一个指向此状态的指针存储在ZooKeeper中。

>high-availability.storageDir: hdfs:///flink/recovery
  
   storageDir存储恢复JobManager故障所需的所有元数据。

配置了masters和ZooKeeper quorum后，您可以照常使用提供的集群启动脚本。 他们将启动HA群集。 请记住，调用脚本时必须运行ZooKeeper quorum，并确保为要启动的每个HA集群配置单独的ZooKeeper根路径。

### Example: Standalone Cluster with 2 JobManagers
1、conf/flink-conf.yaml中配置 high availability mode and ZooKeeper quorum:
```yaml
high-availability: zookeeper
high-availability.zookeeper.quorum: localhost:2181
high-availability.zookeeper.path.root: /flink
high-availability.cluster-id: /cluster_one # important: customize per cluster
high-availability.storageDir: hdfs:///flink/recovery
```
2、conf/masters中配置所有JobManager:
```yaml
localhost:8081
localhost:8082
```
3、conf/zoo.cfg中配置ZooKeeper server:
```yaml
server.0=localhost:2888:3888
```
4、启动 ZooKeeper quorum:
```shell
$ bin/start-zookeeper-quorum.sh
Starting zookeeper daemon on host localhost.
```
5、启动HA-cluster:
```shell
$ bin/start-cluster.sh
Starting HA cluster with 2 masters and 1 peers in ZooKeeper quorum.
Starting standalonesession daemon on host localhost.
Starting standalonesession daemon on host localhost.
Starting taskexecutor daemon on host localhost.
```
6、Stop ZooKeeper quorum and cluster:
```shell
$ bin/stop-cluster.sh
Stopping taskexecutor daemon (pid: 7647) on localhost.
Stopping standalonesession daemon (pid: 7495) on host localhost.
Stopping standalonesession daemon (pid: 7349) on host localhost.
$ bin/stop-zookeeper-quorum.sh
Stopping zookeeper daemon (pid: 7101) on host localhost.
```
# yarn模式下的高可用
在运行高度可用的YARN群集时，我们不必运行多个JobManager（ApplicationMaster）实例，当实例出现故障时，YARN会重新启动该实例。 具体操作取决于您使用的YARN版本。
## 配置
### Maximum Application Master Attempts (yarn-site.xml)
You have to configure the maximum number of attempts for the application masters for your YARN setup in yarn-site.xml:
```xml
<property>
  <name>yarn.resourcemanager.am.max-attempts</name>
  <value>4</value>
  <description>
    The maximum number of application master execution attempts.
  </description>
</property>
```
The default for current YARN versions is 2 (meaning a single JobManager failure is tolerated).

### Application Attempts (flink-conf.yaml)
In addition to the HA configuration (see above), you have to configure the maximum attempts in conf/flink-conf.yaml:
```xml
yarn.application-attempts: 10
```
This means that the application can be restarted 9 times for failed attempts before YARN fails the application (9 retries + 1 initial attempt). Additional restarts can be performed by YARN if required by YARN operations: Preemption, node hardware failures or reboots, or NodeManager resyncs. These restarts are not counted against yarn.application-attempts, see Jian Fang’s blog post. It’s important to note that yarn.resourcemanager.am.max-attempts is an upper bound for the application restarts. Therefore, the number of application attempts set within Flink cannot exceed the YARN cluster setting with which YARN was started.
## Example: Highly Available YARN Session
1、Configure HA mode and ZooKeeper quorum in conf/flink-conf.yaml:
```yaml
high-availability: zookeeper
high-availability.zookeeper.quorum: localhost:2181
high-availability.storageDir: hdfs:///flink/recovery
high-availability.zookeeper.path.root: /flink
yarn.application-attempts: 10
2、Configure ZooKeeper server in conf/zoo.cfg (currently it’s only possible to run a single ZooKeeper server per machine):
```yaml
server.0=localhost:2888:3888
```
3、Start ZooKeeper quorum:
```shell
$ bin/start-zookeeper-quorum.sh
Starting zookeeper daemon on host localhost.
```
4、Start an HA-cluster:
```shell
$ bin/yarn-session.sh -n 2
```
