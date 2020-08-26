# JVM 查看 GC 日志
## 背景介绍
在 JVM 进程启动时，我们通常会指定 JVM 的初始堆内存与最大内存的大小，当内存使用较大，则会发生 GC 操作，回收内存。要了解内存使用情况以及 GC 的频率，因此需要查看 GC 日志。

## GC 在什么时候发生  
要了解什么时候会触发 GC，先得了解 JVM 内存模型，具体请查看 xxxxxxxx

## GC 日志查看与分析
### 配置 GC 查看
要查看 GC 日志，在启动 Java 进程的时候，需要配置 GC 相关的 JVM 参数
GC 相关的 JVM 参数
- -XX:+PrintGC 输出GC日志
- -XX:+PrintGCDetails 输出GC的详细日志
- -XX:+PrintGCTimeStamps 输出GC的时间戳，以基准时间的形式
- -XX:+PrintGCDateStamps 输出GC的时间戳，以日期的形式
- -XX:+PrintHeapAtGC 在进行GC 的前后， 打印出堆的信息
- -Xloggc:/apps/logs/gc.log 日志文件的输出路径

写一个 helloworld 进行模拟
``` java
public class GCMain {
	public static void main(String[] args) {
		byte[] data = new byte[1024 * 1024 * 8];
		data = null;
		System.gc();
	}
}
```
配置启动参数
``` profile
-XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintHeapAtGC -Xloggc:/app/logs/gc.log
```

运行 main 方法后，打开 gc.log 文件，可以看到
``` log
Java HotSpot(TM) 64-Bit Server VM (25.20-b23) for windows-amd64 JRE (1.8.0_20-b26), built on Jul 30 2014 13:51:23 by "java_re" with MS VC++ 10.0 (VS2010)
Memory: 4k page, physical 16671320k(5609080k free), swap 28766692k(2292464k free)
CommandLine flags: -XX:InitialHeapSize=266741120 -XX:MaxHeapSize=4267857920 -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintHeapAtGC -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:-UseLargePagesIndividualAllocation -XX:+UseParallelGC 
{Heap before GC invocations=1 (full 0):
 PSYoungGen      total 76288K, used 13448K [0x000000076b300000, 0x0000000770800000, 0x00000007c0000000)
  eden space 65536K, 20% used [0x000000076b300000,0x000000076c022198,0x000000076f300000)
  from space 10752K, 0% used [0x000000076fd80000,0x000000076fd80000,0x0000000770800000)
  to   space 10752K, 0% used [0x000000076f300000,0x000000076f300000,0x000000076fd80000)
 ParOldGen       total 175104K, used 0K [0x00000006c1800000, 0x00000006cc300000, 0x000000076b300000)
  object space 175104K, 0% used [0x00000006c1800000,0x00000006c1800000,0x00000006cc300000)
 Metaspace       used 3066K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 333K, capacity 388K, committed 512K, reserved 1048576K
2019-04-25T20:38:02.157+0800: 0.180: [GC (System.gc()) [PSYoungGen: 13448K->808K(76288K)] 13448K->808K(251392K), 0.0041643 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap after GC invocations=1 (full 0):
 PSYoungGen      total 76288K, used 808K [0x000000076b300000, 0x0000000770800000, 0x00000007c0000000)
  eden space 65536K, 0% used [0x000000076b300000,0x000000076b300000,0x000000076f300000)
  from space 10752K, 7% used [0x000000076f300000,0x000000076f3ca020,0x000000076fd80000)
  to   space 10752K, 0% used [0x000000076fd80000,0x000000076fd80000,0x0000000770800000)
 ParOldGen       total 175104K, used 0K [0x00000006c1800000, 0x00000006cc300000, 0x000000076b300000)
  object space 175104K, 0% used [0x00000006c1800000,0x00000006c1800000,0x00000006cc300000)
 Metaspace       used 3066K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 333K, capacity 388K, committed 512K, reserved 1048576K
}
{Heap before GC invocations=2 (full 1):
 PSYoungGen      total 76288K, used 808K [0x000000076b300000, 0x0000000770800000, 0x00000007c0000000)
  eden space 65536K, 0% used [0x000000076b300000,0x000000076b300000,0x000000076f300000)
  from space 10752K, 7% used [0x000000076f300000,0x000000076f3ca020,0x000000076fd80000)
  to   space 10752K, 0% used [0x000000076fd80000,0x000000076fd80000,0x0000000770800000)
 ParOldGen       total 175104K, used 0K [0x00000006c1800000, 0x00000006cc300000, 0x000000076b300000)
  object space 175104K, 0% used [0x00000006c1800000,0x00000006c1800000,0x00000006cc300000)
 Metaspace       used 3066K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 333K, capacity 388K, committed 512K, reserved 1048576K
2019-04-25T20:38:02.161+0800: 0.185: [Full GC (System.gc()) [PSYoungGen: 808K->0K(76288K)] [ParOldGen: 0K->741K(175104K)] 808K->741K(251392K), [Metaspace: 3066K->3066K(1056768K)], 0.0075184 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
Heap after GC invocations=2 (full 1):
 PSYoungGen      total 76288K, used 0K [0x000000076b300000, 0x0000000770800000, 0x00000007c0000000)
  eden space 65536K, 0% used [0x000000076b300000,0x000000076b300000,0x000000076f300000)
  from space 10752K, 0% used [0x000000076f300000,0x000000076f300000,0x000000076fd80000)
  to   space 10752K, 0% used [0x000000076fd80000,0x000000076fd80000,0x0000000770800000)
 ParOldGen       total 175104K, used 741K [0x00000006c1800000, 0x00000006cc300000, 0x000000076b300000)
  object space 175104K, 0% used [0x00000006c1800000,0x00000006c18b97f8,0x00000006cc300000)
 Metaspace       used 3066K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 333K, capacity 388K, committed 512K, reserved 1048576K
}
Heap
 PSYoungGen      total 76288K, used 3277K [0x000000076b300000, 0x0000000770800000, 0x00000007c0000000)
  eden space 65536K, 5% used [0x000000076b300000,0x000000076b633500,0x000000076f300000)
  from space 10752K, 0% used [0x000000076f300000,0x000000076f300000,0x000000076fd80000)
  to   space 10752K, 0% used [0x000000076fd80000,0x000000076fd80000,0x0000000770800000)
 ParOldGen       total 175104K, used 741K [0x00000006c1800000, 0x00000006cc300000, 0x000000076b300000)
  object space 175104K, 0% used [0x00000006c1800000,0x00000006c18b97f8,0x00000006cc300000)
 Metaspace       used 3243K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 353K, capacity 388K, committed 512K, reserved 1048576K
```

### GC 日志分析
在上面 GC 日志中，可以看到进行了两次 GC，一次是 GC，一次是 full gc，每进行一次 GC，都会打印堆前后内存使用情况。
![Alt text](./1556197126331.png)
首先第一段内容是输出年轻代和老年代的使用情况以及 eden、from、to 区域的内存使用占比
然后执行 GC 操作
![Alt text](./1556197235872.png)
可以看到 gc 后，内存使用从 13448K 变成 808K，且耗时 0.0041643 secs，后面的 user：用户耗时，sys：系统耗时，real：实际耗时
GC 后，输出堆使用情况，可以看到年轻代的使用情况降低到 808K

另外除了直接查看 GC 日志，也可通过 gcviewer 进行分析，下载链接为 https://github.com/chewiebug/GCViewer/wiki/Changelog
