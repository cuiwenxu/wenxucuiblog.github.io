制作拉链表step
以订单表为例
- 1、 拉取线上mysql订单表以初始化拉链表
- 2、 以后每天仅需要拉取当天创建或者当天更新的数据，放入增量表中
- 3、 拉链表的更新分为两部分
         part1 增量表中的记录（历史从未出现，即create_time=当天 & 历史出现过，当天更新）
         part2 拉链表left join增量表，获取历史出现当天更新的数据，将历史数据关链
```sql
insert overwrite order_chain
select * 
from
(
--part1 历史出现，当天更新的数据
select chain.order_id,chain.order_status,chain.begin_date,
if(incr.order_id is null,chain.end_date,date_sub(incr.update_time,1)) as end_date
from 
(
select order_id,order_status,begin_date,end_date
from order_chain --拉链表
) chain
left join
(
select order_id,order_status,update_time
from order_d --增量表
where update_time=${date}
) incr
on chain.order_id=incr.order_id

union all

--part2 历史未出现，当天新增或历史出现，当天更改的数据
select order_id,order_status,${date} as begin_date,'9999-12-31' as end_date
from order_d --增量表
where create_time=${date} or update_time=${date}
) a 
```


