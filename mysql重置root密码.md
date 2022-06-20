1、打开配置文件/etc/mysql/conf.d/，在mysqld下添加一行skip-grant-tables，如下：
```
[mysqld]
#...
skip-grant-tables
#...
```
这样我们就可以免密登录MySQL了。
然后保存并退出。
2、重启MySQL
重启mysql服务sudo /etc/init.d/mysql restart
3、终端输入 mysql 直接登录MySQL数据库：
```
$ mysql
```
成功进入mysql

切换到MySQL系统库mysql：
```
mysql> use mysql;
```
5、重置root密码

需要注意的是，在MySQL5.7之后，已经没有password这个字段了，

password字段改成了authentication_string。

修改密码我们要修改这个字段的值。
```
update user set authentication_string=password('新密码') where user='root';
```
这样，我们就已经修改密码成功了。

5、修改 /etc/my.cnf 文件，将之前添加的skip-grant-tables 这句话注释掉。
不然我们仍然还是免密的方式登录Mysql。
6、再次重启MySQL就大功告成了。
