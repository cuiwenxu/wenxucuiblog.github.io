
- Map
**功能：输入一个元素，转换成另一个元素，一对一的关系**
**流的变换：<font color="red">DataStream → DataStream</font>**
```java
//元素的值乘2
DataStream<Integer> dataStream = //...
dataStream.map(new MapFunction<Integer, Integer>() {
    @Override
    public Integer map(Integer value) throws Exception {
        return 2 * value;
    }
});
```
- FlatMap
**功能：输入一个元素，转换成一个元素，零个或多个，输入和输出元素类型需要在flatMap参数中指定，flatMap(InputType in,Collector<OutputType> out)**
**流的变换：<font color="red">DataStream → DataStream</font>**
```java
//将一个string拆成多个单词
dataStream.flatMap(new FlatMapFunction<String, String>() {
    @Override
    public void flatMap(String value, Collector<String> out)
        throws Exception {
        for(String word: value.split(" ")){
            out.collect(word);
        }
    }
});
```
- Filter
**功能：为每个元素计算一个布尔值，并保留该布尔值为true的元素**
**流的变换：<font color="red">DataStream → DataStream</font>**
```java
//返回值不等于0的元素
dataStream.filter(new FilterFunction<Integer>() {
    @Override
    public boolean filter(Integer value) throws Exception {
        return value != 0;
    }
});
```
- KeyBy
**功能：从逻辑上将流划分为不相交的分区。 具有相同键的所有记录都分配给同一分区。 原理上是通过哈希分区实现的。 有多种指定key的方法。**
**流的变换：<font color="red">DataStream → KeyedStream</font>**
```java
dataStream.keyBy("someKey") // Key by field "someKey"
dataStream.keyBy(0) // Key by the first element of a Tuple
```
> 需要注意的是，以下两种类型的元素不能作为key
> 1、如果一个POJO类型的元素未覆盖hashCode方法则不能作为key.
2、array类型的元素
- Reduce
**功能：将当前元素和reduce产生的结果结合，并输出一个新的值，对keyed stream元素滚动执行**
**流的变换：<font color="red">KeyedStream → DataStream</font>**
```java
//
keyedStream.reduce(new ReduceFunction<Integer>() {
    @Override
    public Integer reduce(Integer value1, Integer value2)
    throws Exception {
        return value1 + value2;
    }
});
```
