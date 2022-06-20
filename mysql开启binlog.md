mysql版本5.7.24
执行`show variables like '%log_bin%';`发现没有开启binlog，于是执行`sudo find / -name my.cnf`或者`sudo find / -name mysql.cnf`来查找mysql的配置文件。my.cnf中引入了两个目录，这两个目录下的.cnf后缀文件都会成为配置文件
```
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/
```
选择其中之一在`/etc/mysql/conf.d/`目录下的mysql.cnf文件，在其中添加
```
[mysqld]
log-bin=mysql-bin
server-id=1
```
重启mysql服务`sudo /etc/init.d/mysql restart`，再次执行执行`show variables like '%log_bin%';`可发现binlog已经成功开启
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191218210427309.png)

