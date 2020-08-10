# redis 的内存淘汰策略
redis 部署时需要指定内存大小，当存储的数据超过内存大小则会根据配置的淘汰策略进行处理。

内存大小与淘汰策略配置方式
1. redis 配置文件配置
``` conf
maxmemory 100mb
maxmemory-policy noeviction 
```

2. 通过命令修改
config set maxmemory 100mb
config set maxmemory-policy noeviction

## Redis 的淘汰策略类型
redis 在 4.0 以下的版本支持 LRU，在 4.0 开始支持 LFU，策略有

- noeviction（默认值）：非 del 的写请求不触发淘汰，返回错误
- allkeys-lru：对所有 key 进行 lru 淘汰
- volatile-lru：对配置了超时时间的 key 进行 lru 淘汰
- allkeys-ramdom：对所有 key 中随机淘汰数据
- volatile-random：对配置了超时时间的 key 随机淘汰数据
- volatile-ttl：对配置了超时时间 的 key，根据过期时间进行淘汰，越早过期的优先淘汰
- allkeys-lfu：对所有 key 进行 lfu 淘汰
- volatile-lfu：对配置了超时时间的 key 进行 lfu 淘汰

目前生产部署现状：所有集群淘汰策略均默认配置为 noeviction，不进行淘汰。在监控系统会对 redis 内存使用情况进行监控，当内存到达告警线，则会触发告警，与业务沟通是否需要调整或者清除数据

## Redis 的 LRU
Redis 的 LRU 采用 近似 LRU 的方式，不是采用标准的 LRU，主要原因是因为实现简单，且节省内存，同时淘汰效果与标准 LRU 相差不大

近似LRU的原理：
在 Redis 中，每一个 key 都会存储一个 24 位的时间戳，key 被更新时该时间戳同时也会被刷新，当触发 LRU 时，会随机抽样取 N（默认是5，可通过 maxmemory-samples 配置） 个 key，然后从这 N 个 key 中，使用时间戳与全局时间戳比较，淘汰最久未被访问的 key。

## Redis 的 LFU
redis 4.0 开始支持 LFU，使用 LFU 的淘汰机制时，redis 将 24 位时间戳拆分成2部分，低 8 位记为 counter，高16位记为时钟

计算机制：
- lfu-log-factor：counter 的增长因子
- lfu-decay-time：counter 的衰变周期

**降低LFUDecrAndReturn**
1. 获取高16位的时间 ldt，低8位的计数器 counter
2. 计算 now 与 ldt 的差值，如果 ldt 大于 now，则采用 65535 - ldt + now 计算
3. 使用第二步计算的差值除以 lfu_decay_time，即 LFUTimeElapsed(ldt)/ lfu_decay_time,过去了多少个time，则将 counter - n

**增加 LFULogIncr**
1. 获取 0-1 的随机数 r
2. 计算 0-1 之间的控制因子 p，p = 1 / ((counter - LFU_INIT_VAL) * lfu_log_factor + 1)
3. 如果 r < p，则 counter + 1

## Redis 的 key 删除机制
在 redis，配置了超时的 key 会存放在一个 expire 的哈希表中，当 key 过期厚，key 不会自动移除，而是通过定时删除和惰性删除机制进行移除 
1. 定时删除
redis 有一个定时任务，每 100ms 触发一次，该定时任务会执行移除过期的 key，每次随机取 20 个 key，对这 20 个 key 进行移除；如果移除的比率超过 1/4，则重复上述步骤

2. 惰性删除
惰性删除的触发是客户端访问已过期的 key 时触发，当被访问的 key 已过期，redis 则会直接 remove 掉，返回空