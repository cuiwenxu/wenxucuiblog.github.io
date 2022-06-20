org.apache.flink.metrics.slf4j.Slf4jReporter是flink提供的reporter的一种，要想使用Slf4jReporter，需要：
1、在flink-conf.yaml中添加repoter配置
>metrics.reporter.slf4j.class: org.apache.flink.metrics.slf4j.Slf4jReporter
metrics.reporter.slf4j.interval: 60 SECONDS

2、在客户端的lib目录下添加
