Sink function to write the data to the underneath filesystem.
 *
 * <p><h2>Work Flow</h2>
 *
 * <p>The function firstly buffers the data as a batch of {@link HoodieRecord}s,
 * It flushes(write) the records batch when the batch size exceeds the configured size {@link FlinkOptions#WRITE_BATCH_SIZE}
 * or the total buffer size exceeds the configured size {@link FlinkOptions#WRITE_TASK_MAX_SIZE}
 * or a Flink checkpoint starts. After a batch has been written successfully,
 * the function notifies its operator coordinator {@link StreamWriteOperatorCoordinator} to mark a successful write.
 *
 * <p><h2>The Semantics</h2>
 *
 * <p>The task implements exactly-once semantics by buffering the data between checkpoints. The operator coordinator
 * starts a new instant on the time line when a checkpoint triggers, the coordinator checkpoints always
 * start before its operator, so when this function starts a checkpoint, a REQUESTED instant already exists.
 *
 * <p>The function process thread blocks data buffering after the checkpoint thread finishes flushing the existing data buffer until
 * the current checkpoint succeed and the coordinator starts a new instant. Any error triggers the job failure during the metadata committing,
 * when the job recovers from a failure, the write function re-send the write metadata to the coordinator to see if these metadata
 * can re-commit, thus if unexpected error happens during the instant committing, the coordinator would retry to commit when the job
 * recovers.
 *
 * <p><h2>Fault Tolerance</h2>
 *
 * <p>The operator coordinator checks and commits the last instant then starts a new one after a checkpoint finished successfully.
 * It rolls back any inflight instant before it starts a new instant, this means one hoodie instant only span one checkpoint,
 * the write function blocks data buffer flushing for the configured checkpoint timeout
 * before it throws exception, any checkpoint failure would finally trigger the job failure.
 *
 * <p>Note: The function task requires the input stream be shuffled by the file IDs.


sink function负责将数据写入文件系统
# 工作流程
该函数首先将数据缓冲为一批HoodieRecord，
当满足以下某条件时，将会进行刷写
- 批大小超过`FlinkOptions.WRITE_BATCH_SIZE`配置的大小时
- 总缓冲区大小超过`FlinkOptions.WRITE_TASK_MAX_SIZE`配置的大小
- Flink做checkpoint时
当一批写成功后，该function会通知其operator coordinator(StreamWriteOperatorCoordinator)去记录一次成功的写入。
# 语义
该function对应任务通过缓冲检查点之间的数据来实现精确一次的语义。当checkpoint trigger时，operator coordinator会在timeline上新建一个instant(瞬态)，coordinator的checkpoint总是在其对应的operator之前启动，因此当sink function启动检查点时，一个REQUESTED instant已经存在。
# 容错
当一个checkpoint成功完成后，一个instant会随之commit，并且在创建新的instant之前
操作员协调器检查并提交最后一个瞬间，然后在检查点成功完成后启动一个新的瞬间。
*它会在任何飞行瞬间开始一个新的瞬间前回滚，这意味着一个连帽衫瞬间只能跨越一个检查站，
*写入函数阻塞配置的检查点超时的数据缓冲区刷新
*在它抛出异常之前，任何检查点失败都会最终触发作业失败。
＊
* 
注意:函数任务要求输入流被文件id打乱。
