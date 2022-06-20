>1.将字符串的时间转换为时间戳

方法:
        a = "2013-10-10 23:40:00"
        将其转换为时间数组
        import time
        timeArray = time.strptime(a, "%Y-%m-%d %H:%M:%S")
    转换为时间戳:
    timeStamp = int(time.mktime(timeArray))
    timeStamp == 1381419600

    
>2.字符串格式更改

    如a = "2013-10-10 23:40:00",想改为 a = "2013/10/10 23:40:00"
    方法:先转换为时间数组,然后转换为其他格式
    timeArray = time.strptime(a, "%Y-%m-%d %H:%M:%S")
    otherStyleTime = time.strftime("%Y/%m/%d %H:%M:%S", timeArray)
>3.时间戳转换为指定格式日期

import datetime
        timeStamp = 1381419600
        dateArray = datetime.datetime.utcfromtimestamp(timeStamp)
        otherStyleTime = dateArray.strftime("%Y-%m-%d %H:%M:%S")
        otherStyletime == "2013-10-10 23:40:00"
