timestamp转2020-04-13 00:04:19格式
>select from_unixtime(cast(`timestamp`/1000 as bigint))

timestamp获取hour
>select hour(from_unixtime(cast(`timestamp`/1000 as bigint)))

2020-04-13 00:04:19格式转时间戳
>select unix_timestamp('2011-12-07 13:01:03','yyyy-MM-dd HH:mm:ss')

