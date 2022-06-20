The ProcessFunction
The ProcessFunction is a low-level stream processing operation, giving access to the basic building blocks of all (acyclic) streaming applications:

events (stream elements)
state (fault-tolerant, consistent, only on keyed stream)
timers (event time and processing time, only on keyed stream)
The ProcessFunction can be thought of as a FlatMapFunction with access to keyed state and timers. It handles events by being invoked for each event received in the input stream(s).

For fault-tolerant state, the ProcessFunction gives access to Flink’s keyed state, accessible via the RuntimeContext, similar to the way other stateful functions can access keyed state.

The timers allow applications to react to changes in processing time and in event time. Every call to the function processElement(...) gets a Context object which gives access to the element’s event time timestamp, and to the TimerService. The TimerService can be used to register callbacks for future event-/processing-time instants. When a timer’s particular time is reached, the onTimer(...) method is called. During that call, all states are again scoped to the key with which the timer was created, allowing timers to manipulate keyed state.
