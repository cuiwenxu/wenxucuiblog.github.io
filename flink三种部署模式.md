##### 部署模式
flink可以通过以下三种模式部署：

- Session Mode
- Per-Job Mode
- Application Mode

以上方式主要区别在于
- 集群生命周期和资源隔离保证
- 应用程序的main方法是在客户端还是在集群上执行。

##### Session Mode(会话模式)
会话模式假定存在已经在运行的集群，并使用该集群的资源来执行任何提交的应用程序。在同一（会话）集群中执行的应用程序使用并因此竞争相同的资源。这样做的好处是，您不必为每个提交的作业都创建集群。但是，如果其中一项作业行为不当或关闭了taskmanager，则该taskmanager上运行的所有作业都会受到故障的影响。除了对导致故障的作业产生负面影响外，这还意味着潜在的大规模恢复过程。此外，只有一个集群运行多个作业意味着JobManager的负载增加，JobManager负责簿记集群中所有作业。

##### Per-Job Mode
为了提供更好的资源隔离保证，Per-Job模式使用可用的集群管理器框架（例如YARN，Kubernetes）为每个提交的作业启动集群。该群集仅适用于该作业。作业完成后，集群将被torn down，所有附属的资源（文件等）将被清除。这提供了更好的资源隔离，因为行为不当的工作只能拖垮其自己的taskmanager。另外，由于每个作业有一个，因此它可以将簿记工作分散到多个JobManager中。== 由于这些原因，由于许多生产原因，首选工作资源分配模型是首选模式。==

##### Application Mode
在上述所有模式下，应用程序的main方法都是在客户端执行的。==此过程包括本地下载应用程序的依赖项，执行main提取Flink运行时可以理解的应用程序表示形式（即JobGraph），并将依赖项和JobGraph运送到集群中。这使客户端成为大量的资源消耗者，因为它可能需要大量的网络带宽来下载依赖项并将二进制文件运送到群集，并且需要CPU周期来执行main。当跨用户共享客户端时，此问题可能更加明显。==
基于此，“应用程序模式”将为每个提交的应用程序创建一个集群，但是这次，该应用程序的main方法在JobManager上执行。每个应用程序创建集群可以看作是创建仅在特定应用程序的作业之间共享的会话集群，并且在应用程序完成时被拆除。通过这种体系结构，应用程序模式可以提供与逐作业模式相同的资源隔离和负载平衡保证，但要保证整个应用程序的粒度。在JobManager上执行main可以节省所需的CPU周期，还可以节省本地下载依赖项所需的带宽。此外，由于每个应用程序只有一个JobManager，因此它可以更均匀地分散下载群集中应用程序的依赖项的网络负载。

注意：__在应用程序模式下，main在集群JobManager上而不是在客户端上执行。这可能会对您的代码产生影响，例如，您必须使用应用程序的JobManager访问使用registerCachedFile在环境中注册的任何路径。__
与逐作业模式相比，应用程序模式允许提交包含多个作业的应用程序。作业执行的顺序不受部署模式的影响，但受启动作业的调用的影响。使用被阻塞的execute可以建立一个命令，这将导致“下一个”作业的执行被推迟到“该”作业完成为止。使用非阻塞的executeAsync（）会导致“下一个”作业在“此”作业完成之前开始
##### 总结
在会话模式下，集群生命周期与集群上运行的任何作业的生命周期无关，并且资源在所有作业之间共享。 “逐个作业”模式为每个提交的作业分配一个集群，需要更高的成本，但这具有更好的隔离保证，因为资源不会在各个作业之间共享。 在这种情况下，集群的生命周期将与作业的生命周期绑定在一起。 最后，应用程序模式会为每个应用程序创建一个会话集群，并在该集群上执行应用程序的main方法。

最后附上这三种模式对应的cli
- Session mode
>./flink run -c com.aibee.mall.task.DailyTimeSliceActionPersonCountStatisticV2 -m yarn-cluster /home/wxcui/test/RealTimeProject-1.1.0-jar-with-dependencies.jar --brand dyc --flag dev --job_name dyc_DailyActionPersonCountSliceStatistic_dyc_test --kafka_topic mall_dyc_person_count --group_id wxcui001 --start_offset timestamp --offset_timestamp 1595174400000 --security.protocol PLAINTEXT --parallelism 8 --enable_checkpoint true
- Per-Job Mode (==在-m yarn-cluster前添加-yd(detach mode)即可==)
> ./flink run -c com.aibee.mall.task.DailyTimeSliceActionPersonCountStatisticV2 -yd -m yarn-cluster /home/wxcui/test/RealTimeProject-1.1.0-jar-with-dependencies.jar --brand dyc --flag dev --job_name dyc_DailyActionPersonCountSliceStatistic_dyc_test --kafka_topic mall_dyc_person_count --group_id wxcui001 --start_offset timestamp --offset_timestamp 1595174400000 --security.protocol PLAINTEXT --parallelism 8 --enable_checkpoint true
- Application Mode
> ./bin/flink run-application -t yarn-application ./examples/batch/WordCount.jar
