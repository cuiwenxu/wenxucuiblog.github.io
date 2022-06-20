Flink 支持两种 watermark 生成方式:
- 第一种是在 SourceFunction 中产生，相当于把整个的 timestamp 分配和 watermark 生成的逻辑放在流处理应用的源头。我们可以在SourceFunction 在sourcefunction的run方法中，通过SourceContext的collectWithTimestamp和emitWatermark这两个方法可以产生 watermark
```java
@Override
public void run(SourceContext<MyType> ctx) throws Exception {
	while (/* condition */) {
		MyType next = getNext();
		ctx.collectWithTimestamp(next, next.getEventTimestamp());

		if (next.hasWatermarkTime()) {
			ctx.emitWatermark(new Watermark(next.getWatermarkTime()));
		}
	}
}
```
- 有时候我们不想在 SourceFunction 里生成 timestamp 或者 watermark，或者说使用的 SourceFunction 本身不支持，我们还可以在使用 DataStream API 的时候指定，调用的 DataStream.assignTimestampsAndWatermarks 这个方法，能够接收不同的 timestamp 和 watermark 的生成器。
timestamp 和 watermark 的生成器主要有两类
>1、AssignerWithPeriodicWatermarks
  2、AssignerWithPunctuatedWatermarks

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200413195547301.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
  其中AssignerWithPeriodicWatermarks还有一个重要的子类AscendingTimestampExtractor，这个类适用于具有单调递增timestamp的数据流。
