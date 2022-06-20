@[TOC](interval join实现解析)
interval join的实现主要依靠IntervalJoinOperator，下面详细分析IntervalJoinOperator代码
# 概览
IntervalJoinOperator可以通过使用可配置的下界和上界，精确地发出(T1, T2)其中T2.ts∈(T1.ts +下界，T1.ts + upperBound]。下界和上界都可以配置为包含或排他。
元素一旦连接起来，就会传递给用户定义的ProcessJoinFunction。
这个实现的基本思想如下:每当我们在processElement1(StreamRecord)(也就是左边)接收到一个元素时，我们将它添加到左边缓冲区。然后检查右侧的缓冲区，看看是否有可以连接的元素。如果有，则将它们连接并传递给上述函数。当接收到右边的元素时，同样的情况也会发生。
发送的这一对元素的时间戳=max(左侧元素时间戳，右侧元素时间戳)。
为了避免元素缓冲区无限增长，我们为每个元素注册一个清理计时器。这个计时器指示什么时候一个元素不再被join并且可以从状态中删除。
# 怎么管理左右缓存
管理缓存的逻辑在IntervalJoinOperator.processElement方法中，每来一个元素，都会调用internalTimerService.registerEventTimeTimer方法为该元素注册一个cleanupTime，代码如下
```java
private <THIS, OTHER> void processElement(
            final StreamRecord<THIS> record,
            final MapState<Long, List<IntervalJoinOperator.BufferEntry<THIS>>> ourBuffer,
            final MapState<Long, List<IntervalJoinOperator.BufferEntry<OTHER>>> otherBuffer,
            final long relativeLowerBound,
            final long relativeUpperBound,
            final boolean isLeft)
            throws Exception {

        final THIS ourValue = record.getValue();
        final long ourTimestamp = record.getTimestamp();

        if (ourTimestamp == Long.MIN_VALUE) {
            throw new FlinkException(
                    "Long.MIN_VALUE timestamp: Elements used in "
                            + "interval stream joins need to have timestamps meaningful timestamps.");
        }

        /**
         * private boolean isLate(long timestamp) {
         *         long currentWatermark = internalTimerService.currentWatermark();
         *         return currentWatermark != Long.MIN_VALUE && timestamp < currentWatermark;
         * }
         * 比较当前元素和currentWatermark的大小，如果timestamp < currentWatermark，则视为迟到元素，直接return
         */
        if (isLate(ourTimestamp)) {
            return;
        }

        //添加到自己一侧的buffer中
        addToBuffer(ourBuffer, ourValue, ourTimestamp);

        /**
         * 去另一侧的buffer中遍历，如果在时间范围内则发送到下游
         */
        for (Map.Entry<Long, List<BufferEntry<OTHER>>> bucket : otherBuffer.entries()) {
            final long timestamp = bucket.getKey();

            if (timestamp < ourTimestamp + relativeLowerBound
                    || timestamp > ourTimestamp + relativeUpperBound) {
                continue;
            }

            for (BufferEntry<OTHER> entry : bucket.getValue()) {
                if (isLeft) {
                    collect((T1) ourValue, (T2) entry.element, ourTimestamp, timestamp);
                } else {
                    collect((T1) entry.element, (T2) ourValue, timestamp, ourTimestamp);
                }
            }
        }

        /**
         * 为当前元素注册一个cleanupTime清理时间，cleanupTime=当前元素时间戳+relativeUpperBound
         */
        long cleanupTime =
                (relativeUpperBound > 0L) ? ourTimestamp + relativeUpperBound : ourTimestamp;
        if (isLeft) {
            internalTimerService.registerEventTimeTimer(CLEANUP_NAMESPACE_LEFT, cleanupTime);
        } else {
            internalTimerService.registerEventTimeTimer(CLEANUP_NAMESPACE_RIGHT, cleanupTime);
        }
    }
```
前面已经在processElement方法中注册了cleanupTime，在onEventTime方法中执行具体的缓存清理工作,缓存是MapState，key为时间戳，value为List<BufferEntry<T1>>（具有相同时间戳的元素集合）
>private transient MapState<Long, List<BufferEntry<T1>>> leftBuffer;
    private transient MapState<Long, List<BufferEntry<T2>>> rightBuffer;

清理是按照时间清理的，一次清理会移除一个list（包含多个元素）
```java
 @Override
    public void onEventTime(InternalTimer<K, String> timer) throws Exception {

        long timerTimestamp = timer.getTimestamp();
        String namespace = timer.getNamespace();

        logger.trace("onEventTime @ {}", timerTimestamp);

        switch (namespace) {
            case CLEANUP_NAMESPACE_LEFT:
                {
                    long timestamp =
                            (upperBound <= 0L) ? timerTimestamp : timerTimestamp - upperBound;
                    logger.trace("Removing from left buffer @ {}", timestamp);
                    leftBuffer.remove(timestamp);
                    break;
                }
            case CLEANUP_NAMESPACE_RIGHT:
                {
                    long timestamp =
                            (lowerBound <= 0L) ? timerTimestamp + lowerBound : timerTimestamp;
                    logger.trace("Removing from right buffer @ {}", timestamp);
                    rightBuffer.remove(timestamp);
                    break;
                }
            default:
                throw new RuntimeException("Invalid namespace " + namespace);
        }
    }
```


