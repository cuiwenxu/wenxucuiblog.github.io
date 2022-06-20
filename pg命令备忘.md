search_path类似于linux中的path环境变量。
postgres=# show search_path;

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201118135709528.png#pic_center)

默认值是$user,public,意思就是当以某个用户登录到数据库的时候,默认就是先查找和登录用户同名的schema,再查找public。

修改search_path：

postgres=# set search_path = schema_name1,schema_name2,...;

或者

postgres=# set search_path to schema_name1,schema_name2,...;
恢复：

postgres=# set search_path = "$user", public;
