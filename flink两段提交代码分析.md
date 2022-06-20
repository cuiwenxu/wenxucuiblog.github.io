若要sink支持 exactly-once semantics，它必须以事务的方式写数据，这样当提交事务时==两次checkpoint间的所有写入操作当作为一个事务被提交==。这确保了出现故障或崩溃时这些写入操作能够被回滚。
