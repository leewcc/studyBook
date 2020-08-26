# redis cluster 的机制
redis cluster 与 sharding 不一样，内部真正做到数据分区，而不需要依赖外部的一致性哈希算法

## **数据分区机制**
redis cluster 有多个主节点，每个主节点又挂着多个从节点。它先分配 16384 个虚拟槽，并将槽分配给不同的主节点。

**数据存储结构**
redis 用 clusterNode 和 clusterState 维护集群状态，前者记录了一个节点的状态，后者记录了集群整体的状态

``` java
class ClusterNode {
	long createTime;  //创建时间
	String nodeId;    // 节点id
	String ip;
	int port;
	int flag;   通过 flag 的位来决定节点的主从状态、是否在线、是否在握手
	long configEpoch;
	char[16384/8] slots; // 标识槽在该节点的分布，每个 bit 对应一个槽，值为1代表存在
	int numSlots; // 节点中槽的数量
}


class ClusterState {
	ClusterNode myself; 
	long currentEpoch; // 当前的配置版本时间
	int state; // 在线还是下线
	int size; 
	Map<String, ClusterNode> dict // 节点名称 → clusterNode
	Map<int, ClusterNode> slots;  // slot → clusterNode

```

客户端访问集群的过程：
1. 先通过 CRC16(key) & 16383 计算 key 属于哪个槽 i
2. 判断 clusterState.slots[i] == clusterState.myself，如果是，则槽在当前节点，可以直接在当前节点执行命令；否则查询槽对应的节点的地址，包装到 MOVE 错误返回

## **节点通信机制**
redis cluster 与哨兵不一样，它是去中心化，各个节点都存储数据，同时也都参与集群状态的维护，集群中的每个节点，都打开了两个 TCP 端口
- 普通端口：用于客户端访问，与节点间迁移数据
- 集群端口：普通端口+10000，用于节点之间的通信

节点间采用 Gossip 协议，该协议的特点是，在节点数量有限的网络中，每个节点都随机的与部分节点通信，节点收到信息后再传播给其他节点，这样最后每个节点的状态会达成一致。该协议的优点：负载低（比广播模式低）、去中心化、容错性高，缺点主要是集群的收敛速度慢

节点采用固定频率（每秒10次）的定时任务进行通信，判断是否需要发送消息及消息类型、确定接收节点、发送消息等。如果集群状态发生了变化，如增减节点、槽状态变更，通过节点间的通信，所有节点会很快得知整个集群的状态，使集群收敛。

消息类型有5种：
- MEET 消息
> 在节点握手阶段，当节点收到客户端的CLUSTER MEET命令时，会向新加入的节点发送MEET消息，请求新节点加入到当前集群；新节点收到MEET消息后会回复一个PONG消息

- PING 消息
> 集群里每个节点每秒钟会选择部分节点发送PING消息，接收者收到消息后会回复一个PONG消息。PING消息的内容是自身节点和部分其他节点的状态信息；作用是彼此交换信息，以及检测节点是否在线。PING消息使用Gossip协议发送，接收节点的选择兼顾了收敛速度和带宽成本，具体规则如下：
> 1. 随机找5个节点，在其中选择最久没有通信的1个节点
> 2. 扫描节点列表，选择最近一次收到PONG消息时间大于cluster_node_timeout/2的所有节点，防止这些节点长时间未更新。

- PONG 消息
> PONG消息封装了自身状态数据。可以分为两种：第一种是在接到MEET/PING消息后回复的PONG消息；第二种是指节点向集群广播PONG消息，这样其他节点可以获知该节点的最新信息，例如故障恢复后新的主节点会广播PONG消息

- FAIL 消息
> 当一个主节点判断另一个主节点进入FAIL状态时，会向集群广播这一FAIL消息；接收节点会将这一FAIL消息保存起来，便于后续的判断。

- PUBLISH 消息
> 节点收到PUBLISH命令后，会先执行该命令，然后向集群广播这一消息，接收节点也会执行该PUBLISH命令

## 常见问题
**集群伸缩**
运行期间对集群进行扩容缩容，其核心是槽迁移，修改槽与节点的对应关系，实现槽在节点之间的移动

addNode、assignSlot 的执行流程 
**cluster meet**
假设要向A节点发送cluster meet命令，将B节点加入到A所在的集群，则A节点收到命令后，执行的操作如下：

1)  A为B创建一个clusterNode结构，并将其添加到clusterState的nodes字典中
2)  A向B发送MEET消息
3)  B收到MEET消息后，会为A创建一个clusterNode结构，并将其添加到clusterState的nodes字典中
4)  B回复A一个PONG消息
5)  A收到B的PONG消息后，便知道B已经成功接收自己的MEET消息
6)  然后，A向B返回一个PING消息
7)  B收到A的PING消息后，便知道A已经成功接收自己的PONG消息，握手完成
8)  之后，A通过Gossip协议将B的信息广播给集群内其他节点，其他节点也会与B握手；一段时间后，集群收敛，B成为集群内的一个普通节点


**cluster addslots**
集群中槽的分配信息，存储在clusterNode的slots数组和clusterState的slots数组中，两个数组的结构前面已做介绍；二者的区别在于：前者存储的是该节点中分配了哪些槽，后者存储的是集群中所有槽分别分布在哪个节点。

cluster addslots命令接收一个槽或多个槽作为参数，例如在A节点上执行cluster addslots {0..10}命令，是将编号为0-10的槽分配给A节点，具体执行过程如下：

1)  遍历输入槽，检查它们是否都没有分配，如果有一个槽已分配，命令执行失败；方法是检查输入槽在clusterState.slots[]中对应的值是否为NULL。
2)  遍历输入槽，将其分配给节点A；方法是修改clusterNode.slots[]中对应的比特为1，以及clusterState.slots[]中对应的指针指向A节点
3)  A节点执行完成后，通过节点通信机制通知其他节点，所有节点都会知道0-10的槽分配给了A节点

**ASK**
槽迁移的过程中，会出现 ASK 错误
![Alt text](./1593783521880.png)
客户端收到ASK错误后，从中读取目标节点的地址信息，并向目标节点重新发送请求，就像收到MOVED错误时一样。但是二者有很大区别：ASK错误说明数据正在迁移，不知道何时迁移完成，因此重定向是临时的，SMART客户端（jedis）不会刷新slots缓存；MOVED错误重定向则是(相对)永久的，SMART客户端（jedis）会刷新slots缓存。

**故障转移**
集群故障转移和哨兵思路类似：通过定时任务发送 PING 消息检测其他节点状态；节点下线分为主观下线和客观下线；客观下线后则会选取从节点进行故障转移

集群只实现了主节点的故障转移，从节点故障时只会被下线。

集群恢复的时间 <= 1.5 * cluster_node_timeout + 1000ms