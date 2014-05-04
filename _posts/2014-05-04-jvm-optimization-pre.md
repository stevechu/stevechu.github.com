---
layout: post
title: jvm 优化
category: java
---
###JVM调优
@[jvm|优化]

Base：jvm结构，gc机制

    堆
    
堆中存放的是对象，可以被线程共享。可以用以下方法获取总使用量。
{% highlight java %}
System.out.println(Runtime.getRuntime().totalMemory());
{% endhighlight %}
堆包括两个区域：新生代和老年代。
新生代：
- 程序新创建的对象都是从新生代分配内存，新生代由Eden Space和两块相同大小的Survivor Space(通常又称S0和S1或From和To)构成(可通过`-XX:PrintHeapAtGC`等参数来查看)，可通过`-Xmn`参数来指定新生代的大小，也可以通过`-XX:SurvivorRation`来调整Eden Space及Survivor Space的大小。

老年代：
- 用于存放经过多次新生代GC后仍然存活的对象，例如缓存对象，新建的对象也有可能直接进入老年代，主要有两种情况：1.大对象，可通过启动参数设置`-XX:PretenureSizeThreshold=1024` (单位为字节，默认为0)来代表超过多大时就不在新生代分配，而是直接在老年代分配。2.大的数组对象，切数组中无引用外部对象。老年代所占的内存大小为`-Xmx对应的值减去-Xmn对应的值`。


    参数举例
    
`-Xmx`：设置JVM最大可用内存  
`-Xms`：设置JVM促使程序初始化的时候内存。此值可以设置与 -Xmx相同 ，以避免每次垃圾回收完成后JVM重新分配内存。  
`-Xmn`：设置年轻代大小为2G。增大年轻代后，将会减小年老代大小。此值对系统性能影响较大，Sun官方推荐配置为整个堆的3/8。  
`-Xss`：设置每个线程的堆栈大小。JDK5.0以后每个线程堆栈大小为1M，以前每个线程堆栈大小为256K。更具应用的线程所需内存大小进行调整。  
`-XX:NewRatio=4`:设置年轻代（包括Eden和两个Survivor区）与年老代的比值（除去持久代）。设置为4，则年轻代与年老代所占比值为1：4，年轻代占整个堆栈的1/5  
`-XX:SurvivorRatio=4`：设置年轻代中Eden区与Survivor区的大小比值。设置为4，则两个Survivor区与一个Eden区的比值为2:4，一个Survivor区占整个年轻代的1/6  
`-XX:MaxPermSize=16m`:设置持久代大小为16m。
`-XX:MaxTenuringThreshold=0`：设置垃圾最大年龄。如果设置为0的话，则年轻代对象不经过Survivor区，直接进入年老代。对于年老代比较多的应用，可以提高效率。如果将此值设置为一个较大值，则年轻代对象会在Survivor区进行多次复制，这样可以增加对象再年轻代的存活时间，增加在年轻代即被回收的概率。  
`-XX:+UseParallelGC`：选择垃圾收集器为并行收集器。此配置仅对年轻代有效。即上述配置下，年轻代使用并发收集，而年老代仍旧使用串行收集。  
`-XX:ParallelGCThreads=20`：配置并行收集器的线程数，即：同时多少个线程一起进行垃圾回收。此值最好配置与处理器数目相等。  
`-XX:+UseParallelOldGC`：配置年老代垃圾收集方式为并行收集。JDK6.0支持对年老代并行收集。  
`-XX:MaxGCPauseMillis=100`:设置每次年轻代垃圾回收的最长时间，如果无法满足此时间，JVM会自动调整年轻代大小，以满足此值。  
`-XX:CMSFullGCsBeforeCompaction`：由于并发收集器不对内存空间进行压缩、整理，所以运行一段时间以后会产生“碎片”(标记清除)，使得运行效率降低。此值设置运行多少次GC以后对内存空间进行压缩、整理。  
`-XX:+UseCMSCompactAtFullCollection`：打开对年老代的压缩。可能会影响性能，但是可以消除碎片。  

如果堆设置小了，可以会造成内存碎片、高回收频率以及应用暂停而使用传统的标记清除方式；如果堆大了，则需要较长的收集时间。一般设置参数可以参考一下数据：

 - 并发垃圾器收集信息 
 - 持久代并发收集信息
 - GC信息
 - 年轻带年老代回收时间

当响应时间要求以及吞吐量大的时候，可以增大年轻代的大小。年老代可以设置并发收集器。这样可以尽可能回收掉大部分短期对象，减少中期的对象，而年老代尽存放长期存活对象。



较小堆引起的碎片问题：因为年老代的并发收集器使用标记清除或标记整理算法，所以不会对堆进行压缩。当收集器回收时，他会把相邻的空间进行合并，这样可以分配给较大的对象。但是，当堆空间较小时，运行一段时间以后，就会出现“碎片”，如果并发收集器找不到足够的空间，那么并发收集器将会停止，然后使用传统的标记清除(整理)方式进行回收。如果出现“碎片”，可能需要进行如下配置：
`-XX:+UseCMSCompactAtFullCollection`：使用并发收集器时，开启对年老代的压缩。
`-XX:CMSFullGCsBeforeCompaction=0`：上面配置开启的情况下，这里设置多少次Full GC后，对年老代进行压缩。  

应用举例  












- 高并发, 多cpu, 大内存场景   
-server   
-Xmx3g  
-Xms3g  
-XX:MaxPermSize=128m  
-XX:NewRatio=1  eden/old 的比例  
-XX:SurvivorRatio=8  s/e的比例  
-XX:+UseParallelGC  
-XX:ParallelGCThreads=8  
-XX:+UseParallelOldGC  这个是JAVA 6出现的参数选项  
-XX:LargePageSizeInBytes=128m 内存页的大小， 不可设置过大， 会影响Perm的大小。  
-XX:+UseFastAccessorMethods 原始类型的快速优化   
-XX:+DisableExplicitGC  关闭System.gc()   
- 针对低延迟的Java虚拟机调优选项   
-Xmx512m   
-XX:+UseParNewGC  
-XX:+UseConcMarkSweepGC  
-XX:+UseTLAB  
-XX:+CMSIncrementalMode  
-XX:+CMSIncrementalPacing   
-XX:CMSIncrementalDutyCycleMin=0  
-XX:CMSIncrementalDutyCycle=10  
如果想最小化执行时间，通常最好是尽量减少垃圾收集次数，这可以通过设置一个更大的堆大小而做到。这将减少垃圾收集次数，但是会使垃圾收集时间变长，而且垃圾收集暂停时间会比使用低延迟选项时长。 

>参考《深入理解JVM虚拟机》
jvm调优汇总 http://macrochen.iteye.com/blog/599698
jvm调优总结 http://unixboy.iteye.com/blog/174173