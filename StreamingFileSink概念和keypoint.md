以下内容基于flink 1.10
==使用StreamingFileSink需要注意的几点：==
- <font color="red">在使用StreamingFileSink时需要开启checkpoint支持</font>，因为part files只有在做checkpoint时才会finalize,否则文件状态一直是`in-progress` or `pending`状态，下游非常容易误读。
- <font color="red">在使用的hadoop版本<2.7时，只能使用OnCheckpointRollingPolicy</font>，因为OnCheckpointRollingPolicy只根据checkpoint滚动part file。hadoop2.7及之后的版本可以实现当一个part file跨越两个checkpoint interval时，StreamingFileSink进行失败恢复时会使用file system的truncate方法来抛弃in-progress文件中uncommitted数据，这个方法在hadoop 2.7之前是不支持的，flink会抛异常
如果是hadoop>=2.7，则可以不使用OnCheckpointRollingPolicy。
- <font color="red">flink的StreamingFileSink不会重写committed的文件，鉴于此，当尝试从旧的检查点/保存点还原时（假定该文件是后续执行的成功检查点提交的正在进行的文件），Flink将拒绝继续进行操作，并且由于无法找到进行中的文件而将引发异常</font>。flink sink到hdfs的任务无法从savepoint恢复，原因是找不到in-progress的文件。
- Rolling Policy决定了in-progress文件何时会关闭并转换成为pending状态，<font color="red">而令pending文件转化为finished则只有checkpoint,只有flink做checkpoint时才会将pending part file转化为finished</font>

#### flink hdfs sink基本概念

flink有按行和批量两种写入文件的方法，比如要想用 Apache Parquet格式存储文件，就有两种builder创建方法：
- 按行编码的sink:StreamingFileSink.forRowFormat(basePath, rowEncoder)
- 批量编码的sink:StreamingFileSink.forBulkFormat(basePath, bulkWriterFactory)

#### part fille的命名规则和lifecycle
part file有三种状态：
- in-progress:表示当前part file正在被写入
- pending:写入完成后等待提交的状态（flink写hdfs的二段提交的体现）
- finished:写入完成状态，只有finished状态下的文件才会保证不会再有修改，下游可安全读取

#### Part file example


为了更好的理解part file的生命周期，我们来看一个有两个子任务的例子（Parallelism=2）

└── 2019-08-25--12
    ├── part-0-0.inprogress.bd053eb0-5ecf-4c85-8433-9eff486ac334
    └── part-1-0.inprogress.ea65a428-a1d0-4a0b-bbc5-7a436a75e575

当part file part-1-0滚动时（假设这个文件太大触发rollingpolicy），它变成pending状态，但是名称并没有改变，streamingfilesink会开一个新的part file:part-1-1

└── 2019-08-25--12
    ├── part-0-0.inprogress.bd053eb0-5ecf-4c85-8433-9eff486ac334
    ├── part-1-0.inprogress.ea65a428-a1d0-4a0b-bbc5-7a436a75e575
    └── part-1-1.inprogress.bc279efe-b16f-47d8-b828-00ef6e2fbd11
As part-1-0 is now pending completion, after the next successful checkpoint, it is finalized:
随着checkpoint完成，part-1-0 pending状态结束，pending文件重新命名，变成finished状态：

└── 2019-08-25--12
    ├── part-0-0.inprogress.bd053eb0-5ecf-4c85-8433-9eff486ac334
    ├── part-1-0
    └── part-1-1.inprogress.bc279efe-b16f-47d8-b828-00ef6e2fbd11


#### Rolling Policy
The RollingPolicy defines when a given in-progress part file will be closed and moved to the pending and later to finished state. Part files in the “finished” state are the ones that are ready for viewing and are guaranteed to contain valid data that will not be reverted in case of failure. The Rolling Policy in combination with the checkpointing interval (pending files become finished on the next checkpoint) control how quickly part files become available for downstream readers and also the size and number of these parts.
Rolling Policy决定了in-progress文件何时会关闭并转换成为pending状态，而决定

#### Part file命名规则
part file默认命名规则如下：
- In-progress / Pending: part-<subtaskIndex>-<partFileIndex>.inprogress.uid
- Finished: part-<subtaskIndex>-<partFileIndex>

flink允许用户给part file自定义前缀和后缀，通过OutputFileConfig即可配置，比如使用"prefix"和"ext"分别作为前、后缀，得到的文件如下：
└── 2019-08-25--12
    ├── prefix-0-0.ext
    ├── prefix-0-1.ext.inprogress.bd053eb0-5ecf-4c85-8433-9eff486ac334
    ├── prefix-1-0.ext
    └── prefix-1-1.ext.inprogress.bc279efe-b16f-47d8-b828-00ef6e2fbd11

OutputFileConfig代码实例：
```java
OutputFileConfig config = OutputFileConfig
 .builder()
 .withPartPrefix("prefix")
 .withPartSuffix(".ext")
 .build();
            
StreamingFileSink<Tuple2<Integer, Integer>> sink = StreamingFileSink
 .forRowFormat((new Path(outputPath), new SimpleStringEncoder<>("UTF-8"))
 .withBucketAssigner(new KeyBucketAssigner())
 .withRollingPolicy(OnCheckpointRollingPolicy.build())
 .withOutputFileConfig(config)
 .build();
```


