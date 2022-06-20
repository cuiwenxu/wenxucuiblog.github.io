maven换源需要修改settings.xml，首先用`mvn -version`查找出maven的安装位置
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191211144649943.png)然后cd到maven目录下的conf文件夹下，打开`settings.xml`文件，修改`<mirrors>`节点，添加镜像（常用的如阿里的镜像）

```
<!--阿里maven镜像-->
<mirror>
  <id>aliyun</id>
  <mirrorOf>central</mirrorOf>
  <name>aliyun maven</name>
  <url>http://maven.aliyun.com/nexus/content/groups/public</url>
</mirror>
```

