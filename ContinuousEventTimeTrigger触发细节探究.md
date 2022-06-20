#### 背景
> 现有业务需要在同一个页面展示全天人次和分时人次，全天和分时对应两个任务，消费同一个topic，两者都使用ContinuousEventTimeTrigger，每隔30s trigger window的计算，两任务operator的背压都是0，那么现在问题来了，如果两个任务间隔10s先后启动，那么之后这两个任务的trigger的时间点是相同的吗？![在这里插入图片描述](https://img-blog.csdnimg.cn/20200908162331123.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70#pic_center)

#### 个人误解
误认为任务触发的时间为任务启动时间+n*trigger interval，即任务的触发时间和任务的启动时间有关，如下图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200908163711781.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70#pic_center)
#### 源码正解
##### ReducingState
ContinuousEventTimeTrigger类使用ReducingState来保存要trigger的时间戳（long型）
```java
private final ReducingStateDescriptor<Long> stateDesc =
			new ReducingStateDescriptor<>("fire-time", new Min(), LongSerializer.INSTANCE);
```
ReducingState中reduce function为Min，功能为返回较小的时间戳，flink对此注释为
/** When merging we take the lowest of all fire timestamps as the new fire timestamp. */
```java
private static class Min implements ReduceFunction<Long> {
		private static final long serialVersionUID = 1L;

		@Override
		public Long reduce(Long value1, Long value2) throws Exception {
			return Math.min(value1, value2);
		}
	}
```
定义好了state后，来看下操作state的两个方法
##### onElement
```java
@Override
public TriggerResult onElement(Object element, long timestamp, W window, TriggerContext ctx) throws Exception {
        //如果window的最大时间戳<当前watermark,则fire,否则将window的最大时间戳注册为timer
		if (window.maxTimestamp() <= ctx.getCurrentWatermark()) {
			// if the watermark is already past the window fire immediately
			return TriggerResult.FIRE;
		} else {
			ctx.registerEventTimeTimer(window.maxTimestamp());
		}
        //获取ReducingState,如果ReducingState.get为null,则使用根据element的timestamp注册一个timer
		ReducingState<Long> fireTimestamp = ctx.getPartitionedState(stateDesc);
		if (fireTimestamp.get() == null) {
			long start = timestamp - (timestamp % interval);
			long nextFireTimestamp = start + interval;
			ctx.registerEventTimeTimer(nextFireTimestamp);
			fireTimestamp.add(nextFireTimestamp);
		}

		return TriggerResult.CONTINUE;
	}
```
注意以上代码的细节：
```java
long start = timestamp - (timestamp % interval);
long nextFireTimestamp = start + interval;
ctx.registerEventTimeTimer(nextFireTimestamp);
fireTimestamp.add(nextFireTimestamp);
```
`根据element的timestamp注册的timer为1970-01-01 08:00:00+N*trigger_interval,与任务启动时间无关`
##### onEventTime

```java
@Override
public TriggerResult onEventTime(long time, W window, TriggerContext ctx) throws Exception {

		if (time == window.maxTimestamp()){
			return TriggerResult.FIRE;
		}

		ReducingState<Long> fireTimestampState = ctx.getPartitionedState(stateDesc);

		Long fireTimestamp = fireTimestampState.get();

        //如果fireTimestamp==event_time，则触发计算
        //触发计算的第一步就是clear ReducingState,一以便注册下一个trigger时间
		if (fireTimestamp != null && fireTimestamp == time) {
			fireTimestampState.clear();
			fireTimestampState.add(time + interval);
			ctx.registerEventTimeTimer(time + interval);
			return TriggerResult.FIRE;
		}

		return TriggerResult.CONTINUE;
}
```
