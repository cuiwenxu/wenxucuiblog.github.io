flink应用可以按以下三种方式部署
* session模式 
* per-job模式
* Application模式

##### session模式
会话模式假定已有在运行的集群，并使用该集群的资源来执行任何提交的应用程序。 在同一（会话）集群中执行的应用程序会使用，因此竞争
Session mode assumes an already running cluster and uses the resources of that cluster to execute any submitted application. Applications executed in the same (session) cluster use, and consequently compete for, the same resources. This has the advantage that you do not pay the resource overhead of spinning up a full cluster for every submitted job. But, if one of the jobs misbehaves or brings down a Task Manager, then all jobs running on that Task Manager will be affected by the failure. This, apart from a negative impact on the job that caused the failure, implies a potential massive recovery process with all the restarting jobs accessing the filesystem concurrently and making it unavailable to other services. Additionally, having a single cluster running multiple jobs implies more load for the JobManager, who is responsible for the book-keeping of all the jobs in the cluster.


