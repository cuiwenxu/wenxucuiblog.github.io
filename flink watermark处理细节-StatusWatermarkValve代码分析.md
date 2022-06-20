首先抛出一个问题：
> kafka topic下有3个partition，下游consumer为flink job，flink job的并行度为4，如下图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201021202232336.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70#pic_center)
那么window operator的watermark是否会一直很小，导致窗口迟迟不触发计算

理清这个问题需要看flink对watermark的处理，StatusWatermarkValve类嵌入了Watermark和StreamStatus两种元素怎么发送到下游到逻辑，inputStreamStatus方法包含了主要的处理逻辑
```java
public void inputStreamStatus(StreamStatus streamStatus, int channelIndex) {
		// only account for stream status inputs that will result in a status change for the input channel
		if (streamStatus.isIdle() && channelStatuses[channelIndex].streamStatus.isActive()) {
			// handle active -> idle toggle for the input channel
			channelStatuses[channelIndex].streamStatus = StreamStatus.IDLE;

			// the channel is now idle, therefore not aligned
			channelStatuses[channelIndex].isWatermarkAligned = false;

			// if all input channels of the valve are now idle, we need to output an idle stream
			// status from the valve (this also marks the valve as idle)
			if (!InputChannelStatus.hasActiveChannels(channelStatuses)) {

				// now that all input channels are idle and no channels will continue to advance its watermark,
				// we should "flush" all watermarks across all channels; effectively, this means emitting
				// the max watermark across all channels as the new watermark. Also, since we already try to advance
				// the min watermark as channels individually become IDLE, here we only need to perform the flush
				// if the watermark of the last active channel that just became idle is the current min watermark.
				if (channelStatuses[channelIndex].watermark == lastOutputWatermark) {
					findAndOutputMaxWatermarkAcrossAllChannels();
				}

				lastOutputStreamStatus = StreamStatus.IDLE;
				outputHandler.handleStreamStatus(lastOutputStreamStatus);
			} else if (channelStatuses[channelIndex].watermark == lastOutputWatermark) {
				// if the watermark of the channel that just became idle equals the last output
				// watermark (the previous overall min watermark), we may be able to find a new
				// min watermark from the remaining aligned channels
				findAndOutputNewMinWatermarkAcrossAlignedChannels();
			}
		} else if (streamStatus.isActive() && channelStatuses[channelIndex].streamStatus.isIdle()) {
			// handle idle -> active toggle for the input channel
			channelStatuses[channelIndex].streamStatus = StreamStatus.ACTIVE;

			// if the last watermark of the input channel, before it was marked idle, is still larger than
			// the overall last output watermark of the valve, then we can set the channel to be aligned already.
			if (channelStatuses[channelIndex].watermark >= lastOutputWatermark) {
				channelStatuses[channelIndex].isWatermarkAligned = true;
			}

			// if the valve was previously marked to be idle, mark it as active and output an active stream
			// status because at least one of the input channels is now active
			if (lastOutputStreamStatus.isIdle()) {
				lastOutputStreamStatus = StreamStatus.ACTIVE;
				outputHandler.handleStreamStatus(lastOutputStreamStatus);
			}
		}
	}

```
以上代码可用如下流程图概括
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020102121410174.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70#pic_center)

todo:KafkaConsumer长时间不接收数据，怎么切换StreamStatus


