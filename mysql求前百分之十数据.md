mysql求前百分之十数据借助用户自定义变量来实现
##### 用户自定义变量
我们可以定义自己的变量，变量名为 @var_name，用户自定义变量都是会话级的

>SET @var_name = expr [, @var_name = expr] …
For SET, either = or := can be used as the assignment operator.

我们也可以使用select来使用变量，或者`给变量赋值，但必须使用:=`
>mysql> SET @t1=1, @t2=2, @t3:=4;
Query OK, 0 rows affected
mysql> SELECT @t1, @t2, @t3, @t4 := @t1+@t2+@t3;
+-----+-----+-----+--------------------+
| @t1 | @t2 | @t3 | @t4 := @t1+@t2+@t3 |
+-----+-----+-----+--------------------+
|   1 |   2 |   4 |                  7 |
+-----+-----+-----+--------------------+
1 row in set

**自定义变量，是在结果返回到客户端时，才进行处理的，所以我们HAVING, GROUP BY, or ORDER BY中使用的时候，并没有效果。**
有了自定义变量，求求前百分之几数据就简单很多，代码如下
```sql
set @row_num:=0;
select *
from
(
select *,@row_num:=@row_num+1 as rn
from common_date_selector
order by id
) a 
where rn<@row_num*0.1
```
