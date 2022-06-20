- FlinkKafkaConsumer是怎么提交offset的
FlinkKafkaConsumer提交offset的模式有以下三种
```java
public enum OffsetCommitMode {

	/** Completely disable offset committing. */
	DISABLED,

	/** Commit offsets back to Kafka only when checkpoints are completed. */
	ON_CHECKPOINTS,

	/** Commit offsets periodically back to Kafka, using the auto commit functionality of internal Kafka clients. */
	KAFKA_PERIODIC;
}
```
在创建FlinkKafkaConsumer时会确定offset的提交模式，FlinkKafkaConsumer的父类FlinkKafkaConsumerBase的open方法
```java
public void open(Configuration configuration) throws Exception {
		// determine the offset commit mode
		this.offsetCommitMode = OffsetCommitModes.fromConfiguration(
				getIsAutoCommitEnabled(),
				enableCommitOnCheckpoints,
				((StreamingRuntimeContext) getRuntimeContext()).isCheckpointingEnabled());
				
		// 此处省略部分代码
}
```
open方法所调用的OffsetCommitModes.fromConfiguration方法,其中enableCommitOnCheckpoint默认为true
```java
public static OffsetCommitMode fromConfiguration(
			boolean enableAutoCommit,
			boolean enableCommitOnCheckpoint,
			boolean enableCheckpointing) {

		if (enableCheckpointing) {
			// if checkpointing is enabled, the mode depends only on whether committing on checkpoints is enabled
			return (enableCommitOnCheckpoint) ? OffsetCommitMode.ON_CHECKPOINTS : OffsetCommitMode.DISABLED;
		} else {
			// else, the mode depends only on whether auto committing is enabled in the provided Kafka properties
			return (enableAutoCommit) ? OffsetCommitMode.KAFKA_PERIODIC : OffsetCommitMode.DISABLED;
		}
	}
```

