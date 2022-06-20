__==StreamOperator相当于function的一个封装，起到类似于config文件的作用，flink taskmanager所管理的线程中真正运行的类是StreamTask==__
![在这里插入图片描述](https://img-blog.csdnimg.cn/2020072121550511.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)


##### StreamTask与StreamOperator
前面提到，Task对象在执行过程中，把执行的任务交给了StreamTask这个类去执行。在我们的wordcount例子中，实际初始化的是OneInputStreamTask的对象（参考上面的类图）。那么这个对象是如何执行用户的代码的呢？

复制代码
    protected void run() throws Exception {
        // cache processor reference on the stack, to make the code more JIT friendly
        final StreamInputProcessor<IN> inputProcessor = this.inputProcessor;

        while (running && inputProcessor.processInput()) {
            // all the work happens in the "processInput" method
        }
    }
复制代码
 

它做的，就是把任务直接交给了InputProcessor去执行processInput方法。这是一个StreamInputProcessor的实例，该processor的任务就是处理输入的数据，包括用户数据、watermark和checkpoint数据等。我们先来看看这个processor是如何产生的：

复制代码
    public void init() throws Exception {
        StreamConfig configuration = getConfiguration();

        TypeSerializer<IN> inSerializer = configuration.getTypeSerializerIn1(getUserCodeClassLoader());
        int numberOfInputs = configuration.getNumberOfInputs();

        if (numberOfInputs > 0) {
            InputGate[] inputGates = getEnvironment().getAllInputGates();

            inputProcessor = new StreamInputProcessor<>(
                    inputGates,
                    inSerializer,
                    this,
                    configuration.getCheckpointMode(),
                    getCheckpointLock(),
                    getEnvironment().getIOManager(),
                    getEnvironment().getTaskManagerInfo().getConfiguration(),
                    getStreamStatusMaintainer(),
                    this.headOperator);

            // make sure that stream tasks report their I/O statistics
            inputProcessor.setMetricGroup(getEnvironment().getMetricGroup().getIOMetricGroup());
        }
    }
复制代码
 

这是OneInputStreamTask的init方法，从configs里面获取StreamOperator信息，生成自己的inputProcessor。那么inputProcessor是如何处理数据的呢？我们接着跟进源码：

复制代码
public boolean processInput() throws Exception {
        if (isFinished) {
            return false;
        }
        if (numRecordsIn == null) {
            numRecordsIn = ((OperatorMetricGroup) streamOperator.getMetricGroup()).getIOMetricGroup().getNumRecordsInCounter();
        }

        //这个while是用来处理单个元素的（不要想当然以为是循环处理元素的）
        while (true) {
            //注意 1在下面
            //2.接下来，会利用这个反序列化器得到下一个数据记录，并进行解析（是用户数据还是watermark等等），然后进行对应的操作
            if (currentRecordDeserializer != null) {
                DeserializationResult result = currentRecordDeserializer.getNextRecord(deserializationDelegate);

                if (result.isBufferConsumed()) {
                    currentRecordDeserializer.getCurrentBuffer().recycle();
                    currentRecordDeserializer = null;
                }

                if (result.isFullRecord()) {
                    StreamElement recordOrMark = deserializationDelegate.getInstance();

                    //如果元素是watermark，就准备更新当前channel的watermark值（并不是简单赋值，因为有乱序存在），
                    if (recordOrMark.isWatermark()) {
                        // handle watermark
                        statusWatermarkValve.inputWatermark(recordOrMark.asWatermark(), currentChannel);
                        continue;
                    } else if (recordOrMark.isStreamStatus()) {
                    //如果元素是status，就进行相应处理。可以看作是一个flag，标志着当前stream接下来即将没有元素输入（idle），或者当前即将由空闲状态转为有元素状态（active）。同时，StreamStatus还对如何处理watermark有影响。通过发送status，上游的operator可以很方便的通知下游当前的数据流的状态。
                        // handle stream status
                        statusWatermarkValve.inputStreamStatus(recordOrMark.asStreamStatus(), currentChannel);
                        continue;
                    } else if (recordOrMark.isLatencyMarker()) {
                    //LatencyMarker是用来衡量代码执行时间的。在Source处创建，携带创建时的时间戳，流到Sink时就可以知道经过了多长时间
                        // handle latency marker
                        synchronized (lock) {
                            streamOperator.processLatencyMarker(recordOrMark.asLatencyMarker());
                        }
                        continue;
                    } else {
                    //这里就是真正的，用户的代码即将被执行的地方。从章节1到这里足足用了三万字，有点万里长征的感觉
                        // now we can do the actual processing
                        StreamRecord<IN> record = recordOrMark.asRecord();
                        synchronized (lock) {
                            numRecordsIn.inc();
                            streamOperator.setKeyContextElement1(record);
                            streamOperator.processElement(record);
                        }
                        return true;
                    }
                }
            }

            //1.程序首先获取下一个buffer
            //这一段代码是服务于flink的FaultTorrent机制的，后面我会讲到，这里只需理解到它会尝试获取buffer，然后赋值给当前的反序列化器
            final BufferOrEvent bufferOrEvent = barrierHandler.getNextNonBlocked();
            if (bufferOrEvent != null) {
                if (bufferOrEvent.isBuffer()) {
                    currentChannel = bufferOrEvent.getChannelIndex();
                    currentRecordDeserializer = recordDeserializers[currentChannel];
                    currentRecordDeserializer.setNextBuffer(bufferOrEvent.getBuffer());
                }
                else {
                    // Event received
                    final AbstractEvent event = bufferOrEvent.getEvent();
                    if (event.getClass() != EndOfPartitionEvent.class) {
                        throw new IOException("Unexpected event: " + event);
                    }
                }
            }
            else {
                isFinished = true;
                if (!barrierHandler.isEmpty()) {
                    throw new IllegalStateException("Trailing data in checkpoint barrier handler.");
                }
                return false;
            }
        }
    }
