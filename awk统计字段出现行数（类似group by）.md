使用`hdfs dfs -cat /prod/brand=xxx/*|awk -F "," '{print $4}'`

可查看逗号分隔的第四个字段，如果想要统计该字段各值出现的次数，可以使用以下命令

>hdfs dfs -cat /prod/brand=xxx/*|awk -F "," '{count[$4]++;} END {for(i in count) {print i count[i]}}'

```shell
命令可分解为两部分


1、{count[$4]++;} END #声明数组（数组名字可随便起，叫arr、list等也行），数组用来存放$4,每存放一个值就+=1
2、{for(i in count) {print i count[i]}} #遍历数组，输出累加结果（结果为每个值出现的行数，类似group by count(1)）

```

