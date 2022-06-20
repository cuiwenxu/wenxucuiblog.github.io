

@[TOC](将用户代码的classloader与TaskExecutor上Job的生存周期绑定)

# 问题描述
Flink任务频繁重启时很容易报出metaspace OOM，经过查找发现Flink不会在不同的作业提交中重用类加载器。如果您提交一个作业(即使它是与前一个作业相同的用户代码)，那么Flink将把它视为一个新作业，并为其创建一个新的类装入器。换句话说，如果您的用户代码中有类泄漏，或者任务频繁重启，那么您可以通过多次向Flink集群提交相同的作业来关闭Flink集群，这最终会耗尽元空间。
Flink在1.11.0修复了这个问题，下面为jira issue翻译和pr的代码分析。
# jira issue&pr
https://issues.apache.org/jira/browse/FLINK-16408
https://github.com/apache/flink/pull/11963

# 解决方案
This PR binds the user code class loader to the lifetime of a JobTable.Job on the TaskExecutor. This means that the TaskExecutor will not release the user code class loader as long as it contains an allocated slot for the respective job. This will ensure that a TaskExecutor can reuse the user code class loader across failovers (task and JM failovers). By reusing user code class loaders Flink will avoid to reload classes and, thus, decrease the pressure it puts on the JVM's metaspace. This will significantly improve situations where a class leak exists because Flink won't deplete the JVM's metaspace under recoveries.

A side effect is that static class fields won't be reset in case of failovers. This can be considered as behaviour changing.


这个PR将用户代码类装入器绑定到JobTable的生命周期。TaskExecutor上的作业。这意味着TaskExecutor不会释放用户代码类装入器，只要它包含为各自的作业分配的槽。这将确保TaskExecutor可以跨故障转移(任务和JM故障转移)重用用户代码类加载器。通过重用用户代码类加载器，Flink将避免重新加载类，从而减少对JVM元空间的压力。这将显著改善存在类泄漏的情况，因为在恢复时Flink不会耗尽JVM的元空间。

一个副作用是静态类字段在故障转移时不会被重置。这可以被认为是行为的改变。
