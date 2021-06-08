---
layout:     post
title:      "Java基础: GC(四) SA调试工具"
date:       2019-09-01
author:     "ZhouJ000"
header-img: "img/in-post/2019/post-bg-2019-headbg.jpg"
catalog: true
tags:
    - jvm
--- 

[Java基础: GC(一) 介绍](https://zhouj000.github.io/2019/04/23/java-base-gc1/)  
[Java基础: GC(二) G1垃圾收集器](https://zhouj000.github.io/2019/05/04/java-base-gc2/)  
[Java基础: GC(三) G1垃圾收集器优化](https://zhouj000.github.io/2019/08/05/java-base-gc3/)  
[Java基础: GC(四) SA调试工具](https://zhouj000.github.io/2019/09/01/java-base-gc4/)  




# Serviceability Agent

JDK提供了Hotspot SA，这是**一系列**优秀的API和工具的统称，适用于语言层和虚拟机层，**支持调试运行着的Java进程、core文件、虚拟机crash之后的dump文件**。构建高性能的Java应用程序过程中，必然遇到线程问题、内存泄露、应用崩溃等意想不到的情况，这个过程中，SA能很好帮助发现和分析问题

SA链接到运行着的Java进程时，会停止当前进程，并检查该时间点的堆栈信息、线程运行情况、虚拟机内部的数据结构、类、方法信息等，当在SA断开Java进程链接后，进程恢复运行状态。而SA也能检查JVM快照和core文件，相当于一个静态查看工具。因此SA是**独立于目标进程的单独进程**，不会影响目标进程的代码逻辑，并且观察时会**暂停**目标进程的执行，直到SA**断开**连接，并且由于是**快照调试**，因此与常见的调试工具不同，并**不具备单步调试的能力**

SA由**少量的原生代码**和**Java类组成**，原生代码负责从进程和core文件读取原始二进制数据。在当前版本的JDK中，SA的二进制文件由2个：  
1、sa-jdi.jar(提供SA中的Java API和基于这些API实现的调试工具)  
2、Windows下的sawindbg.dll 或 Linux下的libsaproc.so

#### SA如何获取Hotspot虚拟机的内部数据结构

Hotspot的[vmStructs.cpp](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/runtime/vmStructs.cpp#l269)定义了**vmStructs**类，该类包含了所有Hotspot的声明、类属性字段，并且也包含了跟处理器相关的内容，如寄存器信息、sizeof类型等，SA从这个定义文件中载入Java对象和Hotspot数据结构。

而且vmStructs被声明为**友元类**(friend class)，对于大多数Hotspot定义的类来说，vmStructs都是友善的，因此vmStructs可以**访问他们的私有属性**。在Hotspot编译中，vmStructs.cpp被编译为vmStructs.o，该链接库被打包在libjvm.so或jvm.dll中，vmStructs.o中包含了所有SA需要从Hotspot虚拟机中获取的**数据结构描述**，在虚拟机运行期间，SA能够从目标虚拟机中加载数据

SA中对应的Java代码，会用到vmStructs.cpp中声明的结构体和属性名，并保持一致。因此如果vmStructs.cpp中定义的属性名称被修改或删除导致与对应操作该属性的Java代码不同步，SA在尝试分析Java进程或core文件的时候，就可能会遇到问题。所以SA的部分Java代码实际上是Hotspot虚拟机的C++代码的**镜像**，如果有数据结构体或算法的修改或删除，需要在SA的Java代码部分做**相应调整**，因此SA会与Hotspot虚拟机的版本对应起来。JDK提供了一个说明文件在sa-jdi.jar中，其中的sa.propertiies包含了SA的版本属性
```
// jdk1.8.0_77
sun.jvm.hotspot.runtime.VM.saBuildVersion=25.77-b03
```
运行分析时SA从这里读取版本信息与目标虚拟机的**版本信息对比**，如果不匹配则抛出异常。也可通过`-Dsun.jvm.hotspot.runtime.VM.disableVersioinCheck`来关闭，这样即使版本不匹配，也不会报错并强制链接到目标虚拟机，这种情况下可能遇到问题，也可能没问题，取决于是否用到了修改的部分



# SA调试工具

## HSDB

HSDB是主要的图形用户界面工具，帮助分析Java进程、core文件或远程的Java进程，在之前的[博客](https://zhouj000.github.io/2019/03/27/java-base-jvm6/)中也已经使用过HSDB了，配好环境变量后使用`java -classpath "%JAVA_HOME%/lib/sa-jdi.jar" sun.jvm.hotspot.HSDB`启动，在jdk 9之后的版本，还增加了一种启动方式`\bin\jhsdb.exe hsdb/clhsdb/jstack/jmap/jinfo`

SA可以通过3种不同形式连接到目标的Hotspot虚拟机上：  
1、连接到**本地Hotspot进程**上(运行中的Java进程)  
2、链接到**core文件**(选择Core Dump文件或Crash Dump File)  
3、连接到**远程debug**服务商(没用过，要靠RMI注册服务)

![hsdb1](/img/in-post/2019/09/hsdb1.png)
通过jps或ps获取进程ID，由于SA是**快照调试器**，因此链接上后该进程就处于**暂停**状态，直到SA**断开**连接(Detach)

HSDB链接上后的第一个窗口，就列出了目标JVM的所有线程。选择一个线程可以查看线程详情(放大镜)，将会给出线程对象的VM的中间表示(VM representation，JIT编译过程中把Java字节码转换为自己的中间表现形式(IR)，然后做一些优化，最终生成机器码与相关的元数据，对于VM层来说IR是通用描述)。可以已通过第二个方框查看栈内存，显示栈内存数据
![hsdb2](/img/in-post/2019/09/hsdb2.png)
另外边上还有显示栈调用路径、线程信息、发现Crashe

点击**tool**，里面有一些实用工具，可以帮助定位问题。比如类浏览器、死锁检测、对象监视器、对象直方图(显示所有能到达选中实例的访问路径、以及GC状态、存活或待回收)、计算反向指针(计算出从GC根能够到达的对象的引用路径)、查找Object对象(类SQL语言)、查找指针(某个对象在Java堆中所有引用的地址)、代码缓存区查值(对查找JIT相关问题有用)、内存视图、对象监视器缓存暂存、代码查看器(展示方法的字节码和JIT编译器编译后的机器码)、堆要素、系统变量、虚拟机版本信息、命令行参数
![hsdb3](/img/in-post/2019/09/hsdb3.png)

同样的，除了HSDB窗口化模式，JDK还提供了命令行调试器CLHSDB，通过`java -classpath "%JAVA_HOME%/lib/sa-jdi.jar" sun.jvm.hotspot.CLHSDB`进入，使用命令进行查询，提供的功能与界面版的一模一样，通过`help`可以查看参数，比如可以通过inspect命令分析

## 其他

除了这个，SA还打包了很多非常便利的**小工具**，这些小工具也能连接上进程、core文件和远程调试：  
1、终结器信息(`...hotspot.tools.FinalizerInfo <PID>`)，打印出目标虚拟机可销毁对象的详细信息  
2、堆Dumper(`...hotspot.tools.HeapDumper <PID>`)，可以用hprof格式转存Java堆数据到文件中  
3、PMap(`...hotspot.tools.PMap <PID>`)，类似于Solaris的pmap工具，打印出目标进程或core文件的进程内存映像信息  
4、对象直方图(`...hotspot.tools.ObjectHistogram <PID>`)，不光在HSDB中使用，还能单独使用  
5、结构化对象查询(`...hotspot.tools.soql.SOQL <PID>`)，即OQL语言  
6、ClassDump(`... -Dsun.jvm.hotspot.tools.jcore.outputDir=classes sun.jvm.hotspot.tools.jcore.ClassDump <PID>`)，可以到处目标虚拟机装载的类，可以选择导出单个类或特定包中的全部类，使用outputDir属性指定Dump指定类到目标文件夹中  
7、到处Java飞行记录表(`...hotspot.tools.DumpJFR <PID>`)，能导出Java飞行记录表，这个是JDK7u40版本开始提供的，JFR收集运行中应用的诊断信息和分析数据的工具，它集成在Java虚拟机中，并且基本不会造成性能损耗，Java任务控制面板可以分析JFR收集的数据  

扩展：  
[JVM 对象查询语言（OQL）](https://blog.csdn.net/pange1991/article/details/82023771)  

## 另外两种方式

core文件和崩溃dump文件是运行程序在**特定时间的所有状态信息构成的二进制文件**。core文件通常是应用因为**违反**了操作系统或硬件的保护机制而**异常终止**的时候，由**操作系统自动创建**，操作系统会**杀死**应用并**创建一个core文件**以便后续分析使用，它包含了应用程序死亡时的**详细状态描述信息**。当然在程序运行正常的时候也可以**手动**去创建core和崩溃dump文件，在Linux/Solari上使用gcore工具从运行中/挂起的进程中导出core文件；在windows上利用winDbg、userdump或ADPlus去获取崩溃dump文件，这样在不具备调试运行中应用的时候，core文件对诊断问题可以提供很大帮助

当core文件是在一台机器上创建的，我们需要在另一台机器上调试它，本地调试器(dbx、gdb)和SA一样，并不是每次都能打开非本地生成的core文件，这是因为core服务器和调试服务器上的**Kernel和依赖库不一致**。如果core文件是在虚拟机崩溃时生成的，那么系统同时也生成了崩溃日志，我们可以从崩溃日志(hs_err)得到进程加载的共享库的明细。当遇到共享库问题时，需要拷贝core服务器上的程序依赖共享库到服务器上，并设定SA环境变量指向它

SA的二进制分发包sa-jdi.jar也提供了Java调试接口(**JDI**)实现，因此任何的JDI客户端(比如JDB)都可以通过JDI连接器链接到core文件和Java进程上。连接器方法attach()返回的虚拟机对象是只读的，这些虚拟机对象不能被修改，因此使用连接器的JDI客户端**不能调用任何JDI方法**，否则会抛出异常

我们还可以通过继承SA API提供的sun.jvm.hotspot.tools.Tool去实现自己的查错调试工具


## VisualVM的SA插件

VisualVM是一个可视化的Java应用查错、监控和分析工具，被定位为全功能的多合一Java应用监控查错工具(JDK6、7、8绑定了jvisualvm)。大部分随着JDK发布的独立工具，比如jstack、jmap等都可以在VisualVM中使用。VisualVM另一个非常好的设计就是它的插件功能，可以基于它的API编写插件，并把插件发布给其他人使用。VisualVM的SA插件让我们能在VisualVM中使用SA的主要功能，该插件让VisualVM具备查看堆中Java对象、VM结构体、Java应用的线程、方法编译后的字节码、查看堆中指定地址的对象和引用等等，都可以在可视化界面中看到

VisualVM的可用插件都在VisualVM**插件中心**，通过工具--插件，可以选择下载并安装。安装完毕后在应用视图中会出现SA插件的标签页。在SA插件标签页中可以单击链接到进程，与之前相同，当SA完成链接操作，目标进程**暂停运行**，就可以查看目标进程的**快照**了
![vmsa](/img/in-post/2019/09/vmsa.png)
![vmsa2](/img/in-post/2019/09/vmsa2.png)


# 故障分析

## OOM

出现OOM异常是在应用想给对象分配空间却没有足够的内存可用时候抛出的，不足的原因可能是**堆空间不足、也可能是元数据空间不足，并且此时没有任何对象可以被垃圾回收以释放空间或扩容Java堆空间**

有多种途径可以分析OOM错误：  
1、用**jmap**工具、JConsole工具或JVM启动参数-XX:+HeapDumpOnOutOfMemoryError生成Java堆的快照文件，然后利用jhat或VisualVM去分析  
2、利用SA工具**直接链接**到运行程序上去获取堆直方图  
3、利用JVM启动参数-XX:OnOutOfMemoryError在遇到OOM错误时生成core或崩溃快照文件，然后利用SA工具从**core文件**中查看堆直方图



# 虚拟机命令行附加参数

一般来说，只有在需要调整命令行参数的时候才去做参数调整，**不能为了调整而去调整**。Hotspot命令行参数使用有2种格式，第一种是如果参数是**布尔类型**，就在参数前面加上"+"或"-"来表示真假；另一种是个**数值型**的参数或指定的关键词列表使用的，这种格式直接使用=数值

**-XX:+UseG1GC**：启用G1垃圾收集器，在JDK 7和JDK 8中启用G1需要显式使用该参数，在JDK 9后G1成为默认垃圾收集器，取代之前的并行垃圾收集器

**-XX:ConcGCThreads**：设定并发执行GC的Java线程数，默认值近似于整个应用线程数的1/4，减少这个参数可能提升并行回收效率，即提高系统内部吞吐量(系统是一个整体，大家都需要占用CPU资源)，不过如果这个数值过低，也会导致并行回收机制耗时过长

**-XX:G1HeapRegionSize**：设置G1每个Region的大小，默认是内存大小的1/2000，单位MB，且只能是1、2、4、8、16、32中的一个。如果对象大小超过一个Region的一半，那么G1需要特别管理此类大对象。增大这个设置大对象就能正常进入Region，对于拥有很多**大对象**的应用，有助于提高应用性能。同样这样做的缺点是直接干预了各年龄代的分配大小，降低了G1自动分配的灵活性，例如新生代的大小就依赖Region的大小

**-XX:G1MixedGCCountTarget**：设置并行循环之后需要有多少个**混合GC**启动，默认8个。老年代Region的回收时间通常比年轻代的收集时间长一些，所以该参数设置允许更多的混合GC同时回收老年代，同样增加该参数会增大GC启动时间，如果混合GC中断应用的时间太长，可以考虑调大该参数

**-XX:+G1PrintRegionLivenessInfo**：该参数需要和-XX:+UnlockDiagnosticVMOptions配合启动，它们本身就属于VM的调试信息，如果开启了，VM会打印堆内存里每个Region的存活对象信息，该信息包含了对象的具体使用、可记忆化集合大小、GC效率的统计值。这些信息在标记循环结束后和收集集合排序之后可以打印出来，这些信息有助于理解堆内存的具体使用情况，分析**可记忆化集合**的问题。如果在线上应用直接记录每个Region的这些日志信息，堆很大的时候日志数据会套多导致管理不便

**-XX:G1ReservePercent**：是G1为了保留一些空间用于年代之间的提升，默认值是堆空间的10%。注意这个空间保留后就**不会用在年轻代**了，如果应用中有比较大的堆空间，比较多的大对象存活，还是减少一点保留空间，这样会给年轻代更多的预留空间、GC之间更长的间隔时间，对性能提升有些帮助

**-XX:+G1SummarizeRSetStats**：是虚拟机的诊断调试参数，要与-XX:+UnlockDiagnosticVMOptions配合启动，如果开启了JVM退出时打印可记忆化集合的详细统计信息，如果和-XX:G1SummarizeRSetStatsPeriod参数配合使用，统计信息会周期性输出，如果怀疑有与**可记忆化集合**有关的问题，应使用该参数配合分析

**-XX:+G1TraceConcRefinement**：也是虚拟机的诊断调试参数，要与-XX:+UnlockDiagnosticVMOptions配合启动，启动后多线程的并发执行优化的具体信息会被输出到日志信息中。日志会记录多线程**并发执行优化的开始和结束**，有利于分析多线程并发执行优化导致的问题

**-XX:+G1UseAdaptiveConcRefinement**：默认启用，虚拟机会在每次GC之后动态重新计算-XX:G1ConcRefinementGreenZone、-XX:G1ConcRefinementYellowZone和-XX:G1ConcRefinementRedZone3个参数的值

**-XX:GCTimeRatio**：GC在某些阶段是STW的，这个参数就是计算花费在Java应用线程和花费在GC线程上的**时间比率**，默认是9。这个参数主要目的是让用户可以控制花在引用上的时间，如果采用默认的9，即最多10%时间会花在GC工作上，而Parallel GC的默认值是99，即1%的时间被用在GC上，这是因为Parallel GC贯穿整个GC，而G1则是通过Region划分，不需要全局性扫描Java Heap

**-XX:+HeapDumpBeforeFullGC**：让虚拟机在每次FULL GC之前创建**hprof文件**，文件位于虚拟机启动目录中，通过联合使用-XX:+HeapDumpAfterFullGC参数可以比较全的比较GC前后的hprof文件差异

**-XX:+HeapDumpAfterFullGC**：上同，每次FULL GC后创建hprof文件

**-XX:InitiatingHeapOccupancyPercent**：G1内部并行循环启动的设置值，默认为Java Heap的45%，可以理解为**老年代空间占用的空间**，GC收集后需要低于45%的占用率。这个值主要是为了决定什么时间启动老年代的并行回收循环，这个循环从初始化并行回收开始，可以避免FULL GC的发生。如果老年代用尽导致FULL GC，可调低该值使得更早启动并行回收循环，避免可用空间用尽。如果没有FULL GC，那么为了提高吞吐量可以调高该值，减少并发回收循环，G1的并发循环需要占用CPU资源，因此频繁并发回收会降低应用的吞吐量。但是综合来说早启动好于晚启动，其他可参考-XX:+G1UseAdaptiveIHOP

**-XX:+UseStringDeduplication**：告诉虚拟机启用**字符串去重**功能，从JDK 8U20版本开始，默认不启用。字符串去重功能利用字符串内部实际的字符串数组，当一个字符串与另一个equals的时候，字符串对象会被更新共享同一个字符串数组，从而降低内存的使用。这个功能只有G1上有，判断是否需要去重，需要满足对象类型是字符串、从新生代移除迁移到新生代/幸存区并对象存活时间达到去重时间阈值

**-XX:StringDeduplicationAgeThreshold**：设置字符串考虑去重需要满足的**生存时间**，生存时间是该对象存活了多少次垃圾回收，默认为3

**-XX+G1UseAdaptiveIHOP**：是JDK 9提供的新特性，它自适应的调整参数InitiatingHeapOccupancyPercent设定的IHOP值，目的是为了确保不撑爆老年代的前提下，**尽可能晚启动老年代循环回收**以提高系统的吞吐量，该特性在JDK 9中默认启用

**-XX:MaxGCPauseMillis**：设定G1允许的**最大暂停时间目标**，默认200ms。这是个期望值而不是强制规定，所以实际会有超出，该参数尝试通过限制年轻代的大小来实现目标，推荐与-Xms、-Xmx一起使用，这3个参数也是初次使用G1时推荐的调优参数，从并行GC、CMS GC迁移到G1也应如此

**-XX:MinHeapFreeRatio**：设定堆中允许空闲的空间**最小比例**，默认40，即40%。Hotspot虚拟机根据该参数决定是否需要增大堆大小

**-XX:+PrintAdaptiveSizePolicy**：启用堆大小变化的日志输出，有助于理解G1做出的启发式决策

**-XX:+ResizePLAB**：设定是否需要动态调整**线程局部提升缓存**还是使用固定大小，默认为动态调整。有证据证明G1该特性会在某些应用中导致性能问题，可禁用该特性降低G1垃圾回收暂停时间以提高应用性能

**-XX:+ResizeTLAB**：设定是否需要动态调整**线程局部分配缓存**还是使用固定大小，默认为动态调整。绝大多数情况下，动态调整通过减少分配路径的竞争来提高应用性能，对比PLAB，虚拟机调优很少涉及到TLAB，对于大多数应用使用默认值就可以了

**-XX:+ClassUnloadingWithConcurrentMark**：该参数在G1老年代的并发循环回收中**启用对象卸载功能**，默认启动。一般情况下，并发循环回收阶段卸载对象要优于最后FULL GC时才卸载。该阶段卸载对象有时候会增大G1的标记回收次数，如果标记回收次数过大有可能会突破设定的最大回收暂停时间，这种情况下建议禁用该特性

**-XX:+ClassUnloading**：卸载对象开关，默认**启用卸载**功能，虚拟机卸载不再使用的类，禁用后即使是FULL GC，虚拟机也不会卸载不再使用的类

**-XX:+UnlockDiagnosticVMOptions**：设定是否允许使用诊断类参数，默认不允许

**-XX:+UnlockExperimentalVMOptions**：设定是否启用被标记位实验性的虚拟机参数，默认不允许

**-XX:+UnlockCommercialFeatures**：设定是否启用Oracle特性的授权检查，默认不启动

