# ThreadLocal 详解
## ThreadLocal 概述
ThreadLocal，又称线程本地变量，常用于线程间的数据隔离。当使用ThreadLocal维护变量时，ThreadLocal为每个使用该变量的线程提供独立的变量副本，所以每一个线程都可以独立地改变自己的副本，而不会影响其他线程所对应的副本。

## ThreadLocal 的实现原理
从 ThreadLocal 源码中来看，关键方法为 get、set 方法

### get
``` java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
		ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

从源码可以看到，首先从当前线程中获取 ThreadLocalMap，如果 map 存在，则从 map 中获取 this（当前ThreadLocal）的值。如果 map 不存在对应的值或者 map 未初始化，则调用 setInitialValue()。

### setInitialValue
``` java
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
```
该方法首先调用 initialValue 初始化 value，默认返回 null，用户可重写该方法；再通过当前线程获取 ThreadLocalMap，如果 map 存在，则直接将值 set 到 map 里面；如果不存在，则创建 map 并将值 set 进去

### set
``` java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
 }
```
set 方法和 setInitialValue 方法逻辑一致，区别是 value 由外部传入

从 ThreadLocal 的源码中看到，实现原理与 Thread、ThreadLocalMap 有关。

在 Thread 的代码中可以看到，每个 thread 对象都有一个 ThreadLocalMap 的成员变量 threadLocals，该 Map 由 createMap 方法，该 Map 与 HashMap 机制类似，key 为 threadLocal 变量，而 value 为对应的值。

因此实现数据隔离的原理就是通过每个 thread 创建独立的 map 对象，这个map对象存放了当前 thread 的所有 ThreadLocal 变量，从而达到互相隔离的效果。

## F&Q
Q：value 为全局共享对象时，某个线程改变了对象内的值，不是也会影响到其他线程？
![Alt text](./1565784856376.png)




## 附录
彻底理解 ThreadLoacal https://blog.csdn.net/lufeng20/article/details/24314381