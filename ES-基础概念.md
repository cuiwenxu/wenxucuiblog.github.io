##### 用数据库做搜索会怎么样
1、数据库搜索某个关键字会全文遍历，性能低
2、数据库搜索没办法将关键词拆开，比如搜索生化机就搜索不到生化危机

##### ES基本概念和数据库类比

ES     | 数据库
-------- | -----
document  | 行
type | 表
index  | 库
##### ES常用命令
###### GET _cat/health?v 

1、用途
此命令常见的用途一般有两个:
1、验证节点之间的健康状况是否一致，
2、跟踪大型集群随时间的故障恢复情况

2、结果解析
正常情况下，执行GET _cat/health?v命令得到的结果如下：
```java
epoch      timestamp cluster            status node.total node.data shards  pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1602118776 00:59:36  kubernetes-logging green          15        10   3452 1726    0    0        0             0                  -                100.0%
```
含义如下：
前两个是时间戳，不过多介绍。其余如下：
1、cluster ，集群名称
2、status，集群状态 green代表健康；yellow代表分配了所有主分片，但至少缺少一个副本，此时集群数据仍旧完整；red代表部分主分片不可用，可能已经丢失数据。
3、node.total，代表在线的节点总数量
4、node.data，代表在线的数据节点的数量
5、shards， active_shards 存活的分片数量
pri，active_primary_shards 存活的主分片数量 正常情况下 shards的数量是pri的两倍。
6、relo， relocating_shards 迁移中的分片数量，正常情况为 0
7、init， initializing_shards 初始化中的分片数量 正常情况为 0
8、unassign， unassigned_shards 未分配的分片 正常情况为 0
9、pending_tasks，准备中的任务，任务指迁移分片等 正常情况为 0
10、max_task_wait_time，任务最长等待时间
11、active_shards_percent，正常分片百分比 正常情况为 100%


