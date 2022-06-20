A Stream Status element informs stream tasks whether or not they should continue to expect records and watermarks
 * from the input stream that sent them. There are 2 kinds of status, namely {@link StreamStatus#IDLE} and
 * {@link StreamStatus#ACTIVE}. Stream Status elements are generated at the sources, and may be propagated through
 * the tasks of the topology. They directly infer the current status of the emitting task; a {@link SourceStreamTask} or
 * {@link StreamTask} emits a {@link StreamStatus#IDLE} if it will temporarily halt to emit any records or watermarks
 * (i.e. is idle), and emits a {@link StreamStatus#ACTIVE} once it resumes to do so (i.e. is active). Tasks are
 * responsible for propagating their status further downstream once they toggle between being idle and active. The cases
 * that source tasks and downstream tasks are considered either idle or active is explained below:
 *
 * <ul>
 *     <li>Source tasks: A source task is considered to be idle if its head operator, i.e. a {@link StreamSource}, will
 *         not emit records for an indefinite amount of time. This is the case, for example, for Flink's Kafka Consumer,
 *         where sources might initially have no assigned partitions to read from, or no records can be read from the
 *         assigned partitions. Once the head {@link StreamSource} operator detects that it will resume emitting data,
 *         the source task is considered to be active. {@link StreamSource}s are responsible for toggling the status
 *         of the containing source task and ensuring that no records (and possibly watermarks, in the case of Flink's
 *         Kafka Consumer which can generate watermarks directly within the source) will be emitted while the task is
 *         idle. This guarantee should be enforced on sources through
 *         {@link org.apache.flink.streaming.api.functions.source.SourceFunction.SourceContext} implementations.</li>
 *
 *     <li>Downstream tasks: a downstream task is considered to be idle if all its input streams are idle, i.e. the last
 *         received Stream Status element from all input streams is a {@link StreamStatus#IDLE}. As long as one of its
 *         input streams is active, i.e. the last received Stream Status element from the input stream is
 *         {@link StreamStatus#ACTIVE}, the task is active.</li>
 * </ul>
 *
 * <p>Stream Status elements received at downstream tasks also affect and control how their operators process and advance
 * their watermarks. The below describes the effects (the logic is implemented as a {@link StatusWatermarkValve} which
 * downstream tasks should use for such purposes):
 *
 * <ul>
 *     <li>Since source tasks guarantee that no records will be emitted between a {@link StreamStatus#IDLE} and
 *         {@link StreamStatus#ACTIVE}, downstream tasks can always safely process and propagate records through their
 *         operator chain when they receive them, without the need to check whether or not the task is currently idle or
 *         active. However, for watermarks, since there may be watermark generators that might produce watermarks
 *         anywhere in the middle of topologies regardless of whether there are input data at the operator, the current
 *         status of the task must be checked before forwarding watermarks emitted from
 *         an operator. If the status is actually idle, the watermark must be blocked.
 *
 *     <li>For downstream tasks with multiple input streams, the watermarks of input streams that are temporarily idle,
 *         or has resumed to be active but its watermark is behind the overall min watermark of the operator, should not
 *         be accounted for when deciding whether or not to advance the watermark and propagated through the operator
 *         chain.
 * </ul>
 *
 * <p>Note that to notify downstream tasks that a source task is permanently closed and will no longer send any more
 * elements, the source should still send a {@link Watermark#MAX_WATERMARK} instead of {@link StreamStatus#IDLE}.
 * Stream Status elements only serve as markers for temporary status.