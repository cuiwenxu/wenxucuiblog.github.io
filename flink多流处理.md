# 概览
离线批处理中两个表做关联很简单，把两个表的数据根据关联条件做shuffle，然后做匹配就可以了，但是实时的两个流做关联就比较复杂，需要考虑的问题有：
- 两个流是无限的，其中一个流来一条新数据，去另一个流做关联时，怎么确定关联的范围？
- 如果关联范围确定了，那么左右两个流的数据怎么存放？如果使用缓存，那么缓存的清理策略是什么？
- 怎么做排列组合？是full outer join(笛卡尔积)还是left join
- 适用场景有哪些？
接下来分析flink提供的集中流关联的方式，找到以上四个问题的答案

# JoinedStreams & CoGroupedStreams
1、怎么确定关联范围
JoinedStreams和CoGroupedStreams的范围都是通过开窗来确定的，两个流具有相同key的record被放到同一个window
```java
DataStream<T> result = one.join(two)
   .where(new MyFirstKeySelector())
   .equalTo(new MyFirstKeySelector())
   .window(TumblingEventTimeWindows.of(Time.of(5, TimeUnit.SECONDS)))
   .apply(new MyJoinFunction());
 ```
2、怎么存放左右流的数据
>放到jvm内存中，很有可能导致内存溢出

3、怎么做排列组合
JoinedStreams是对`每一对`元素做笛卡尔积，JoinedStreams相当于CoGroupedStream的特例
```java
public void coGroup(Iterable<T1> first, Iterable<T2> second, Collector<T> out)
                throws Exception {
            for (T1 val1 : first) {
                for (T2 val2 : second) {
                    out.collect(wrappedFunction.join(val1, val2));
                }
            }
        }
```
CoGroupedStream是对两侧流的`两组`元素做处理，组合后产出的结果由用户自定义
```java
public void apply(KEY key, W window, Iterable<TaggedUnion<T1, T2>> values, Collector<T> out)
                throws Exception {

            List<T1> oneValues = new ArrayList<>();
            List<T2> twoValues = new ArrayList<>();

            for (TaggedUnion<T1, T2> val : values) {
                if (val.isOne()) {
                    oneValues.add(val.getOne());
                } else {
                    twoValues.add(val.getTwo());
                }
            }
            wrappedFunction.coGroup(oneValues, twoValues, out);
        }
        
```
4、适用场景

> 两个事件流，比如订单和红包适用，关联之后用于生成新的较宽的事件流
# ConnectedStreams
1、怎么确定关联范围

依靠用户自定义实现，比如A.connect(B)，可以把B的数据都放到state中，然后对于A中每一条数据都和B所有的数据进行匹配

2、怎么存放左右流的数据

依靠自定义state存放

3、怎么做排列组合
依靠用户自定义实现，可以是full outer join(笛卡尔积) 或者 left join

4、适用场景有哪些


