@[TOC](viewfs指南)
# 介绍
View File System (ViewFs) 提供了一种管理多个 Hadoop 文件系统namespace（或命名空间卷）的方法。 它对于在 HDFS Federation中具有多个名称节点并因此具有多个命名空间的集群特别有用。 ViewFs 类似于某些 Unix/Linux 系统中的客户端挂载表。 ViewFs 可用于创建个性化的命名空间视图以及每个集群的公共视图。

本指南是在具有多个集群的 Hadoop 系统的使用场景中介绍的，每个集群可以由多个命名空间组成。 它还描述了如何在 HDFS Federation中使用 ViewFs 来提供每个集群的全局命名空间，以便应用程序可以以类似于操作单节点集群的方式操作联邦HDFS。

# 旧世界（联邦前）
## 单个 Namenode 集群
在 HDFS 联邦之前的旧世界中，集群有一个namenode，为该集群提供单个文件系统命名空间。假设有多个集群。每个集群的文件系统命名空间是完全独立且不相交的。此外，物理存储不跨集群共享（即数据节点不跨集群共享。）

每个集群的 core-site.xml 都有一个配置属性，用于将默认文件系统设置为该集群的 namenode：
```xml
<property>
  <name>fs.default.name</name>
  <value>hdfs://namenodeOfClusterX:port</value>
</property>
```
这样的配置属性允许使用斜杠相对名称来解析相对于集群名称节点的路径。例如，路径 /foo/bar 是指使用上述配置的 hdfs://namenodeOfClusterX:port/foo/bar。

此配置属性设置在集群上的每个网关上以及该集群的关键服务上，例如 JobTracker 和 Oozie。

## 路径名的使用模式
因此，在上面设置core-site.xml的Cluster X上，典型的路径名是
example 1:
> /foo/bar

这相当于hdfs://namenodeOfClusterX:port/foo/bar。

example2:
>hdfs: //namenodeOfClusterX:端口/foo/bar

虽然这是一个有效的路径名，但最好使用/foo/bar，因为它允许应用程序及其数据在需要时透明地移动到另一个集群。

example3:
>hdfs://namenodeOfClusterY:端口/foo/bar

它是一个URI，用于引用另一个集群(比如集群Y)上的路径名。特别是，将文件从集群Y复制到集群Z的命令如下所示:
>distcp hdfs: //namenodeClusterY:port/pathSrc hdfs://namenodeClusterZ:port/pathDest
webhdfs: //namenodeClusterX:http_port/foo/bar

它是一个通过WebHDFS文件系统访问文件的URI。注意WebHDFS使用namenode的HTTP端口，而不是RPC端口。
> http://namenodeClusterX: http_port/webhdfs/v1/foo/bar和http://proxyClusterX: http_port/foo/ bar

这些分别是通过WebHDFS REST API和HDFS代理访问文件的HTTP url。
## 路径名使用最佳实践
当一个URI位于集群中时，建议使用上面类型(1)的路径名，而不是像(2)那样的完全限定URI。完全限定URI类似于地址，不允许应用程序随其数据移动。

# 新世界-Federation and ViewFs
## 集群的新形态
假设有多个集群。每个集群都有一个或多个namenode。每个namenode都有自己的名称空间。一个namenode只能属于一个集群。同一个集群中的namenode共享该集群的物理存储。跨集群的名称空间和以前一样是独立的。

根据所要操作的目录决定具体存放在哪个namenode下。例如，他们可能会把所有的用户数据(/user/<username>)放在一个namenode中，所有的feed数据(/data)放在另一个namenode中，所有的项目(/projects)放在另一个namenode中，等等。

## 使用ViewFs提供全局命名空间
为了提供与旧世界相同的透明性，使用ViewFs文件系统(即客户端挂载表)为每个集群创建一个独立的集群名称空间视图，这与旧世界中的名称空间类似。客户端挂载表(如Unix挂载表)，它们使用旧的命名约定挂载新的名称空间卷。下图显示了一个挂载四个命名空间卷/user、/data、/projects和/tmp的挂载表:
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021062211355065.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
ViewFs implements the Hadoop file system interface just like HDFS and the local file system. It is a trivial file system in the sense that it only allows linking to other file systems. Because ViewFs implements the Hadoop file system interface, it works transparently Hadoop tools. For example, all the shell commands work with ViewFs as with HDFS and local file system.

In the configuration of each cluster, the default file system is set to the mount table for that cluster as shown below (compare it with the configuration in Single Namenode Clusters).

<property>
  <name>fs.defaultFS</name>
  <value>viewfs://clusterX</value>
</property>
The authority following the viewfs:// scheme in the URI is the mount table name. It is recommended that the mount table of a cluster should be named by the cluster name. Then Hadoop system will look for a mount table with the name “clusterX” in the Hadoop configuration files. Operations arrange all gateways and service machines to contain the mount tables for ALL clusters such that, for each cluster, the default file system is set to the ViewFs mount table for that cluster as described above.

The mount points of a mount table are specified in the standard Hadoop configuration files. All the mount table config entries for viewfs are prefixed by fs.viewfs.mounttable.. The mount points that are linking other filesystems are specified using link tags. The recommendation is to have mount points name same as in the linked filesystem target locations. For all namespaces that are not configured in the mount table, we can have them fallback to a default filesystem via linkFallback.

In the below mount table configuration, namespace /data is linked to the filesystem hdfs://nn1-clusterx.example.com:8020/data, /project is linked to the filesystem hdfs://nn2-clusterx.example.com:8020/project. All namespaces that are not configured in the mount table, like /logs are linked to the filesystem hdfs://nn5-clusterx.example.com:8020/home.

ViewFs实现Hadoop文件系统接口，就像HDFS和本地文件系统一样。它是一个虚拟的文件系统，因为它只允许链接到其他文件系统。因为ViewFs实现了Hadoop文件系统接口，所以它透明地工作于Hadoop工具。所有HDFS shell命令都可用于ViewFs。

在每个集群的配置中，默认文件系统被设置为该集群的挂载表，如下所示(与Single Namenode Clusters中的配置进行比较)。
```xml
<property>
  <name>fs.defaultFS</name>
  <value>viewfs://clusterX</value>
</property>
```
URI中viewfs:// scheme后面的权威是挂载表名。建议集群的挂载表以集群名称命名。然后Hadoop系统将在Hadoop配置文件中查找名为“clusterX”的挂载表。操作安排所有网关和服务机器包含所有集群的挂载表，这样，对于每个集群，默认文件系统被设置为上述集群的ViewFs挂载表。

挂载表的挂载点在标准Hadoop配置文件中指定。所有viewfs的挂载表配置项都以fs.viewfs.mounttable为前缀。链接其他文件系统的挂载点是使用链接标记指定的。建议使用与链接文件系统目标位置相同的挂载点名称。对于挂载表中没有配置的所有名称空间，我们可以通过linkFallback让它们回退到默认文件系统。

在下面的挂载表配置中，命名空间/data链接到文件系统hdfs://nn1-clusterx.example.com:8020/data， /project链接到文件系统hdfs://nn2-clusterx.example.com:8020/project。挂载表中没有配置的所有命名空间，比如/logs，都被链接到文件系统hdfs://nn5-clusterx.example.com:8020/home。
