# 分布式一致性协议
用于约束分布式系统数据一致性的约束，保证各节点的数据一致性。

协议关注的问题：
1. 数据写入时，如何保证节点数据一致
2. 节点故障时，如何保证节点数据一致


常见的一致性协议
- 单主协议：2pc、3pc(CA模型)；Paxos、RAFT、ZAB（CP模型，分区容忍性一致性协议）
分布式系统作为一个单体系统，所有写操作都由主节点处理，主节点处理后同步给从节点

- 多主协议：Gossip、POW
写操作可以由不同的主节点发起，然后同步给其他副本
议

# RAFT 算法
简单易懂的文章：https://www.cnblogs.com/mindwind/p/5231986.html
**基本概念**
- 状态机：当前数据的状态
- Log：所有的修改，可用于故障恢复与数据复制
- 一致性模块：使用一致性算法保证写入 Log 的操作的一致性

raft 的两个核心模块
选主协议、数据一致协议

## 选主协议
raft 属于单主协议，即在整个分布式系统中，只允许存在一个 leader，所有的写操作都由 leader 完成。

在集群中，raft node 有三种状态
- leader：主节点，写请求都由主节点执行，主节点完成后同步给从节点
- follower：从节点，读请求可以直接执行，写请求需要转发给主节点执行
- candidate：从节点收不到主节点的心跳，则认为主节点故障，则变为该状态，直到选主结束

**选主流程**
>所有节点启动时都是 follower 状态；在一个时间内如果没有收到leader 的心跳，则切换到 candidate 状态，发起选举。如果收到半数以上的票数，则切换成 leader 状态；如果有自己更新的节点，则主动切换到 follower
raft 将时间分成一段一段 term，在每一个 term 开始时都会发起选举，当选举成功，则 leader 在 term 期间任期，直到本次 term 结束

**发起选举**
>触发机制：当从节点在 election timeout 丢失主节点的心跳，则会主动发起选举
>1. current term 增加，切换到 candidate 状态
>2. 投自己一票
>3. 给其他节点发送 requestVote RPC
>4. 等待其他节点回复
	1) 收到 majority 的投票，则成为 leader，并广播给其他节点
	2) 已经有节点当选，则主动成为 follower
	3) 等待一个随机选举超时的时间后没有收到 majority 投票，则保持 candidate 状态，重新发出选举
	
**投票**
>约束项：
>- 在每一个 term，每个节点最多只能投一票
>- 候选节点的数据不能比我旧
>- 先来先得，谁先发起选举，则先选谁

## 数据复制 
**log replication**
leader 执行操作时，将操作写到 log 里面，leader 将 log 同步给 follower，follower 按照相同的顺序执行，保证数据状态一致

leader 执行请求的过程：
1. 追加一个 log entry 到 log 中
2. 并发地将该 log entry 通过 AppendEntries RPC 发送给 follower
3. 等待大多数节点的响应，并将 log entry 提交（如果此时前面的log entry 为被提交，则会顺带一起提交，在主 crash 的场景下，可能会存在ji）
4. 将节点 apply 到状态机中
5. 回复客户端 ok
6. 通知 follower apply log entry

每个 log entry 除了存储命令，还存储了 leader 的 term，当 log entry 复制到大多数节点后，则为 commit 状态；当 log entry 被 apply 到状态机，则为 apply 状态

**safety**
raft 模型保证以下属性
- election safety：在一个 term 最多只有一个 leader
>系统同时出现多个 leader，称为脑裂
raft 通过一下机制保证只有一个leader：
1. 一个节点在一个任期内最多只能投一次票
2. 只有获得 majority 投票的节点才会成为 leader

- leader-append-only：leader 不会重写或者删除 log，只允许追加
- log matching：如果两个节点的日志存在相同 index 和 term 的 entry，则该 entry 前面的 entry 也一样

![Alt text](./1593776639541.png)
>在正常情况下，log matching 很容满足，但是当出现节点掉线的时候，则可能会出现不一致
在故障情况中，leader 和 follower 维护的日志可能会出现一下情况
- 比leader 日志少
- 比leader 日志多
当 leader 和 follwer 不一致时，leader 强制 follwer 复制自己的 log

> leader 维护一个 nextIndex[] 数组，记录leader 可以发送给每一个 follower 的 log index，初始值为 leader 当前的 log index + 1。当 leader 选取成功后，leader 会立刻向所有 follower 发送 AppendEntries RPC（不包含 Log Entry，相当于 ping，用于广播leader 已经选出），然后开始进行同步，流程为每个 follower 找到一个 index 与 leader 相同 term 的 log entry，则该位置开始追加后面的 log entry，流程：
> 1. leader  初始化 nextIndex[x]（x 表示第几个follower），每个初始值为当前的 log index + 1
> 2. 发送 appendEntries RPC  给从节点，发送 log[nextIndex[x] - 1] 的 preTerm 和 preIndex
> 3. 如果 follower 判断当前的 preIndex 位置的 term 不等于 preTerm，则返回 false；否则返回 true
> 4. leader 收到回复，如果是 fasle ，则将 nextIndex[x] - = 1，跳转到 2，否则将从当前 nextIndex 位置开始追加后面的 log entries

- leader completeness：如果一个日志项在某个任期被提交了，那么在任期更高的 leader 的日志里一定包含这个日志条目
> 这是由两个特性来保证的
> - 一个日志被复制到 majority 节点才可以 commit
> 一个节点收到 majority 的投票才能成为 leader；节点 A 给 节点 B投票的前提是，B的日志不能比 A 的旧
>
>判断日志的新旧：
>- 如果日志比候选人的日志要更新，则拒绝投票
>- 如果日志中的最后一个 index 项的 term 不一样，则term 越大的越新；如果最后一个 index 项的 term 一样，则log 越长的越新

- state machine safety：如果一个节点已经 apply 了一个指定 index 的日志项给状态机，那么不会有另外的节点在该 index 上 apply 不同的日志项

## 常见问题
### 脑裂问题
引入 region leader 的概念，对于每个分区来说，任意时刻只有一个 region leader，所有读写请求通过 region leader 完成，region leader 会将请求转发给当前的 raft leader，当网络出现分区时，会出现以下几种情况：
- region leader 落在多数派，老 raft leader 在多数派这边
> region leader 的 lease 不会过期，因为 region leader 的心跳仍然能更新到多数派的节点上，老的 raft leader 仍然能同步到大多数节点上，少数派这边也不会选举出新的 leader， 这种情况下不会出现 stale read。

- region leader 落在多数派，老 raft leader 在少数派这边
> 老的 raft leader 被分到了少数派这边，多数派这边选举出了新的 raft leader ，如果此时的 region leader 在多数派这边
> 因为所有的读写请求都会找到 region leader 进行，即使在原来没有出现网络分区的情况下，客户端的请求也都是要走 node 1 ，经由 node 1 转发给 node 5，客户端不会直接访问 node 5，所以此时即使网络出现分区，新 leader 也正好在多数派这边，读写直接就打到 node 1 上，皆大欢喜，没有 stale read。

- region leader 落在少数派，老 raft leader 在多数派这边
> region leader 落在少数派这边，老 raft leader 在多数派这边，这种情况客户端的请求找到 region leader，他发现的无法联系到 leader（因为在少数派这边没有办法选举出新的 leader），请求会失败，直到本次 region leader 的 lease 过期，同时新的 region leader 会在多数派那边产生（因为新的 region leader 需要尝试走一遍 raft 流程）。因为老的 region leader 没办法成功的写入，所以也不会出现 stale read。但是付出的代价是在 region leader lease 期间的系统的可用性。

- region leader 落在少数派，老 raft leader 在少数派这边
> 第四种情况和第三种情况类似，多数派这边会产生新的 raft leader 和 region leader。
