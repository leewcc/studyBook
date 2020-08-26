# redis 的持久化机制
redis 是内存数据库，数据都是存储在内存中，为了避免数据丢失，需要将数据持久到硬盘上，下次 redis 重启时，可以进行恢复

redis 持久化分为 RDB 持久化和 AOF 持久化。RDB 持久化是将当前快照数据保存到硬盘，AOF 持久化是将每次执行的写命令保存到磁盘（类似于 MySQL 的 bin log）

## RDB 持久化
快照持久化，因为只是保存触发的快照，因为会存在数据丢失的风险。可通过手动触发或者自动触发


**手动触发**
调用 save 、bgsave 命令都可以生成 RDB 文件

1. save 命令会阻塞 Redis 服务器主进程，导致无法执行其他命令，直到 RDB 文件创建完
2. bgsave 命令会 fork 一个子进程，由子进程负责创建 RDB 文件，父进程可以继续处理请求（fork 的时候会阻塞主进程）

**自动触发**
>在配置文件中配置 save m n
save m n：当 m 秒发生 n 次写变化，就会触发 bgsave
实现原理：
redis 有一个周期函数，每隔 100ms 执行一次， 检查是否持久化是通过 dirty 计数器和 lastsave 时间戳实现的。每执行完一次 save/bgsave，dirty 计数器都会重置，每有一次写变化，则dirty 都会增加。lastsave 记为上次 save/bgsave 的时间,如果当前时间和计数器命中 m n 规则则会触发 bgsave

bgsave 的执行流程
![Alt text](./1593689548970.png)


## AOF 持久化
将 redis 每次执行的写命令记录到单独的日志文件中，该种方式的实时持久效果更好，目前该机制为主流的持久化方案，不过 Redis 服务器默认开启 RDB，关闭 AOF 的

AOF 开启后的执行流程：
1. 命令追加：redis 的写命令都会追加到缓冲区的 aof_buf
2. 文件写入(write) 和文件同步(sync)：根据不同的同步策略将 aof_buf 种的内容同步到硬盘
3. 文件重写：定期重写 AOF 文件，进行压缩

**命令追加**
Redis先将写命令追加到缓冲区，而不是直接写入文件，主要是为了避免每次有写命令都直接写入硬盘，导致硬盘IO成为Redis负载的瓶颈

**文件写入和文件同步**
当用户调用write函数将数据写入文件时，操作系统通常会将数据暂存到一个内存缓冲区里，当缓冲区被填满或超过了指定时限后，才真正将缓冲区的数据写入到硬盘里。这样的操作虽然提高了效率，但也带来了安全问题：如果计算机停机，内存缓冲区中的数据会丢失；因此系统同时提供了fsync、fdatasync等同步函数，可以强制操作系统立刻将缓冲区中的数据写入到硬盘里，从而确保数据的安全性。

AOF缓存区的同步文件策略由参数appendfsync控制，各个值的含义如下：
- always：命令写入aof_buf后立即调用系统fsync操作同步到AOF文件，fsync完成后线程返回。这种情况下，每次有写命令都要同步到AOF文件，硬盘IO成为性能瓶颈，Redis只能支持大约几百TPS写入，严重降低了Redis的性能；即便是使用固态硬盘（SSD），每秒大约也只能处理几万个命令，而且会大大降低SSD的寿命。
- no：命令写入aof_buf后调用系统write操作，不对AOF文件做fsync同步；同步由操作系统负责，通常同步周期为30秒。这种情况下，文件同步的时间不可控，且缓冲区中堆积的数据会很多，数据安全性无法保证。
- everysec：命令写入aof_buf后调用系统write操作，write完成后线程返回；fsync同步文件操作由专门的线程每秒调用一次。everysec是前述两种策略的折中，是性能和数据安全性的平衡，因此是Redis的默认配置，也是推荐的配置。

**文件重写**
文件重写是指定期重写AOF文件，减小AOF文件的体积。需要注意的是，AOF重写是把Redis进程内的数据转化为写命令，同步到新的AOF文件；不会对旧的AOF文件进行任何读取、写入操作!

触发机制：手动触发和自动触发
手动触发通过调用 bgrewriteaof 命令

自动触发：根据auto-aof-rewrite-min-size和auto-aof-rewrite-percentage参数，以及aof_current_size（当前文件大小）和aof_base_size（上一次重写文件的大小）状态确定触发时机。

- auto-aof-rewrite-min-size：执行AOF重写时，文件的最小体积，默认值为64MB。
- auto-aof-rewrite-percentage：执行AOF重写时，当前AOF大小(即aof_current_size)和上一次重写时AOF大小(aof_base_size)的比值。

文件重写的流程：
![Alt text](./1593690673054.png)
注：重写期间Redis执行的写命令，需要追加到新的AOF文件中，为此Redis引入了aof_rewrite_buf缓存。

1) Redis父进程首先判断当前是否存在正在执行 bgsave/bgrewriteaof的子进程，如果存在则bgrewriteaof命令直接返回，如果存在bgsave命令则等bgsave执行完成后再执行。前面曾介绍过，这个主要是基于性能方面的考虑。

2) 父进程执行fork操作创建子进程，这个过程中父进程是阻塞的。

3.1) 父进程fork后，bgrewriteaof命令返回”Background append only file rewrite started”信息并不再阻塞父进程，并可以响应其他命令。Redis的所有写命令依然写入AOF缓冲区，并根据appendfsync策略同步到硬盘，保证原有AOF机制的正确。

3.2) 由于fork操作使用写时复制技术，子进程只能共享fork操作时的内存数据。由于父进程依然在响应命令，因此Redis使用AOF重写缓冲区(图中的aof_rewrite_buf)保存这部分数据，防止新AOF文件生成期间丢失这部分数据。也就是说，bgrewriteaof执行期间，Redis的写命令同时追加到aof_buf和aof_rewirte_buf两个缓冲区。

4) 子进程根据内存快照，按照命令合并规则写入到新的AOF文件。

5.1) 子进程写完新的AOF文件后，向父进程发信号，父进程更新统计信息，具体可以通过info persistence查看。

5.2) 父进程把AOF重写缓冲区的数据写入到新的AOF文件，这样就保证了新AOF文件所保存的数据库状态和服务器当前状态一致。

5.3) 使用新的AOF文件替换老文件，完成AOF重写。

## RDB和AOF的优缺点
RDB和AOF各有优缺点：

RDB持久化

优点：RDB文件紧凑，体积小，网络传输快，适合全量复制；恢复速度比AOF快很多。当然，与AOF相比，RDB最重要的优点之一是对性能的影响相对较小。

缺点：RDB文件的致命缺点在于其数据快照的持久化方式决定了必然做不到实时持久化，而在数据越来越重要的今天，数据的大量丢失很多时候是无法接受的，因此AOF持久化成为主流。此外，RDB文件需要满足特定格式，兼容性差（如老版本的Redis不兼容新版本的RDB文件）。

AOF持久化

与RDB持久化相对应，AOF的优点在于支持秒级持久化、兼容性好，缺点是文件大、恢复速度慢、对性能影响大。