# 常见代码疏漏
对于mysql

1.将 WHERE `some_field` = '${变量}' 替换为 WHERE `some_field` = #{param}

2.将 like '%${变量}%' 替换为 like concat('%', #{变量}, '%')

3.将 like concat('%', ${变量}, '%') 替换成 like concat('%', #{变量}, '%')

4. 将 WHERE `some_field` IN (${变量}) 替换为


```sql
WHERE `some_field` IN
<foreach item="item" index="index" collection="list" open="(" separator="," close=")">
#{item}
</foreach>
```

对于oracle:
将 like '%${变量}%' 替换为 LIKE '%'||#{变量}||'%'

# 附
## SQL中# 与$ 的区别
区别：
（1）#将传入的数据都当成一个字符串，会对自动传入的数据加一个双引号。如：order by #user_id#，如果传入的值是id，则解析成的sql为order by "id"。
（2）$将传入的数据直接显示生成在sql中。如：order by $user_id$，如果传入的值是id，则解析成的sql为order by id。
（3）#方式在很大程度上能够防止sql注入。
（4）$方式无法防止sql注入。
（5）$方式一般用于传入数据库对象，例如传入表名。（这里得注意SQL注入问题）
（6）一般能用#的就别用$。

ps:在使用mybatis中还遇到<![CDATA[]]>的用法，在该符号内的语句，将不会被当成字符串来处理，而是直接当成sql语句，比如要执行一个存储过程。

>总结区别：#{} 传入值时，sql解析时，参数是带引号的，而${}穿入值，sql解析时，参数是不带引号的。

 
举个例子：

`select * from ${table_Name} where name = #{name}`

在这个例子中，如果表名为

`user; delete user; --`

　　则动态解析之后 sql 如下：

`select * from user; delete user; -- where name = ?;`

--之后的语句被注释掉，而原本查询用户的语句变成了查询所有用户信息+删除用户表的语句，会对数据库造成致命损伤。
但是表名用参数传递进来的时候，只能使用 ${} 。这也提醒在这种用法中要小心sql注入的问题。

 

防止SQL注入方法：

首先，永远不要相信用户的输入。

（1）不使用SQL，考虑NoSQL。

（2）正则表达式，字符串过滤。

（3）参数绑定PreparedStatement。

（4）使用正则表达式过滤传入的参数。

（5）JSP中调用该函数检查是否包函非法字符或JSP页面判断代码。JSP参考JSP使用过滤器防止SQL注入

 


 



