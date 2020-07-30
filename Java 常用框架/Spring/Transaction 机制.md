# Transactional

Transacttional 注解的参数：
- value：指定 transactionManager
- timeout：指定事务的超时时间，不设置默认为 -1
- propagation：指定事务的传递行为
> spring 中支持7种事务传播属性
> 1. REQUIRED:：支持当前事务，如果当前没有事务，则新建一个事务（默认值）
> 2. SUPPORTS：支持当前事务，如果当前没有事务，则以非事务执行
> 3. MANDATORY：支持当前事务，如果当前没有事务，就抛出异常
> 4. REQUIRED_NEW：新建事务，如果当前存在事务，则把当前事务挂起
> 5. NOT_SUPPORTED：以非事务的方式执行操作，如果当前存在事务，则将当前事务挂起
> 6. NEVER：以非事务的方式执行，如果当前存在事务，则抛出异常
> 7. NESTED：如果当前存在事务，则启用嵌套事务；否则创建一个新的事务；该种方式与 REQUIRED_NEW 的区别在于，内层事务受外层事务的影响，当外层事务 commit，内层事务才 commit；当外层事务 rollback，内层事务也会 rollback。**启动该传播机制必须在 transactionManager 打开 nestedTransactionAllowed 属性**
 
- readOnly：指定事务是否只读，若设置为 true，执行写将会抛出异常
- rollbackFor：指定触发事务回滚的异常类型。
> 当该参数不配置时，则默认只对 RuntimeExcepton 和 Error 进行回滚；当配置指定异常后，执行时出现异常，会根据 Rule 规则判断抛出的异常是否命中，如果均不命中，则走回默认检查（RuntimeException + Error）
- noRollbackFor：指定不会触发事务回滚的异常类型
- isolation：指定事务的隔离级别

## 事务的 ACID 特性
- A 原子性 Atomicty
- C 一致性 Consistency
- I 隔离性 Isolation
- D 持久性 Durability

## 事务的隔离级别
事务的隔离级别有 4 种
| 隔离级别   |    脏读 |   不可重复读   | 幻读 |
| :--------| :--------:| :------: | :-----:|
| Read Uncommited(未提交读)  |   Y |  Y  | Y |
| Read commited(提交读)  |   × |  Y  | Y |
| Repeatable Read(可重复读)  |  ×  | ×   | Y |
| Serializable(串行化)  |   × |  ×  | × |

脏读：事务 A 读到其他事务未提交的数据
不可重复读：事务 A 连续读取同一行已存在的数据，两次读出的数据不一致，第二次能读到其他事务已提交的更新值
幻读：事务 A 能读到其他事务已提交的插入记录

在 MySQL innoDB 引擎中， RR 级别下通过 MVCC 与 next-key lock 解决了幻读的问题
具体请查看 **MySQL InnoDB——mvcc、锁机制**

## spring 事务执行机制
对于使用 @Transactional 注解的类或者方法，spring 会为该类生成一个代理类，代理类增加事务的功能，通过注入 TransactionInterceptor，在调用方法前先进入拦截器进行事务的相关操作。

事务的主要功能由 TransactionAspectSupport 的 invokeWithinTransaction 方法来实现

流程：
1. 获取 transactionManager
2. 根据配置判断是否需要创建事务
3. 执行目标方法
4. 如果正常执行，则执行 commitTransactionAfterReturning，进行事务的提交 
5. 如果抛出异常了，则触发 completeTransactionAfterThrowing 方法，该方法根据异常类型是否需要回滚；该方法默认对所有的 RuntimeException 和 Error 进行回滚；如果配置 rollbackFor，则命中对应的异常也会进行回滚，如果不命中则按默认规则处理；配置了 noRollbackFor，则会排除对应的异常类型；如果异常均没有命中，则仍然会触发提交 
