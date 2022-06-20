为什么小表不走BHJ，而是走的SMJ？

A：因为是否小表走广播是根据spark对元数据的估算得到的，如果join表有很多的filter容易把表估大，造成本可以广播的情况实际没有广播

估算表大小的细节
def canBroadcastBySize(plan: LogicalPlan, conf: SQLConf): Boolean = {
    val autoBroadcastJoinThreshold = if (plan.stats.isRuntime) {
      conf.getConf(SQLConf.ADAPTIVE_AUTO_BROADCASTJOIN_THRESHOLD)
        .getOrElse(conf.autoBroadcastJoinThreshold)
    } else {
      conf.autoBroadcastJoinThreshold
    }
    plan.stats.sizeInBytes >= 0 && plan.stats.sizeInBytes <= autoBroadcastJoinThreshold
  }
