# Redis 常见的数据类型和使用场景
redis 支持五种类型
1. String 字符串
2. List 列表
3. Hash 字典
4. Set 集合
5. Sorted Set 有序集合

redis Key/value 的内存存储模型
![Alt text](./1593674261748.png)

**String**
redis 的 String二进制安全，它可以存储 Sring，也可以是数值，**最大只能存储 512MB**

实现原理：
每一个 String 类型的用一个结构体 SDS 存储
```
struct SDS {
	int length;   // buf 中已存储的长度
	int free
	char[] buf;
}
```

当字符串进行扩展时，则会判断当前 free 是否足够，不足够的时候会重新申请空间；
当字符串进行收缩时，则会直接修改 free 和 length

优点：
1. 获取字符串长度时间复杂度为 O(1)
2. 防止缓冲区溢出，避免自身的修改修改了其他缓冲区的数据
3. 减少扩展或收缩字符串带来的内存重分配次数
4. 二进制安全，因为 C 语言的字符串是通过空字符 \0 来结束的，redis 使用 length 字段来决定字符串的长度，所以允许字符串中出现 \0


使用场景：
1. 常规的 key-value 缓存
2. 常规计数，如微博数、粉丝数

**Hash**
键值对，即Map，适合存储对象

实现原理：
在 redis 的 Map 结构体定义
``` 
哈希表节点 (java 表示)
class DicEntry {
	String key;
	String value;
	DicEntry next;
}

哈希表
class Dictht {
	DicEntry[] tables;
	long size;
	long sizeMask; 	// 用于计算 hash 值
	long used;		// key 存储个数
}

class Dict {
	Dictht[2] dicts;// 两个哈希表
	long rehashIdx;	// 用于表示当前 rehash 第几个桶 
}
```

上述结构上看，在 redis 字典里面，存储了两个哈希表，表里的每个元素通过链地址法来解决哈希冲突

为什么需要使用两个哈希表来存储字典呢
>在 Dict 的结构体里的 dicts 和 rehashIdx 都是用于扩容缩容的

扩容的时机：
>1. used > size（即负载因子 > 1） 且当前没有在执行  BGSAVE 命令或者 BGREWRITEAOF 命令
>2. used > size * 5（可配置）


缩容的时机：
> 有一个周期函数会周期检查，如果 used < size * 0.1,则会触发缩容，最小缩到 4

扩容的机制
>初始化新的 table 赋值给 dict[1],待 rehash 完成，则将 dict[1] 赋值给 dict[0]，将 dict[1] 置为 null
redis 扩容采用渐进式扩容的方式，即不会一次性全部扩容，放到对应的桶上，而是通过其他行为来完成 rehash
>1. 调用增删查改命令，如果当前处于 rehash，则会触发一次帮助 rehash 的操作。对于处于 rehash 状态时，写操作都将写到 dict[1]，读操作先读 dict[0]，如果读不到再读 dict[1]
>2. 如果开启了 activerehashing，则会在周期函数中花费 1ms 的时间进行渐进式 rehash


渐进式rehash 的机制
>1. rehashIdx 记为上次迁移完的桶，先从 rehashIdx 开始找到还没迁移的桶（最多扫描10个桶是空的，此时则退出了）
>2. 找到未迁移的桶，则将该桶的数据全部迁移；若找不到，则 rehash 完成
>3. 如果迁移完，还剩余 used，则将 rehashIdx 加一

渐进式rehash 有什么缺点：
>因为在 rehash 期间，有两个表同时在使用，会使 redis 内存使用量突增，在 redis 满容状态下由于 Rehash 导致dao大量key 被驱逐


**List**
双向链接，能用于作为 stack 或者 queue

实现原理：
底层实现就是个双向链表

使用场景：
1. 消息队列


**Set**
无序集合

实现原理
在存储元素均为整数且元素数目较少时，使用一个整数有序数组存储；否则使用 dict 实现

整数有序数组的结构
![Alt text](./1593685013706.png)


使用场景：
需要存储一个列表数据，但是又不喜欢出现重复数据；可用于所有粉丝、共同关注、共同爱好等功能。

**Sorted Set**
有序集合

实现原理：
通过跳表来实现的，在增删查改可以达到 O(logn)
![Alt text](./1593684812263.png)


使用场景：
1. 带权重的消息队列
2. 排行榜
