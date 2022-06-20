flink官网
>Interaction of watermarks and windows
Before continuing in this section you might want to take a look at our section about event time and watermarks.
When watermarks arrive at the window operator this triggers two things:
the watermark triggers computation of all windows where the maximum timestamp (which is end-timestamp - 1) is smaller than the new watermark
the watermark is forwarded (as is) to downstream operations
Intuitively, a watermark “flushes” out any windows that would be considered late in downstream operations once they receive that watermark.

watermark到达window operator时会触发两件事：
1、watermark会trigger窗口end-timestamp-1<watermark的窗口的计算
2、watermark会传递下去，触发下游的窗口计算

watermark和window之间的关系可用下图概括：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200410152416953.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
org.apache.flink.streaming.api.windowing.triggers.ContinuousEventTimeTrigger类中onElement方法可以看出context中的watermark会和window右侧时间戳进行比较，来决定是否trigger计算
```java
@Override
	public TriggerResult onElement(Object element, long timestamp, W window, TriggerContext ctx) throws Exception {

		if (window.maxTimestamp() <= ctx.getCurrentWatermark()) {
			// if the watermark is already past the window fire immediately
			return TriggerResult.FIRE;
		} else {
			ctx.registerEventTimeTimer(window.maxTimestamp());
		}

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
