maven部署jar包到私服主要包括两步
1、在要部署的jar包的pom文件中，添加<distributionManagement>节点

```bash
<distributionManagement>
        <repository>
            <id>releases</id>
            <name>nexus releases</name>
            <url>http://nexus.xxx.cn:8090/repository/dataplatform-bi-release/</url>
        </repository>
        <snapshotRepository>
            <id>snapshots</id>
            <name>nexus snapshots</name>
            <url>http://nexus.xxx.cn:8090/repository/dataplatform-bi-snapshot/</url>
        </snapshotRepository>
    </distributionManagement>
```
其中url就是要上传私服的url

2、使用mvn命令行上传jar包
```bash
mvn deploy:deploy-file -DgroupId=com..di -DartifactId=OrmUtils -Dversion=1.0-SNAPSHOT -Dpackaging=jar -Dfile=/Users/cuiwenxu/Documents/gitcode/OrmUtils/target/OrmUtils-1.0-SNAPSHOT.jar -Durl=http://nexus.xxx.cn:8090/repository/dataplatform-bi-snapshot/ -DrepositoryId=snapshots
```
除了使用命令行外，还可以用idea的可视化maven插件来上传，点击deploy即可
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200105120538529.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)

