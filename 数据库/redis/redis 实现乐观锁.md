# redis 乐观锁：
依赖 MULTI 、EXEC、WATCH 命令实现
MULTI：表示打开事务，在执行 EXEC 前，客户端的命令都将进入事务队列
EXEC：将事务队列的命令按先进先出的顺序全部执行，执行前会检 REDIS_DIRTY_CAS 标识，该标识为 true，则表示当前事务不会被执行
WATCH：在执行 EXEC 命令执行之前，监视任意数量的数据库键，并在 EXEC 命令执行时，检查被监视的键是否被修改了，如果被修改了则拒绝执行事务，并返回空

WATCH 的机制：
每个 redis 数据库都会保存一个 watched_keys 字典来保存被监控的键，字典值是一个链表，链表中记录了所有监视对应键的客户端

键值被修改时
所有对数据库进行修改的命令，执行候会调用 touchWathKey 对 watched_keys 字典进行检查，如果存在客户端监视该 key，则会将 REDIS_DIRTY_CAS 标识打开，标识客户端事务不可用