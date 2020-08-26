# MySQL 的主从复制
MySQL 的主从复制有三种类型
- 同步复制：主节点执行完 SQL ，需要同步给所有从库，等待所有从库返回确认，才commit
- 半同步复制：主节点执行完 SQL，主动将 bin log 发给从库，等待至少一台从库确认，才 commit；当在一定时间内超时，没有任意一台从库回复，则退为异步复制模式
- 异步复制：主节点执行完 SQL，不会主动将 bin log 发给从库，直接 commit

## 实现方式
MySQL 复制是通过传输 bin log 来实现的。
从库与主库建立连接后，主节点会开启一条线程，用于将 bin log 发给从库；
从库会开启两条线程，一条线程用于接受主库传输过来的 bin log，写到本地的 relay log 中；另外一条线程负责读取 relay log 的内容，解析称具体的操作执行（在 MySQL 5.6 前，执行 bin log 是单线程的；5.6 后开始支持多线程执行）

bin log 存储格式分为三种：
- RDR (ROW BASE REPLICATION)：RDB 是指bin log 存储的页面数据，将每次 SQL 修改影响的页面数据存储进 bin log，复制的时候按照同样的顺序页面数据写在从库
- SBR(SQL BASE REPLICATION )：SBR 是指bin log 存储的是 sql，将每条执行的 sql 按序保存到 bin log 中，从库按顺序执行每条 SQL 来完成复制
- MBR(MIX BASE REPLICATION)：MBR 是指混合存储模式，以 SBR 为主，RDR 为辅

优缺点：
1. SBR 基于 SQL 存储，日志大小比较小，而 RDR 基于页面数据存储，日志会比较大，前者在传输中会节约 I/O、带宽
2. SBR 基于 SQL，对于一些 now() 函数同步到从库，因为存在延时而出现不一致；或者会出现触发器的调用、存储过程无法被正确的复制。而 RDR 基于页面数据改变来存储，可以避免这种问题
3. MBR 是两种方式的折中，对于无法用 SBR 复制的方式的操作，会采用 RBR 来存储，推荐方式使用 MBR。


额外延伸，MySQL 的复制实现
基于 GTID 复制的工作原理
1. 主节点更新数据时，会在事务前产生 GTID，一起记到 bin log 中
2. 从节点拉到变更的 bin log，写入到 relay log
3. SQL 线程从 relay log 获取 GTID，对比本地binlog 是否有记录
4. 如果有记录，说明该 GTID 的事务已经执行了，从节点就会忽略该操作
5. 如果没有记录，从节点就会执行该 GTID 的事务，并记录到 bin log 中
6. 在解析的过程会判断是否有主键，如果没有就用二级索引，如果有就用全表扫描