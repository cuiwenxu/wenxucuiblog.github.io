executeBatch方法会返回一个int数组
```java
int[] executeBatch() throws SQLException;
```

数组各值可能是以下之一：
- 大于或等于零的数字,表示命令已成功处理，并且是更新计数，给出了
      数据库中受命令影响的行数执行
- `SUCCESS_NO_INFO` ( -2)的值,表示命令为处理成功，但受影响的行数为未知
     
如果批量更新中的命令之一无法正确执行，此方法引发`BatchUpdateException`，JDBC driver可能会也可能不会继续处理剩余的命令。但是driver的行为是与特定的DBMS绑定的，要么总是继续处理命令，要么从不继续处理命令。如果驱动程序继续处理，方法将返回 `EXECUTE_FAILED`(-3)。
     
     
