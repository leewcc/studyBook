# innodb_flush_log_at_trx_commit 参数
该参数控制事务日志的持久化策略，有3个取值，分别是

- 0：由 MySQL 的 main thread 每秒将 log buffer 的 redo log 日志写入到 log file，并调用 sync 同步到磁盘
- 1：每次提交事务时，将 log buffer 的 redo log 写入到 log file 中，并调用 sync 同步到磁盘
- 2：每次提交事务时，将 log buffer 的 redo log 写入到 log file 中，由存储引擎的 main thread 每秒将日志刷到磁盘


# sync_binlog
控制 binlog 的持久化策略，由三个取值，分别是
- 0：引擎不进行 binlog的持久化，而是由操作系统的文件系统控制刷新
- 1：每提交一次事务，则引擎调用 sync 将日志刷新到磁盘
- n：当提交的日志组达到 n，则引擎调用 sync 将日志刷新到磁盘