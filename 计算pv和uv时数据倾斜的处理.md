#### 背景
>计算pv和uv时经常出现数据倾斜，比如在场门客流统计时，地铁口场门是其他场门的上百倍。这很容易导致数据倾斜

##### 解决方案
`整体思路是把数据打撒，做两阶段聚合`。但是在计算pv和uv时，还是略有不同。
- 计算pv时，在group by后简单添加随机数即可，代码如下:
```sql
select dt,
       gate_id,
       sum(pv) as pv
from(
  select
        dt,
        gate_id,
        count(pid) as pv
  from table
        group by
        dt,
        gate_id,
        floor(rand()*100) --随机打散成100份 
    )
    group by 
    dt,
    gate_id;
```
- 计算uv时，因为要去重，`所以group by的时候不能简单地用随机数，而是要 group by mod(pid)(适用于pid为数字的情况)，或者group by concat(gate_id,'$$$',pid)`，代码如下：
```sql
select dt,
       gate_id,
       sum(uv) as uv
from(
  select
        dt,
        gate_id,
        count(distinct pid) as uv
  from table
        group by
        dt,
        gate_id,
        concat(gate_id,'$$$',pid)
    )
    group by 
    dt,
    gate_id;
```
