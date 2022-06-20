>问题背景：
有些jar包虽然在maven仓库可以找到，
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210119213542795.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
但是使用idea的maven插件下载不到（可能要切maven源）![在这里插入图片描述](https://img-blog.csdnimg.cn/20210119213607554.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)

>解决方法
此时可以通过网页把jar包下载下来，然后添加到本地maven仓库中，命令如下：
mvn install:install-file -Dfile=/Users/xxx/Desktop/hive-exec-2.1.1-cdh6.0.1.jar -DgroupId=org.apache.hive -DartifactId=hive-metastore -Dversion=2.1.1-cdh6.0.1 -Dpackaging=jar
