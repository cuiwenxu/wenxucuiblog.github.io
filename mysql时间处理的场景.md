>### timestamp转2020-01-06格式
>from_unixtime(timestamp/1000,"%Y-%m-%d")  
> <font color="red">需要注意的是timestamp如果是毫秒需要/1000转换成秒</font>
```sql
select from_unixtime(timestamp/1000,"%Y-%m-%d"),bi_fake_data,count(1)
from gate_event
where from_unixtime(timestamp/1000,"%Y-%m-%d")>'2019-12-17'
group by from_unixtime(timestamp/1000,"%Y-%m-%d"),bi_fake_data
```
>### timestamp转2020-01-06 10:30:30格式
>from_unixtime(timestamp/1000,"%Y-%m-%d %H-%I-%S") 
><font color="red">通过hour(from_unixtime(timestamp/1000,"%Y-%m-%d %H-%I-%S"))可获取timestamp对应的小时</font>
```sql
select from_unixtime(timestamp/1000,"%Y-%m-%d %H:%I:%S") as dt_str,bi_fake_data,count(1)
from gate_event
where from_unixtime(timestamp/1000,"%Y-%m-%d")>'2019-12-17'
group by from_unixtime(timestamp/1000,"%Y-%m-%d %H:%I:%S"),bi_fake_data
```
MySQL日期格式化（format）取值范围
```sql
值   含义
秒   %S、%s   两位数字形式的秒（ 00,01, ..., 59）
分   %I、%i   两位数字形式的分（ 00,01, ..., 59）
小时  %H  24小时制，两位数形式小时（00,01, ...,23）
%h  12小时制，两位数形式小时（00,01, ...,12）
%k  24小时制，数形式小时（0,1, ...,23）
%l  12小时制，数形式小时（0,1, ...,12）
%T  24小时制，时间形式（HH:mm:ss）
%r   12小时制，时间形式（hh:mm:ss AM 或 PM）
%p  AM上午或PM下午
  周      %W 一周中每一天的名称（Sunday,Monday, ...,Saturday）
 %a 一周中每一天名称的缩写（Sun,Mon, ...,Sat）
%w  以数字形式标识周（0=Sunday,1=Monday, ...,6=Saturday）
%U  数字表示周数，星期天为周中第一天
%u  数字表示周数，星期一为周中第一天
天   %d  两位数字表示月中天数（01,02, ...,31）
%e   数字表示月中天数（1,2, ...,31）
 %D 英文后缀表示月中天数（1st,2nd,3rd ...）
 %j 以三位数字表示年中天数（001,002, ...,366）
月   %M  英文月名（January,February, ...,December）
%b  英文缩写月名（Jan,Feb, ...,Dec）
%m  两位数字表示月份（01,02, ...,12）
%c  数字表示月份（1,2, ...,12）
年   %Y  四位数字表示的年份（2015,2016...）
%y   两位数字表示的年份（15,16...）
文字输出    %文字     直接输出文字内容
```
