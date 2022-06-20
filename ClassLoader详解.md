# Java 类加载顺序
ClassLoader 的顺序
1. Bootstrap ClassLoader
加载 Java lib 底下的包

2. Extensions ClassLoader
    加载 Java lib/ext 底下的包

3. System ClassLoader
    加载 Java classpath 底下的包
4. Custom ClassLoader（用户自定义ClassLoader）


# 为什么叫双亲委派
 一个类加载器查找class和resource时，是通过“委托模式”进行的，它首先判断这个class是不是已经加载成功，如果没有的话它并不是自己进行查找，而是先通过父加载器，然后递归下去，直到Bootstrap ClassLoader，如果Bootstrap classloader找到了，直接返回，如果没有找到，则一级一级返回，最后到达自身去查找这些对象。这种机制就叫做双亲委托。

加载时序图如下
 
 ![在这里插入图片描述](https://img-blog.csdnimg.cn/481f8d67ec944a5aa8c827345eeb7534.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bem5p6X5Y-z5p2OMDI=,size_20,color_FFFFFF,t_70,g_se,x_16)

## 代码详解
java.lang.ClassLoader.loadClass方法
```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                ...省略部分代码
            return c;
        }
    }
```
## 父加载器并不是父类
AppClassLoader的父类是URLClassLoader，而不是ExtClassLoader，类关系图如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/7f4086ab7edb437cae65361a3b2551fa.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5bem5p6X5Y-z5p2OMDI=,size_16,color_FFFFFF,t_70,g_se,x_16)

