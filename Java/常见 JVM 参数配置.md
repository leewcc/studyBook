[TOC]
# JVM 参数配置
## 内存设置
1. -Xmx Java Heap 最大值
2. -Xms Java Heap 初始值
3. -Xmn Java Heap young 区大小
4. -Xss 每个线程的 stack 大小

```
-Xmx2048M -Xms1024M -Xss512k -Xmn1500m
```


## 开启 debug 模式
``` 
-Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=y,address=3999
```

## 配置 JMX
参数
- -Dcom.sun.management.jmxremote
- -Djava.rmi.server.hostname=192.168.0.234
- -Dcom.sun.management.jmxremote.port=9008
- -Dcom.sun.management.jmxremote.authenticate=false
- -Dcom.sun.management.jmxremote.ssl=false

``` 
-Dcom.sun.management.jmxremote -Djava.rmi.server.hostname=192.168.0.234 -Dcom.sun.management.jmxremote.port=9008 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false
```

远程监控工具
- JDK 自带的 jcosole 程序，主要可以查看 JMX 的属性和调用方法
- JDK 自带的 jvisualvm 程序，主要可以可视化的监控进程运行情况

```
-Dcom.sun.management.jmxremote -Djava.rmi.server.hostname=192.168.0.234 -Dcom.sun.management.jmxremote.port=9008 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false
```