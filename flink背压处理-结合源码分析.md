@[TOC](flink背压处理)
# 基本原理
流处理框架需要考虑的一个重要问题是协调上下游算子处理速率的问题。比如A->B的一个处理链，B处理速率变慢时，A需要及时作出响应（减慢发送或停发），否则会耗尽内存导致任务崩溃。

那flink是如何处理背压的呢？flink处理反压可以分为两种情况：
- A、
A->B
# 重要代码分析


未完待续。。。。
