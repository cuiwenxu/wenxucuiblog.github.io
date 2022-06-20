# 双重检查创建单例对象的错误写法
在Java领域一个经典的案例就是利用双重检查创建单例对象，例如下面的代码：在获取实例getInstance()的方法中，我们首先判断instance是否为空，如果为空，则锁定Singleton.class并再次检查instance是否为空，如果还为空则创建Singleton的一个实例。
```java
public class Singleton {
  static Singleton instance;
  static Singleton getInstance(){
    if (instance == null) {
      synchronized(Singleton.class) {
        if (instance == null)
          instance = new Singleton();
        }
    }
    return instance;
  }
}
```
假设有两个线程A、B同时调用getInstance()方法，他们会同时发现 instance = = null ，于是同时对Singleton.class加锁，此时JVM保证只有一个线程能够加锁成功（假设是线程A），另外一个线程则会处于等待状态（假设是线程B）；线程A会创建一个Singleton实例，之后释放锁，锁释放后，线程B被唤醒，线程B再次尝试加锁，此时是可以加锁成功的，加锁成功后，线程B检查 instance == null 时会发现，已经创建过Singleton实例了，所以线程B不会再创建一个Singleton实例。

**<font color="red">这看上去一切都很完美，无懈可击，但实际上这个getInstance()方法并不完美。问题出在哪里呢？出在new操作上**，我们以为的new操作应该是：
>1: 分配一块内存M；
2: 在内存M上初始化Singleton对象；
3: 然后M的地址赋值给instance变量。

但是实际上优化后的执行路径却是这样的：

>1: 分配一块内存M；
2: 将M的地址赋值给instance变量；
3: 最后在内存M上初始化Singleton对象。

优化后会导致什么问题呢？我们假设线程A先执行getInstance()方法，当执行完指令2时恰好发生了线程切换，切换到了线程B上；如果此时线程B也执行getInstance()方法，那么线程B在执行第一个判断时会发现 instance != null ，所以直接返回instance，而此时的instance是没有初始化过的，如果我们这个时候访问 instance 的成员变量就可能触发空指针异常。
![在这里插入图片描述](https://img-blog.csdnimg.cn/8d2ed4b9231841f492911767ec8a44f7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bem5p6X5Y-z5p2OMDI=,size_20,color_FFFFFF,t_70,g_se,x_16)
# 解决方法
单例对象添加volatile修饰
```java
public class Singleton {
  volatile static Singleton instance;
  static Singleton getInstance(){
    if (instance == null) {
      synchronized(Singleton.class) {
        if (instance == null)
          instance = new Singleton();
        }
    }
    return instance;
  }
}
```
# volatile如何防止指令重排和内存可见性



