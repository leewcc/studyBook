# Summary

* [Introduction](README.md)
* Java
  * ArrayList 和 LinkedList 的区别
  * HashMap、HashTable 和 ConcurrentHashMap 的区别
  * Exception 和 Error 的区别，常见的 RuntimeException
  * wait 和 sleep(Java/wait 和 sleep.md)
  * notify 和 notifyAll 的区别
  * finalize 作用
  * ThreadLocal 的实现原理
  * Synchronized 和 Lock 的区别
  * Java 的线程池
  * Future、Runnable 和 Callable
  * JVM 的内存模型
  * validate 关键字
  * Java 的引用类型与区别
  * 反射
  * 类加载过程
  * 双亲委派模型的过程与优势
  * stream 的原理
  * 常见的 GC 算法，CSC 和 G1 的区别
  * Gc root 的判断
* Spring
  * IoC 机制
  * 动态代理的机制
  * AOP 机制
  * 事务机制
  * 异常机制
  * Spring 使用了什么设计模式
* Spring MVC
  * springMVC 的工作流程
  * DispatchServlet 是什么
* Spring Boot
  * Spring Boot 的常用注解
* Mybatis
  * Mybatis 的一级缓存和二级缓存
  * Mybatis 的插件原理
  * MyBatis 如何映射到 mapper
* 常见数据库客户端
  * Jedis
  * Lettuce
  * HikariCP
  * Druid
  * SpyMemcached
  * Xmemcached
  * Guava
  * Caffeine
  * OHC
  * Mycat
* MySQL
  * 常见的存储引擎有哪些，它们有什么区别
  * 事务的隔离解别，innoDB 如何解决 RR 级别下幻读问题
  * MySQL 如何保证事务的正确性
  * 常见的索引实现方式，它们分别适用什么场景
  * innoDB 的索引机制
  * 如何优化慢查询
  * 乐观锁和悲观锁
  * MySQL 的锁机制
  * MySQL 如何保证数据的持久性
  * MySQL 有多少种主从复制模式，它们的区别是什么
  * 如何解决数据写冲突问题
* Redis
  * 常见的数据类型以及应用场景
  * 字符串类型的底层原理
  * hash 字典的底层实现原理
  * sort set 的底层实现原理
  * redis 的 key 删除原理
  * redis 的持久化机制
  * redis 是单线程的嘛
  * redis 的主从复制机制
  * redis sentinel 原理
  * redis cluster 原理
  * redis 实现分布式锁
  * redis 实现乐观锁 
* Memcached
  * memcached 的底层原理
  * memcached 如何解决写冲突
* 计算机网络
  * ISO 网络七层协议模型
  * TCP
    * 三次握手
    * 四次挥手
    * time_wait 的作用——记一次连接风暴带来的故障
    * TCP 的可靠传输机制（TCP 包编号、按序确认、超时重传）
    * TCP 的拥塞控制和流量控制（滑动窗口、拥塞算法）
    * TCP 的 negla 算法
    * TCP 如何解决粘包、半包的问题
    * TCP 的连接保活计数器
  * UDP
    * UDP 的机制
    * 基于 UDP 设计一个可靠的网络协议
  * HTTP
    * HTTP 1.0 和 1.1 的区别
    * HTTP 协议规范
    * HTTP method
    * 常见错误码
    * HTTP 的 keep-alive
    * session 与 cookie
  * HTTPS
* 操作系统
  * 进程与线程
  * 进程的状态，孤儿进程与僵尸进程
  * 进程的通信方式
  * 进程调度
  * 死锁的条件与解除
  * 用户态和内核态
* 分布式框架
* 数据结构
  * 排序算法
    * 选择排序
    * 冒泡排序
    * 插入排序
    * 希尔排序
    * 堆排序
    * 归并排序
    * 快速排序
    * 计数排序
    * 桶排序
    * 基数排序
  * 链表与数组的区别
  * 哈希表，解决冲突的方式
  * 树
    * 二叉树
    * 顺序搜索树
    * 平衡二叉树
    * B树
    * B+树
    * B-树
    * 红黑树
  * 图
    * 有向图
    * 无向图
* 网络编程
* Linux
  * 常见的基本命令使用
    * ps
    * top
    * grep
    * netstat
    * sed
    * awk
    * tcpdump
    * head
    * tail
    * nl
    * id
  * select 和 epoll 的区别
  * fork 的原理
* Netty
* Nginx
* 实际业务解决方案


