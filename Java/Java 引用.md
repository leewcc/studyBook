# Java 引用详解
## 概述
Java 有四种类型的引用，每一种引用的生命周期与其被垃圾回收的时机会有所不同。
## 引用的类型
Java 实际上有四种强度不同的引用，他们的区别在于声明的形式和内存回收机制回收它们的时机不同，从强到弱他们分别是：
- 强引用 StrongeReference
- 软引用 SoftReference
- 弱引用 WeakReference
- 虚引用 PhantomReference

### 强引用
在 Java 中，采用默认的方式新建对象都属于强引用类型。
使用强引用，只要对象还存在引用关系，那么垃圾回收器就不会回收这个对象，即使内存已经不足，JVM 宁愿抛出 OutOfMemoryError 错误，使程序异常终止，也不会随意地回收具有强引用的对象来解决内存不足的问题。
强引用的对象，只有当没有任何引用关系才会被回收。

### 软引用
软引用在强度上弱于强引用，通过 SoftReference 来定义，如
``` java
SoftReference<User> user = new SoftReference<User>(new User("test"));
```
它是用来描述一些还有用但并非必须的对象，对于被软引用关联着的对象，在系统将要发生 OutOfMemoryError 错误之前，会把这些对象列进回收访问之中进行第二次的回收。

软引用被回收还有一个前提是除弱引用引用外，不存在更强的引用引用该对象，否则仍然不会被回收，如以下例子
设置 JVM 参数为 -Xms60M -Xmx60M
``` java
public class TestReference {

	public static void main(String[] args) {
		byte[] a = new byte[1024 * 1024 * 30];
		SoftReference softReference = new SoftReference(a);
		System.gc();
		System.out.println(softReference.get());
		byte [] b = new byte[1024 * 1024 *30];
		a = null;
		System.gc();
		System.out.println(softReference.get());
	}
}
```
程序申请了一个 30M 大的字节数组，并同时用 a 引用与使用 SoftReference 引用，此时该字节数组同时被强引用与弱引用引用，后尝试再申请一个 30M 的字节数组，gc 并没有将 a 引用的字节数组回收，导致程序出现 OOM
![Alt text](./1557557723521.png)
如果把 a = null 移前到 b 初始化前，则会成功初始化完毕，而 a 引用的字节数组也能成功被 gc 掉，因此弱引用被回收的前提是没有被其他强引用引用。
![Alt text](./1557558271062.png)


### 弱引用
弱引用和软引用一样，也是用来描述非必需对象的，当 JVM 进行垃圾回收时，无论内存是否充足，都会回收弱引用。这意味着，被弱引用关联的对象只能生存到下一次垃圾收集发生之前。
注：弱引用和软应用一样，只有当不存在更强的引用，才会表现为弱引用

### 虚引用
虚引用也被称为幽灵引用或者幻影引用，它是最弱的一种引用关系。与其他引用不同的是，虚引用并不会决定对象的生命周期，如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被回收。而且它控制其指向的对象非常弱，以致于它不能获得这个对象。
虚引用的使用：跟踪在 ReferenceQuene 中已经被回收的对象。

### ReferenceQueue
对于一些特别被非强引用的对象被回收时需要做清除、关闭、释放操作的对象，为了避免内存泄露，可以通过使用 ReferenceQueue 来实现引用的清除机制，而且可以为被回收的对象进行收尾处理。
虚引用和弱引用的不同在于其进入 ReferenceQueue 的时机