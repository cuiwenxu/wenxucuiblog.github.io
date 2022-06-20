>开发中经常遇到有一个类在已有项目中存在，在新项目中也想用这个类，那么怎么查找该类对应jar包的maven坐标呢？

比如要找org.apache.http.message.TokenParser的maven坐标
- 1、定位jar包路径
![在这里插入图片描述](https://img-blog.csdnimg.cn/14978c43b8464d408a2b83fa0fc5f480.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAY3Vpd2VueHUx,size_20,color_FFFFFF,t_70,g_se,x_16)
- 2、在jar包meta-inf中找maven坐标
`需要注意pom.properties中的才是正确坐标，不要误拷贝pom.xml中的parent module坐标，因为pom.properties发布的坐标，不一定和pom.xml中的parent module坐标保持一致`
![在这里插入图片描述](https://img-blog.csdnimg.cn/8b51b365025e4be18fa1e5997fb9a202.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBAY3Vpd2VueHUx,size_20,color_FFFFFF,t_70,g_se,x_16)

