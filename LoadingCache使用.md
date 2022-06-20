# guava LoadingCache介绍
Guava Cache : LoadingCache是一个本地缓存。
## 优点
- 线程安全的缓存，与ConcurrentMap相似，但前者增加了更多的元素失效策略，后者只能显示的移除元素。
- 提供了三种基本的缓存回收方式：基于容量回收、定时回收和基于引用回收。定时回收有两种：按照写入时间，最早写入的最先回收；按照访问时间，最早访问的最早回收。
监控缓存加载/命中情况。
- 集成了多部操作，调用get方式，可以在未命中缓存的时候，从其他地方获取数据源（DB，redis），并加载到缓存中。

# LoadingCache的工作原理
工作流程图
![在这里插入图片描述](https://img-blog.csdnimg.cn/ad9110cedf2f470ca68f4b6a3e29b030.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5aSW5Y2W6aqR5omL5bCP5bSU,size_20,color_FFFFFF,t_70,g_se,x_16)
# demo
```java
package task;

import com.google.common.cache.*;

import java.util.concurrent.ExecutionException;
import java.util.concurrent.TimeUnit;

/**
 * @Author: c
 * @Date: 2021/11/25 8:56 下午
 */
public class LoadingCacheTest {

    public static void main(String[] args) throws InterruptedException, ExecutionException {

        LoadingCache<Integer,String> strCache
                //CacheBuilder的构造函数是私有的，只能通过其静态方法newBuilder()来获得CacheBuilder的实例
                = CacheBuilder.newBuilder()
                //设置并发级别为8，并发级别是指可以同时写缓存的线程数
                .concurrencyLevel(8)
                //设置写缓存后8秒钟过期
                .expireAfterWrite(8, TimeUnit.SECONDS)
                //设置缓存容器的初始容量为10
                .initialCapacity(10)
                //设置缓存最大容量为100，超过100之后就会按照LRU最近虽少使用算法来移除缓存项
                .maximumSize(100)
                //设置要统计缓存的命中率
                .recordStats()
                //设置缓存的移除通知
                .removalListener(new RemovalListener<Object, Object>() {
                    @Override
                    public void onRemoval(RemovalNotification<Object, Object> notification) {
                        System.out.println(notification.getKey() + " was removed, cause is " + notification.getCause());
                    }
                })
                //build方法中可以指定CacheLoader，在缓存不存在时通过CacheLoader的实现自动加载缓存
                .build(
                        new CacheLoader<Integer, String>() {
                            @Override
                            public String load(Integer key) throws Exception {
                                System.out.println("load key " + key);
                                return String.valueOf(key)+"input";
                            }
                        }
                );

        for (int i=0;i<20;i++) {
            //从缓存中得到数据，由于我们没有设置过缓存，所以需要通过CacheLoader加载缓存数据
            String value = strCache.get(i);
            System.out.println("get==="+value);
            //休眠1秒
            TimeUnit.SECONDS.sleep(1);
        }
        for (int i=0;i<20;i++) {
            String value = strCache.get(i);
            System.out.println("second get==="+value);
        }

        System.out.println("cache stats:");
        //最后打印缓存的命中率等 情况
        System.out.println(strCache.stats().toString());
    }


}

```
# QA
- Q1、LoadingCache在哪块代码调用了二级存储（远程存储）？以及在哪里进行了缓存填充
>Ans：在CacheLoader类的load方法中，如果调用get方法获取不到元素，则会自动调用CacheLoader.load方法，在load方法中访问远程存储，如果远程存储有该key，则将远程存储的<key,value>填充到本地缓存中，如上面流程图所示。

# 附代码
https://github.com/cuiwenxu/cachepro
