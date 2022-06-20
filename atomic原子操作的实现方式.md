# 1.无锁方案的实现原理
其实原子类性能高的秘密很简单，硬件支持而已。CPU为了解决并发问题，提供了CAS指令（CAS，全称是Compare And Swap，即“比较并交换”）。CAS指令包含3个参数：共享变量的内存地址A、用于比较的值B和共享变量的新值C；并且只有当内存中地址A处的值等于B时，才能将内存中地址A处的值更新为新值C。作为一条CPU指令，CAS指令本身是能够保证原子性的。

你可以通过下面CAS指令的模拟代码来理解CAS的工作原理。在下面的模拟程序中有两个参数，一个是期望值expect，另一个是需要写入的新值newValue，只有当目前count的值和期望值expect相等时，才会将count更新为newValue。
```java
class SimulatedCAS{
  int count；
  synchronized int cas(
    int expect, int newValue){
    // 读目前count的值
    int curValue = count;
    // 比较目前count值是否==期望值
    if(curValue == expect){
      // 如果是，则更新count的值
      count = newValue;
    }
    // 返回写入前的值
    return curValue;
  }
}
```
你仔细地再次思考一下这句话，“只有当目前count的值和期望值expect相等时，才会将count更新为newValue。”要怎么理解这句话呢？

对于前面提到的累加器的例子，count += 1 的一个核心问题是：基于内存中count的当前值A计算出来的count+=1为A+1，在将A+1写入内存的时候，很可能此时内存中count已经被其他线程更新过了，这样就会导致错误地覆盖其他线程写入的值。也就是说，只有当内存中count的值等于期望值A时，才能将内存中count的值更新为计算结果A+1，这不就是CAS的语义吗！

使用CAS来解决并发问题，一般都会伴随着自旋，而所谓自旋，其实就是循环尝试。例如，实现一个线程安全的count += 1操作，“CAS+自旋”的实现方案如下所示，首先计算newValue = count+1，如果cas(count,newValue)返回的值不等于count，则意味着线程在执行完代码①处之后，执行代码②处之前，count的值被其他线程更新过。那此时该怎么处理呢？可以采用自旋方案，就像下面代码中展示的，可以重新读count最新的值来计算newValue并尝试再次更新，直到成功。
```java
class SimulatedCAS{
  volatile int count;
  // 实现count+=1
  addOne(){
    do {
      newValue = count+1; //①
    }while(count !=
      cas(count,newValue) //②
  }
  // 模拟实现CAS，仅用来帮助理解
  synchronized int cas(
    int expect, int newValue){
    // 读目前count的值
    int curValue = count;
    // 比较目前count值是否==期望值
    if(curValue == expect){
      // 如果是，则更新count的值
      count= newValue;
    }
    // 返回写入前的值
    return curValue;
  }
}
```
通过上面的示例代码，想必你已经发现了，CAS这种无锁方案，完全没有加锁、解锁操作，即便两个线程完全同时执行addOne()方法，也不会有线程被阻塞，所以相对于互斥锁方案来说，性能好了很多。

## 1.1.ABA问题
但是在CAS方案中，有一个问题可能会常被你忽略，那就是ABA的问题。什么是ABA问题呢？

前面我们提到“如果cas(count,newValue)返回的值不等于count，意味着线程在执行完代码①处之后，执行代码②处之前，count的值被其他线程更新过”，那如果cas(count,newValue)返回的值等于count，是否就能够认为count的值没有被其他线程更新过呢？显然不是的，假设count原本是A，线程T1在执行完代码①处之后，执行代码②处之前，有可能count被线程T2更新成了B，之后又被T3更新回了A，这样线程T1虽然看到的一直是A，但是其实已经被其他线程更新过了，这就是ABA问题。

可能大多数情况下我们并不关心ABA问题，例如数值的原子递增，但也不能所有情况下都不关心，例如原子化的更新对象很可能就需要关心ABA问题，因为两个A虽然相等，但是第二个A的属性可能已经发生变化了。所以在使用CAS方案的时候，一定要先check一下。

# 2.看Java如何实现原子化的count += 1
在本文开始部分，我们使用原子类AtomicLong的getAndIncrement()方法替代了count += 1，从而实现了线程安全。原子类AtomicLong的getAndIncrement()方法内部就是基于CAS实现的，下面我们来看看Java是如何使用CAS来实现原子化的count += 1的。

在Java 1.8版本中，getAndIncrement()方法会转调unsafe.getAndAddLong()方法。这里this和valueOffset两个参数可以唯一确定共享变量的内存地址。
```java
final long getAndIncrement() {
  return unsafe.getAndAddLong(
    this, valueOffset, 1L);
}
```
unsafe.getAndAddLong()方法的源码如下，该方法首先会在内存中读取共享变量的值，之后循环调用compareAndSwapLong()方法来尝试设置共享变量的值，直到成功为止。compareAndSwapLong()是一个native方法，只有当内存中共享变量的值等于expected时，才会将共享变量的值更新为x，并且返回true；否则返回fasle。compareAndSwapLong的语义和CAS指令的语义的差别仅仅是返回值不同而已。
```java
public final long getAndAddLong(
  Object o, long offset, long delta){
  long v;
  do {
    // 读取内存中的值
    v = getLongVolatile(o, offset);
  } while (!compareAndSwapLong(
      o, offset, v, v + delta));
  return v;
}
//原子性地将变量更新为x
//条件是内存中的值等于expected
//更新成功则返回true
native boolean compareAndSwapLong(
  Object o, long offset, 
  long expected,
  long x);
  ```
另外，需要你注意的是，getAndAddLong()方法的实现，基本上就是CAS使用的经典范例。所以请你再次体会下面这段抽象后的代码片段，它在很多无锁程序中经常出现。Java提供的原子类里面CAS一般被实现为compareAndSet()，compareAndSet()的语义和CAS指令的语义的差别仅仅是返回值不同而已，compareAndSet()里面如果更新成功，则会返回true，否则返回false。
```
do {
  // 获取当前值
  oldV = xxxx；
  // 根据当前值计算新值
  newV = ...oldV...
}while(!compareAndSet(oldV,newV);
```
# 3.代码demo
用原子类AtomicLong实现线程安全的add10k
```java
AtomicLong countAtomic = new AtomicLong(0);

    void add10KAtomic() {
        int idx = 0;
        while (idx++ < 10000) {
            countAtomic.getAndIncrement();
        }
    }

    public static long calcAtomic() throws InterruptedException {
        final TestT1 testT1 = new TestT1();
        // 创建两个线程，执行add()操作
        Thread th1 = new Thread(() -> {
            testT1.add10KAtomic();
        });
        Thread th2 = new Thread(() -> {
            testT1.add10KAtomic();
        });
        // 启动两个线程
        th1.start();
        th2.start();
        // 等待两个线程执行结束
        th1.join();
        th2.join();
        return testT1.countAtomic.get();
    }

    public static void main(String[] args) throws InterruptedException {
        System.out.println(TestT1.calcAtomic());
    }
```
接下来看一下Java的原子类是怎么实现原子操作的
```java
AtomicLong类
public final long getAndIncrement() {
        return unsafe.getAndAddLong(this, valueOffset, 1L);
}

Unsafe类
public final long getAndAddLong(Object o, long offset, long delta) {
        long v;
        /**
         * 这里do while是一个自旋操作，如果内存位置上的值和期望值不同，则重新获取内存位置上的值，重新尝试加操作
         */
        do {
            v = getLongVolatile(o, offset);
        } while (!weakCompareAndSetLong(o, offset, v, v + delta));
        return v;
}

public final boolean weakCompareAndSetLong(Object o, long offset,
                                               long expected,
                                               long x) {
        return compareAndSetLong(o, offset, expected, x);
}

这一步是对应cpu的一个指令，compare and swap
public final native boolean compareAndSetLong(Object o, long offset,
                                                  long expected,
                                                  long x);
```
