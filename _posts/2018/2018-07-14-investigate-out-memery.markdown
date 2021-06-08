---
layout:     post
title:      "java项目 内存溢出问题"
date:       2018-07-14
author:     "ZhouJ000"
header-img: "img/in-post/2018/post-bg-2018-headbg.jpg"
catalog: true
tags:
    - 排查
    - 命令
--- 

<font id="last-updated">最后更新于：2018-07-14</font>

[java项目 CPU占用100%问题](https://zhouj000.github.io/2018/06/24/investigate-cpu-100/)  
[java项目 内存溢出问题](https://zhouj000.github.io/2018/07/14/investigate-out-memery/)

[Java性能优化04-调优工具](https://zhouj000.github.io/2019/01/11/java-optimize-04/)  



# JVM内存体系

JVM的基本组成:
+ 指令集:JVM指令集 
+ 类加载器：在jvm启动时或者类在运行时将需要的class加载到JVM中 
+ 执行引擎：负责执行class文件中的字节码指令，相当于CPU。hotspot是一种基于栈的执行引擎
+ 运行时数据区：将内存划分成若干个区，分别完成不同的任务
+ 本地方法区：调用C或C++实现的本地方法代码返回的结果

> JVM执行字节码指令是基于栈的架构的，所有的操作数必须先入栈，然后根据指令的操作码选择从栈顶弹出若干个元素进行计算后再将结果入栈。JVM操作数可以存放在每一个栈帧中的一个本地变量中，即每个方法调用时就会给这个方法分配一个本地变量集，这个本地变量集在编译时就已经确定，所以操作数入栈可以直接是常量或者从本地变量集中娶一个变量压入栈中。JVM基于栈的设计理由是：  
(1)JVM要设计成与平台无关的，而平台无关性就要保证在没有或者由很少的寄存器的机器上也能同样正确执行java代码，因为寄存器很难做到通用  
(2)基于栈的理由是为JVM更好地优化代码而设计的  
(3)为了指令的紧凑性，因为java代码可能在网络上传输，所以class文件的大小也是设计JVM字节码指令的一个重要因素。

![jvm](/img/in-post/2018/7/jvm-model.png)

JVM运行时内存 = 共享内存区 + 线程内存区  

共享内存区 = 持久代 + 堆  
持久代 = 方法区 + 其他  
堆 = Old Space + Young Space  
Young Space = Eden + S0 + S1  

线程内存区 = 单个线程内存 + 单个线程内存 + ...  
单个线程内存 = PC Regster + JVM栈 + 本地方法栈  
JVM栈 = 栈帧 + 栈帧 + ...  
栈帧 = 局域变量区 + 操作数区 + 帧数据区  

![jvm-m](/img/in-post/2018/7/jvm-memory.png)

# 内存溢出问题

**导致OutOfMemoryError异常的常见原因有以下几种:**
+ 内存中加载的数据量过于庞大，如一次从数据库取出过多数据
+ 集合类中有对对象的引用，使用完后未清空，使得JVM不能回收
+ 代码中存在死循环或递归调用，或者循环产生过多重复的对象实体
+ 使用的第三方软件中的BUG
+ 启动参数内存值设定的过小

**运行时数据区内存溢出:**
+ **程序计数器**: 指向当前线程下一条需要执行的字节码指令的地址
	- 不会发生内存溢出
+ **虚拟机栈**: 由栈帧组成、每个栈帧代表一次方法调用，其包含存储变量表、操作数栈和方法出口三个部分，方法执行完成后该栈帧将被弹出。
	- StackOverflowError: 如果请求的栈的深度大于虚拟机所允许的深度，将会抛出这个异常，如果使用虚拟机默认参数，一般达到1000到2000这样的深度没有问题。
		+ 首先栈溢出会输出异常信息，根据信息查看对应的方法调用是否出现无限调用、或者栈帧过大等代码逻辑上的问题，通过修改代码逻辑解决
		+ 如果确实需要更大的栈容量，可以检查并调大栈容量：-Xss16m
	- OutOfMemoryError: 因为除掉堆内存和方法区容量，剩下的内存由虚拟机栈和本地方法栈瓜分，如果剩下的内存不足以满足更多的工作线程的运行、或者不足以拓展虚拟机栈的时候，就会抛出OutOfMemoryError异常。
		+ 首先检查是否创建过多的线程，减少线程数
		+  可以通过“减少最大堆容量”或“减少栈容量”来解决
+ **本地方法栈**: 与虚拟机栈唯一的不同是虚拟机栈执行的是java方法，而本地方法栈执行的是本地的C/C++方法
	- StackOverflowError: 同虚拟机栈
	- OutOfMemoryError: 同虚拟机栈
+ **堆**: 所有线程共享，存放对象实例
	- OutOfMemoryError:Java heap space: 堆中没有足够内存完成实例分配，并且无法继续拓展
		+ 内存泄露检查：首先通过“内存溢出快照 + MAT等分析工具”，分析是否存在内存泄露现象，检查时可以怀疑的点比如集合、第三方库如数据库连接的使用、new关键字相关等
		+ 如果没有内存泄露，那么是内存溢出，所有对象却是都还需要存活，这个时候就只能调大堆内存了：-Xms和-Xmx
+ **方法区**: 所有线程共享，存放已加载的class信息、常量、静态变量和即时编译后的代码	
	- OutOfMemoryError:PermGen space: 方法区没有足够内存完成内存分配存放运行时新加载的class信息
		+ 内存泄露检查：检查是否加载过多class文件(jar文件)，或者重复加载相同的class文件(jar文件)多次
		+ 通过-XX:PermSize=64M -XX:MaxPermSize=128M改大方法区大小
	- OutOfMemoryError: Metaspace
+ **运行时常量池**: 方法区的一部分，存放常量
	- OutOfMemoryError:PermGen space: 方法区没有足够的内存完成内存分配，存放运行时新创建的常量，比如String类的intern()方法，其作用是如果常量池已经包含一个相同的字符串，则返回其引用，否则将此String对象包含的字符串添加到常量池中
		+ 内存泄露检查：检查是否创建过多常量
		+ 通过-XX:PermSize=64M -XX:MaxPermSize=128M改大方法区大小
+ **直接内存**: 不属于JVM运行时数据区，也不是虚拟机规范中定义的内存区域，JDK1.4引入的NIO中包含通道Channel和缓冲区Buffer，应用程序从通道获取数据是先经过OS的内核缓冲区，再拷贝至Buffer，因为比较耗时，所以Buffer提供了一种直接操作操作系统缓冲区的方式，即ByteBuffer.allocateDirector(size)，这个方法返回DirectByteBuffer应用就是指向这个底层存储空间关联的缓冲区，即直接内存(native memory)，或者叫堆外内存
	- OutOfMemoryError: Direct buffer memory: JVM所需内存 + 直接内存 > 机器物理内存(或操作系统级限制)，无法动态拓展
		+ 例如内存占用较高，机器性能骤降，但是通过GC信息或者jstat发现GC很少，通过jmap获得快照分析后也没发现什么异常，而程序中又直接或者间接地用到了NIO，那么和可能就是直接内存泄露
		+ 分析NIO相关的程序逻辑


**内存溢出常见的错误提示:**
+ java.lang.OutOfMemoryError
+ 堆溢出
	- java.lang.OutOfMemoryError: ... java heap space
	- java.lang.OutOfMemoryError:GC over head limit exceeded
+ 栈溢出
	- java.lang.StackOverflowError  
+ 永久代溢出
	- java.lang.OutOfMemoryError: PermGen space  
	- java.lang.OutOfMemoryError: Metaspace
+ 直接内存溢出
	- java.lang.OutOfMemoryError: Direct buffer memory
+ java.lang.OutOfMemoryError: unable to create new native thread
+ java.lang.OutOfMemoryError: request {} byte for {}out of swap
+ java.lang.OutOfMemoryError: GC overhead limit exceeded


#### java.lang.OutOfMemoryError: ... java heap space
看到heap相关的时候就肯定是堆栈溢出了，出现此种问题有可能是内存泄露，也有可能是内存溢出了。如果内存泄露，我们要找出泄露的对象是怎么被GC ROOT引用起来，然后通过引用链来具体分析泄露的原因。如果出现了内存溢出问题，
此时如果代码没有问题的情况下，适当调整-Xmx和-Xms是可以避免的，不过一定是代码没有问题的前提，为什么会溢出呢，要么代码有问题，要么访问量太多并且每个访问的时间太长或者数据太多，导致数据释放不掉，因为垃圾回收器是要找到那些是垃圾才能回收，这里它不会认为这些东西是垃圾，自然不会去回收了。如果内存泄露，我们要找出泄露的对象是怎么被GC ROOT引用起来，然后通过引用链来具体分析泄露的原因。注意这个溢出之前，可能系统会提前先报错关键字为java.lang.OutOfMemoryError:GC over head limit exceeded

##### 内存泄露与内存溢出
> **内存泄漏memory leak**: 是指程序在申请内存后，无法释放已申请的内存空间，一次内存泄漏似乎不会有大的影响，但内存泄漏堆积后的后果就是内存溢出。如果内存泄露，我们要找出泄露的对象是怎么被GC ROOT引用起来，然后通过引用链来具体分析泄露的原因。
**内存溢出 out of memory**: 指程序申请内存时，没有足够的内存供申请者使用，或者说，给了你一块存储int类型数据的存储空间，但是你却存储long类型的数据，那么结果就是内存不够用，此时就会报错OOM,即所谓的内存溢出。 

二者的关系: 
1. 内存泄漏的堆积最终会导致内存溢出
2. 内存溢出就是你要的内存空间超过了系统实际分配给你的空间，此时系统相当于没法满足你的需求，就会报内存溢出的错误。
3. 内存泄漏是指你向系统申请分配内存进行使用(new)，可是使用完了以后却不归还(delete)，结果你申请到的那块内存你自己也不能再访问（也许你把它的地址给弄丢了），而系统也不能再次将它分配给需要的程序。就相当于你租了个带钥匙的柜子，你存完东西之后把柜子锁上之后，把钥匙丢了或者没有将钥匙还回去，那么结果就是这个柜子将无法供给任何人使用，也无法被垃圾回收器回收，因为找不到他的任何信息。
4. 内存溢出：一个盘子用尽各种方法只能装4个果子，你装了5个，结果掉倒地上不能吃了。这就是溢出。比方说栈，栈满时再做进栈必定产生空间溢出，叫上溢，栈空时再做退栈也产生空间溢出，称为下溢。就是分配的内存不足以放下数据项序列,称为内存溢出。说白了就是我承受不了那么多，那我就报错

```
public static void main(String[] args) {
	List<byte[]> buffer = new ArrayList<byte[]>();
	buffer.add(new byte[10*1024*1024]);
}

VM arguments添加：
-verbose:gc -Xmn10M -Xms20M -Xmx20M -XX:+PrintGC

输出：
[GC (Allocation Failure)  1147K->688K(19456K), 0.0010108 secs]
[GC (Allocation Failure)  688K->656K(19456K), 0.0009890 secs]
[Full GC (Allocation Failure)  656K->588K(19456K), 0.0061391 secs]
[GC (Allocation Failure)  588K->588K(19456K), 0.0003490 secs]
[Full GC (Allocation Failure)  588K->576K(19456K), 0.0040967 secs]
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at test.OOMTest.main(OOMTest.java:10)

从运行结果可以看出，gc以后old区使用率为576K，而字节数组为10M，加起来大于了old generation的空间，所以抛出了异常，如果调整-Xms21M,-Xmx21M,那么就不会触发gc操作也不会出现异常了。	


private static class HeapOOMObject {
}
public static void main(String[] args) {
	List<HeapOOMObject> list = new ArrayList<HeapOOMObject>();
	while (true) {
	   list.add(new HeapOOMObject());
	}
}

VM arguments添加：
-verbose:gc -Xmx50m -Xms50m	

[GC (Allocation Failure)  10951K->7678K(49152K), 0.0110251 secs]
[GC (Allocation Failure)  20204K->16493K(49152K), 0.0137474 secs]
[GC (Allocation Failure)  36415K->31703K(49152K), 0.0388041 secs]
[Full GC (Ergonomics)  31703K->28433K(49152K), 0.2993131 secs]
[Full GC (Ergonomics)  36290K->36198K(49152K), 0.3089769 secs]
[Full GC (Allocation Failure)  36198K->36186K(49152K), 0.1685319 secs]
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOf(Unknown Source)
	at java.util.Arrays.copyOf(Unknown Source)
	at java.util.ArrayList.grow(Unknown Source)
	at java.util.ArrayList.ensureExplicitCapacity(Unknown Source)
	at java.util.ArrayList.ensureCapacityInternal(Unknown Source)
	at java.util.ArrayList.add(Unknown Source)
	at test.OOMTest.main(OOMTest.java:15)
```

结论：  
**当对象大于新生代剩余内存的时候，将直接放入老年代，当老年代剩余内存还是无法放下的时候，触发垃圾收集，收集后还是不能放下就会抛出内存溢出异常了。**

#### java.lang.OutOfMemoryError:GC over head limit exceeded
这种情况是当系统处于高频的GC状态，而且回收的效果依然不佳的情况，就会开始报这个错误，这种情况一般是产生了很多不可以被释放的对象，有可能是引用使用不当导致，或申请大对象导致，但是java heap space的内存溢出有可能提前不会报这个错误，也就是可能内存就直接不够导致，而不是高频GC

#### java.lang.StackOverflowError
栈溢出抛出StackOverflowError错误，出现此种情况是因为方法运行的时候栈的深度超过了虚拟机容许的最大深度所致。出现这种情况，一般情况下是程序错误所致的，比如写了一个死递归，就有可能造成此种情况。也可能是-Xss太小了，我们申请很多局部调用的栈针等内容是存放在用户当前所持有的线程中的，线程在jdk 1.4以前默认是256K，1.5以后是1M(仅针对Hotspot VM)，需要设置-Xss的大小。

> 根据《Java 虚拟机规范》中文版：如果线程请求的栈容量超过栈允许的最大容量的话，Java 虚拟机将抛出一个StackOverflow异常；如果Java虚拟机栈可以动态扩展，并且扩展的动作已经尝试过，但是无法申请到足够的内存去完成扩展，或者在新建立线程的时候没有足够的内存去创建对应的虚拟机栈，那么Java虚拟机将抛出一个OutOfMemory 异常。

```
private static int stackDepth = 1;
public static void stackOverflow() {
	stackDepth++;
	stackOverflow();
}
public static void main(String[] args) {
	stackOverflow();
}

Exception in thread "main" java.lang.StackOverflowError
	at test.OOMTest.stackOverflow(OOMTest.java:13)
```

#### java.lang.OutOfMemoryError: PermGen space
Hotspot jvm在JDK1.8之前通过持久带实现了Java虚拟机规范中的方法区，而运行时的常量池就是保存在方法区中的，因此持久带溢出有可能是运行时常量池溢出，也有可能是方法区中保存的class对象没有被及时回收掉或者class信息占用的内存超过了我们配置。
可能在如下几种场景下出现：  
1. 使用一些应用服务器的热部署的时候，我们就会遇到热部署几次以后发现内存溢出了，这种情况就是因为每次热部署的后，原来的class没有被卸载掉。
2. 如果应用程序本身比较大，涉及的类库比较多，或代码中使用了大量的常量，或通过intern注入常量，但是我们分配给持久带的内存（通过-XX:PermSize和-XX:MaxPermSize来设置）比较小的时候也可能出现此种问题。
3. 一些第三方框架，比如spring,hibernate都通过字节码生成技术（比如CGLib）来实现一些增强的功能，这种情况可能需要更大的方法区来存储动态生成的Class文件。

```
public static void main(String... args){
	List<String> list = new ArrayList<String>();    
	while(true){    
		list.add(UUID.randomUUID().toString().intern());    
	}    
}

VM arguments添加：
-verbose:gc -Xmn5M -Xms10M -Xmx10M -XX:MaxPermSize=1M -XX:+PrintGC

输出：
Exception in thread "main" java.lang.OutOfMemoryError: GC overhead limit exceeded
	at java.util.Arrays.copyOfRange(Unknown Source)
	at java.lang.String.<init>(Unknown Source)
	at java.lang.String.substring(Unknown Source)
	at java.util.UUID.digits(Unknown Source)
	at java.util.UUID.toString(Unknown Source)
	at test.OOMTest.main(OOMTest.java:12)
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=1M; support was removed in 8.0

在JDK8之前的HotSpot JVM，存放这些”永久的”的区域叫做“永久代(permanent generation)”。永久代是一片连续的堆空间，在JVM启动之前通过在命令行设置参数-XX:MaxPermSize来设定永久代最大可分配的内存空间，永久代的垃圾收集是和老年代(old generation)捆绑在一起的，因此无论谁满了，都会触发永久代和老年代的垃圾收集。
而在JDK8里面移除了永生代，而对于存放类的元数据的内存大小的设置变为Metaspace参数，可以通过参数-XX:MetaspaceSize 和-XX:MaxMetaspaceSize设定大小，但如果不指定MaxMetaspaceSize的话，Metaspace的大小仅受限于native memory的剩余大小。也就是说永久代的最大空间一定得有个指定值，而如果MaxPermSize指定不当，就会OOM

修改参数为：
-verbose:gc -Xmn5M -Xms10M -Xmx10M -XX:MaxMetaspaceSize=2M -XX:+PrintGC

[GC (Metadata GC Threshold)  556K->504K(9728K), 0.0014136 secs]
[Full GC (Metadata GC Threshold)  504K->432K(9728K), 0.0028918 secs]
[GC (Last ditch collection)  432K->432K(9728K), 0.0001428 secs]
[Full GC (Last ditch collection)  432K->427K(9728K), 0.0026128 secs]
[GC (Metadata GC Threshold)  427K->427K(9728K), 0.0001225 secs]
[Full GC (Metadata GC Threshold)  427K->427K(9728K), 0.0024573 secs]
[GC (Last ditch collection)  427K->427K(9728K), 0.0001207 secs]
[Full GC (Last ditch collection)  427K->427K(9728K), 0.0024402 secs]
Error occurred during initialization of VM
java.lang.OutOfMemoryError: Metaspace
	at sun.misc.Launcher.<init>(Unknown Source)
	at sun.misc.Launcher.<clinit>(Unknown Source)
	at java.lang.ClassLoader.initSystemClassLoader(Unknown Source)
	at java.lang.ClassLoader.getSystemClassLoader(Unknown Source)
```

一个比较好的译文： [OutOfMemoryError系列](https://blog.csdn.net/renfufei/article/details/76350794)

##### 扩展

在JDK7之前的版本，对于HopSpot JVM，interned-strings存储在永久代（又名PermGen），会导致大量的性能问题和OOM错误。

在JDK7 update 4即随后的版本中，提供了完整的支持对于Garbage-First(G1)垃圾收集器，以取代在JDK5中发布的CMS收集器。使用G1，PermGen仅仅在FullGC（stop-the-word,STW）时才会被收集。G1仅仅在PermGen满了或者应用分配内存的速度比G1并发垃圾收集速度快的时候才触发FullGC。而对于CMS收集器，通过开启布尔参数-XX:+CMSClassUnloadingEnabled来并发对PermGen进行收集。对于G1没有类似的选项，G1只能通过FullGC，stop the world,来对PermGen进行收集。

> -XX:MetaspaceSize，class metadata的初始空间配额，以bytes为单位，达到该值就会触发垃圾收集进行类型卸载，同时GC会对该值进行调整：如果释放了大量的空间，就适当的降低该值；如果释放了很少的空间，那么在不超过MaxMetaspaceSize（如果设置了的话），适当的提高该值。
-XX：MaxMetaspaceSize，可以为class metadata分配的最大空间。默认是没有限制的。
-XX：MinMetaspaceFreeRatio,在GC之后，最小的Metaspace剩余空间容量的百分比，减少为class metadata分配空间导致的垃圾收集
-XX:MaxMetaspaceFreeRatio,在GC之后，最大的Metaspace剩余空间容量的百分比，减少为class metadata释放空间导致的垃圾收集

默认情况下，class metadata的分配仅受限于可用的native memory总量。可以使用MaxMetaspaceSize来限制可为class metadata分配的最大内存。当class metadata的使用的内存达到MetaspaceSize时就会对死亡的类加载器和类进行垃圾收集。设置MetaspaceSize为一个较高的值可以推迟垃圾收集的发生。

在JDK8，Native Memory，包括Metaspace和C-Heap。

Native Heap，就是C-Heap。对于64位的JVM，C-Heap的容量=物理服务器的总RAM+虚拟内存-Java Heap-PermGen

类的元数据信息转移到Metaspace的原因是PermGen很难调整。PermGen中类的元数据信息在每次FullGC的时候可能会被收集，但成绩很难令人满意。而且应该为PermGen分配多大的空间很难确定，因为PermSize的大小依赖于很多因素，比如JVM加载的class的总数，常量池的大小，方法的大小等。

此外，在HotSpot中的每个垃圾收集器需要专门的代码来处理存储在PermGen中的类的元数据信息。从PermGen分离类的元数据信息到Metaspace,由于Metaspace的分配具有和Java Heap相同的地址空间，因此Metaspace和Java Heap可以无缝的管理，而且简化了FullGC的过程，以至将来可以并行的对元数据信息进行垃圾收集，而没有GC暂停。

由于类的元数据可以在本地内存(native memory)之外分配,所以其最大可利用空间是整个系统内存的可用空间。这样，你将不再会遇到OOM错误，溢出的内存会涌入到交换空间。最终用户可以为类元数据指定最大可利用的本地内存空间，JVM也可以增加本地内存空间来满足类元数据信息的存储。

注：永久代的移除并不意味者类加载器泄露的问题就没有了。因此，你仍然需要监控你的消费和计划，因为内存泄露会耗尽整个本地内存，导致内存交换(swapping)，这样只会变得更糟。

Metaspace VM利用内存管理技术来管理Metaspace。这使得由不同的垃圾收集器来处理类元数据的工作，现在仅仅由Metaspace VM在Metaspace中通过C++来进行管理。Metaspace背后的一个思想是，类和它的元数据的生命周期是和它的类加载器的生命周期一致的。也就是说，只要类的类加载器是存活的，在Metaspace中的类元数据也是存活的，不能被释放。

之前我们不严格的使用这个术语“Metaspace”。更正式的，**每个类加载器存储区叫做“a metaspace”。这些metaspaces一起总体称为”the Metaspace”。仅仅当类加载器不在存活，被垃圾收集器声明死亡后，该类加载器对应的metaspace空间才可以回收。** Metaspace空间没有迁移和压缩。但是元数据会被扫描是否存在Java引用。

> jmap -clstats :打印类加载器的统计信息(取代了在JDK8之前打印类加载器信息的permstat)。  
jstat -gc :Metaspace的信息也会被打印出来。  
jcmd GC.class_stats:这是一个新的诊断命令，可以使用户连接到存活的JVM，转储Java类元数据的详细统计。 

#### java.lang.OutOfMemoryError: Direct buffer memory
如果直接或间接使用了ByteBuffer中的allocateDirect方法的时候，而不做clear的时候就会出现类似的问题，常规的引用程序IO输出存在一个内核态与用户态的转换过程，也就是对应直接内存与非直接内存，如果常规的应用程序你要将一个文件的内容输出到客户端需要通过OS的直接内存转换拷贝到程序的非直接内存(也就是heap中)，然后再输出到直接内存由操作系统发送出去，而直接内存就是由OS和应用程序共同管理的，而非直接内存可以直接由应用程序自己控制的内存，jvm垃圾回收不会回收掉直接内存这部分的内存。

#### java.lang.OutOfMemoryError: unable to create new native thread
Stack空间不足以创建额外的线程，要么是创建的线程过多，要么是Stack空间确实小了。其实线程基本只占用heap以外的内存区域，也就是这个错误说明除了heap以外的区域，无法为线程分配一块内存区域了，这个要么是内存本身就不够，要么heap的空间设置得太大了，导致了剩余的内存已经不多了，而由于线程本身要占用内存，所以就不够用了。要么增大进程所占用的总内存，或者减少-Xmx或者-Xss来达到创建更多线程的目的。

> 程序创建的线程数超过了操作系统的限制：对于Linux系统，我们可以通过ulimit -u来查看此限制。
> 假如我们一个进程占用了4G的内存，那么通过下面的公式计算出来的剩余内存就是建立线程栈的时候可以用的内存。线程栈总可用内存 = 4G- (-Xmx的值) - (-XX:MaxPermSize的值) - 程序计数器占用的内存

#### java.lang.OutOfMemoryError: request {} byte for {}out of swap
这类错误一般是由于地址空间不够而导致。

#### java.lang.OutOfMemoryError: GC overhead limit exceeded
JDK6新增错误类型，当GC为释放很小空间占用大量时间时抛出；一般是因为堆太小，导致异常的原因，没有足够的内存。需要查看系统是否有使用大内存的代码或死循环，或者通过添加JVM配置-XX:-UseGCOverheadLimit，来限制使用内存。

#### 其他
比如由于物理内存的硬件问题，导致了code cache的错误(在由byte code转换为native code的过程中出现，但是概率极低)，这种情况内存 会被直接crash掉，类似还有swap的频繁交互在部分系统中会导致系统直接被crash掉。JNI的滥用也会导致一些本地内存无法释放的问题，所以尽量避开JNI，这种内存如果没有在被调用的语言内部将内存释放掉(如C语言)，那么在进程结束前这些内存永远释放不掉，解决办法只有一个就是将进程kill掉。socket连接数据打开过多的socket也会报类似：IOException: Too many open files等错误信息。另外GC本身是需要内存空间的，因为在运算和中间数据转换过程中都需要有内存，所以你要保证GC的时候有足够的内存哦，如果没有的话GC的过程将会非常的缓慢。

## 通用解决方案

1. 修改JVM启动参数，直接增加内存。(-Xms，-Xmx等参数) (关注另一篇博客:[JVM的参数与GC])
2. 检查错误日志，查看“OutOfMemory”错误前是否有其它异常或错误
3. 对代码进行走查和分析，找出可能发生内存溢出的位置



# 查找错误原因

虚拟机常出现的问题包括：内存泄露、内存溢出、频繁GC导致性能下降等，导致这些问题的原因可以通过下面虚拟机内存监视手段来进行分析，具体实施时可能需要灵活选择，同时借助两种甚至更多的手段来共同分析。

比如GC日志可以分析出哪些GC较为频繁导致性能下降、是否发生内存泄露。jstat工具和GC日志类似，同样可以查看GC情况、分析是否发生内存泄露。判断发生内存泄露后，可以通过jmap工具和MAT等分析工具的结合查看虚拟机内存快照，分析发生内存泄露的原因。内存溢出快照可以分析出内存溢出发生的原因等。

## GC日志记录

将JVM每次进行GC的情况记录下来，通过观察GC日志可以看出来GC的频度、以及每次GC都回收了哪些区域的内存，根据这些信息为依据来调整JVM相关设置，可以减少Minor GC的频率以及Full GC的次数，还可以判断是否有内存泄露发生。

下面是常见的GC日志输出参数：
+ -verbose.gc：显示GC的操作内容。打开它，可以显示最忙和最空闲收集行为发生的时间、收集前后的内存大小、收集需要的时间等。
+ -XX:+printGCdetails：详细了解GC中的变化。
+ -XX:+PrintGCTimeStamps：了解垃圾收集发生的时间，自JVM启动以后以秒计量。
+ -XX:+PrintHeapAtGC：了解堆的更详细的信息。
+ -Xloggc:[file]：将GC信息输出到单独的文件中


## 内存溢出快照生成

内存溢出时导出溢出的错误信息：`-XX:+HeapDumpOnOutOfMemoryError`

> 此参数只会在第一次输出OOM的时候才会进行堆的dump操作(java heap的溢出是可以继续运行在运行的程序的，至于web应用是否服务要看应用服务器自身如何处理，而c heap区域的溢出就根本没有dump的机会，因为直接就宕机了，目前系统无法看到c heap的大小以及内部变化，要看大小只能间接通过看JVM进程的内存大小（top或类似参数），这个大小一般会大于heap+perm的大小，多余的部分基本就可以认为是c heap的大小了，而看内部变化呢只有google perftools可以达到这个目的)，如果内存过大这个dump操作将会非常长。

指定导出时的路径:  
`-XX:HeapDumpPath=/home/[userName]/logs/oom.dump`
不然导出的路径就是虚拟机的目标位置，默认的文件名是：java_pid<进程号>.hprof  
也可以使用
`jmap -dump:file=....,format=b <pid>`来dump


## jmap 虚拟机内存映像工具

> 监视进程运行中的jvm物理内存的占用情况，该进程内存所有对象的情况，可以让运行中的JVM生成Dump文件，当JVM内存出现问题时可以通过jmap生成快照，分析整个堆。

`jmap -dump:format=b,file=path/heap.bin <PID>`
jmap -dump参数也不要经常用，会导致应用挂起

`jmap -heap <pid>`此命令有时候，会执行卡顿，不建议加监控


命令格式:
```
jmap [ option ] pid
jmap [ option ] executable core
jmap [ option ] [server-id@]remote-hostname-or-IP
```
option参数：
+ heap: 显示Java堆详细信息，GC使用的算法，heap的配置及使用情况，可以用此来判断内存目前的使用情况以及垃圾回收情况
+ histo: 显示堆中对象的统计信息，包括对象数、内存大小等等。jmap -histo:live 这个命令执行，JVM会先触发gc，然后再统计信息
+ permstat: Java堆内存的永久保存区域的类加载器的统计信息。对于每个类加载器而言，它的名称、活跃度、地址、父类加载器、它所加载的类的数量和大小都会被打印。此外，包含的字符串数量和大小也会被打印。
+ finalizerinfo: 显示在F-Queue队列等待Finalizer线程执行finalizer方法的对象
+ dump: 生成堆转储快照(dump堆到文件,format指定输出格式，live指明是活着的对象,file指定文件名)
+ F: 当-dump没有响应时，强制生成dump快照。不支持live子选项。


## jstat 虚拟机统计信息监控工具

> 实时监视虚拟机运行时的类装载情况、各部分内存占用情况、GC情况、JIT编译情况等。对Java应用程序的资源和性能进行实时的命令行的监控的一个极强的监视内存工具

```
每隔250ms查询一次当前运行的java进程(本地虚拟机进程ID)的垃圾收集情况，查询50次
jstat –gc <VMID> 250 50

jstat -gcutil <VMID>
```

命令格式：  
`jstat [options] VMID [interval] [count]`

参数：
+ -class: 类加载的行为统计。监视类装载、卸载数量、总空间以及类装载所耗费时间
+ -gc: 垃圾回收堆的行为统计
+ -gccapacity: 监视内容与-gc相同，各个垃圾回收代容量(young,old,perm)和他们相应的空间统计。
+ -gcutil: 监视内同与-gc相同，输出主要关注堆各个区域已使用空间所占总空间百分比
+ -gcnew: 新生代行为统计
+ -gcnewcapacity: 新生代与其相应的内存空间的统计
+ -gcold: 年老代和永生代行为统计
+ -gcoldcapacity: 年老代行为统计
+ -gcpermcapacity: 永生代行为统计
+ -compiler: HotSpt JIT编译器行为统计
+ -printcompilation: HotSpot编译方法统计


#### 其他

JVM自带监控工具：
+ jinfo: 实时查看虚拟机的各项参数信息。jps –v可以查看虚拟机在启动时被显式指定的参数信息，但是如果想知道默认的一些参数信息，jinfo就显得很重要了。甚至支持在运行时修改部分参数。
	- jinfo -flags <process_id>: 查看jvm的参数
	- jinfo -sysprops <process_id>:  查看java系统参数
	- java -XX:+PrintFlagsFinal -version | grep manageable: 虚拟机的这些参数可以通过下面的命令查看
	- jinfo -flag +PrintGC <process_id>, jinfo -flag +PrintGCDetails <process_id>: 打开GC日志的打印
	- jinfo -flag -PrintGC <process_id>, jinfo -flag -PrintGCDetails <process_id>: 关闭GC日志的打印
+ jhat: 用来分析dump文件的一个微型的HTTP/HTML服务器，它能将生成的dump文件生成在线的HTML文件，让我们可以通过浏览器进行查阅，然而实际中我们很少使用这个工具，因为一般服务器上设置的堆、栈内存都比较大，生成的dump也比较大，直接用jhat容易造成内存溢出，而是我们大部分会将对应的文件拷贝下来，通过其他可视化的工具进行分析。



## 分析工具

使用上述HeapOOMObject的例子，引起Java heap space，导出dump文件进行分析。

#### MAT(Eclipse插件)

1. Eclipse Marketplace下载Memory Analyzer插件
(JVM内存有多大dump文件就多大，而本地分析的时候内存也需要那么大，所以有时生产下载到本地的无法启动是很正常的)

2. Open Perspective选择Memory Analyzer窗口，打开dump文件。

![jvm](/img/in-post/2018/7/MAT1.png)

1. Histogram可以列出内存中的对象，对象的个数以及大小。
2. Dominator Tree可以列出那个线程，以及线程下面的那些对象占用的空间。
3. Top consumers通过图形列出最大的object。
4. Leak Suspects通过MA自动分析泄漏的原因。

![jvm](/img/in-post/2018/7/MAT2.png)

![jvm](/img/in-post/2018/7/MAT3.png)

> Shallow Heap 为对象自身占用的内存大小，不包括它引用的对象。Retained Heap 为当前对象大小 + 当前对象可直接或间接引用到的对象的大小总和。

右键出来选中List Objects,得到的结果再右键选中"Paths to GC Roots",我们可以通过它快速找到GC ROOT.如果存在GC ROOT，它就不会被回收。

![jvm](/img/in-post/2018/7/MAT4.png)

参考： [内存泄露分析之MAT工具使用](https://blog.csdn.net/yincheng886337/article/details/50524890)


#### jconsole.exe

> jvisualvm同jconsole都是一个基于图形化界面的、可以查看本地及远程的JAVA GUI监控工具，可以认为jvisualvm是jconsole的升级版


#### VisualVM/jvisualvm.exe 

![jvm](/img/in-post/2018/7/jvisualvm1.png)

> 一个主机如果希望支持远程监控，需要在启动时添加以下参数：  
-Dcom.sun.management.jmxremote.port=1099  
-Dcom.sun.management.jmxremote.authenticate=false  
-Dcom.sun.management.jmxremote.ssl=false

jvisualvm分为几个选项卡：
+ 概述
+ 监视: 堆dump 可以dump一个内存映射文件，可以通过"文件"->"装入"来进行分析
+ 线程: 线程dump 会直接把线程信息dump到本地，相当于调用了jstack命令
+ 抽样器
+ Profiler


#### Jprofiler

1. ieda: Settings – plugins – Browse repositories 搜索并安装插件
2. 安装JProfiler监控软件
3. ieda: Settings – Tools – JProflier – JProflier executable配置

扩展：  
[Intellij IDEA集成JProfiler性能分析神器](https://blog.csdn.net/wytocsdn/article/details/79258247)  
[使用Jprofiler+jmeter进行JVM性能调优](https://www.cnblogs.com/HEWU10/p/5580758.html)
