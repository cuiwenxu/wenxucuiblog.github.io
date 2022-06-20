>日期格式字符串转换成时间戳
```java
/** 
* 日期格式字符串转换成时间戳 
* @param date 字符串日期 
* @param format 如：yyyy-MM-dd HH:mm:ss 
* @return 
*/  
public static String date2TimeStamp(String date_str,String format){  
    try {  
        SimpleDateFormat sdf = new SimpleDateFormat(format);  
        return String.valueOf(sdf.parse(date_str).getTime()/1000);  
    } catch (Exception e) {  
        e.printStackTrace();  
    }  
    return "";  
}   
```
> 时间戳转字符串
```java
/** 
* 时间戳转换成日期格式字符串
* @param millis 毫秒 
* @param format 如：yyyy-MM-dd HH:mm:ss 
* @return 
*/  
public static String timestamp2Date(String millis,String format){  
    try {  
SimpleDateFormat sdf = new SimpleDateFormat(format);  
return sdf.format(new Date(Long.valueOf(seconds+"000"))); 
 } catch (Exception e) {  
        e.printStackTrace();  
    }  
    return "";  
}   
```
