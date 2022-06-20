@[TOC](mysql忘记密码-重置密码)
# 1、使用find命令找到mysqld.cnf（或者my.cnf，不同版本mysql该配置文件名称不同），在[mysqld]那一段加上skip-grant-tables
> skip-grant-tables
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210108112131184.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)

# 2、重启mysql服务
```shell
不同版本命令不一样，下面两条可试着用
service mysql restart

#新的mysql执行这个命令
systemctl restart mysqld.service 
```
# 3、登录mysql重置密码
```sql
#连接mysql，直接回车即可，不需要输入密码
mysql -u root -p

#更新root用户密码
update mysql.user set authentication_string=password('') where user='root' and Host = 'localhost';

#刷新权限
flush privileges;

#推出mysql
exit   或者 quit
```
# 4、删除配置文件中的skip-grant-tables
# 5、重启mysql服务
