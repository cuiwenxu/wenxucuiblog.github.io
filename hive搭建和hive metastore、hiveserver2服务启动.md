hive搭建参见 [hive搭建](https://dblab.xmu.edu.cn/blog/2440-2/).
hive metastore服务启动
> bin/hive --service metastore &

启动hiveserver2
> bin/hive --service hiveserver2 &

需要注意的是想要hiveserver2服务可用，需要启动metastore可用
