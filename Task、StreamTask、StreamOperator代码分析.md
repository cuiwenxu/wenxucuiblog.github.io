Task代表在TaskManager上执行并行子任务。Task封装Flink运算符（可能是用户函数）并运行它，提供所有必要的服务，例如消耗输入数据，产生结果（中间结果分区） 并与JobManager通信。
Flink operator（AbstractInvokable的子类）仅具有数据读取，写入和某些事件回调。而Task则负责将operator连接到network stack和actor message，并跟踪operator的执行状态并处理异常。Task不知道它们与其他Task的关系，或者它们是执行任务的第一次尝试还是重复尝试。 所有这些仅JobManager知道。 所有Task知道的都是其自己的可运行代码，任务的配置以及要使用和产生的中间结果的ID（如果有）

