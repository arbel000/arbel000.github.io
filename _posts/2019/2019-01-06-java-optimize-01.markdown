---
layout:     post
title:      "Java性能优化01-程序优化"
date:       2019-01-06
author:     "ZhouJ000"
header-img: "img/in-post/2019/post-bg-2019-headbg.jpg"
catalog: true
tags:
    - java
    - 优化
--- 

[Java性能优化01-程序优化](https://zhouj000.github.io/2019/01/06/java-optimize-01/)  
[Java性能优化02-并行优化](https://zhouj000.github.io/2019/01/08/java-optimize-02/)  
[Java性能优化03-JVM调优](https://zhouj000.github.io/2019/01/10/java-optimize-03/)  
[Java性能优化04-调优工具](https://zhouj000.github.io/2019/01/11/java-optimize-04/)  



# 字符串

String对象特点：不变性、针对常量池的优化、类的final定义

**substring()方法：**  
在JDK7之前，当substring()方法被调用的时候，它会创建一个新的字符串对象，但是这个字符串的值在java堆中仍然指向的是同一个数组，这两个字符串的不同在于他们的count和offset的值。这种算法提高了运算速度却浪费了大量的内存空间，而且可能会造成内存泄露。JDK7之后，substring()方法在堆中真正的创建了一个新的数组,当原字符数组没有被引用后就被GC回收了

**字符串分割：**  
split()方法简单功能强大，但在性能敏感的系统中频繁使用时，可以使用StringTokenizer类，是JDK提供专门用来处理字符串分割子串的工具类

**字符串查找：**  
与indexOf()方法一样，charAt()方法也具有很高的效率，适合高频调用。当使用startsWith()或endsWith()时都可以用charAt()实现

**StringBuilder和StringBuffer(同步)：**  
在java编译时，对于静态字符串的累加会彻底优化，将多个连接操作字符串合成一个单独的长字符串；而对于变量字符串，会使用StringBuilder进行累加处理。但是实际编码中依旧推荐显示使用StringBuilder或StringBuffer，且如果能预先评估StringBuilder的大小，可以节省扩容的数组复制操作



# 集合

## List

Arraylist采用了数组实现，LinkedList使用了循环双向链表的数据结构

**添加元素到队尾：**  
Arraylist的add()方法性能取决于ensureCapacity()方法，所以只要Arraylist的当前容量足够大，add()操作效率是非常高的，而在需要扩容时，会进行大量的数组复制操作，但是最终调用System.arraycopy()方法，因此效率还是相当高的。LinkedList由于使用链表的结构，因此不需要维护容量的大小，而且追加操作非常方便，然而每次元素的添加都需要新建一个Entry对象，并进行更多的赋值操作，在频繁的系统调用中会有一定性能影响。可得LinkedList对堆内存和GC的要求更高

**新增元素到任意位置：**  
Arraylist是基于数组实现，而数组是一块连续的内存空间，如果在数组任意位置插入元素，必然导致该位置后的所有元素需要重新排列，因此效率会比较低，而且插入元素越靠前，数组重组的开销越大。而LinkedList就显示出了优势，对于LinkedList来说在队尾和任意位置插入元素是一样的，如果需要经常在任意位置插入元素，LinkedList的性能优势非常大

**删除任意位置元素：**  
对于Arraylist来说，remove()方法与add()方法雷同，在任意位置移除后，都要进行数组重组。而在LinkedList中，首先需要通过循环找到要删除的元素，如果在List前半段从前往后找，反之亦然，在有大量元素的情况下效率很低。因此对于Arraylist越尾部删除效率越高，对于LinkedList越两端删除效率越高

**遍历：**  
ForEach循环综合性能稍弱与普通的迭代器，是由于编译器将ForEach循环体作为迭代器处理，但是存在一步多余的赋值操作。而使用for循环随机访问遍历列表时，Arraylist表现良好，而LinkedList的表现无法接受，这是因为LinkedList进行随机访问时，总会进行一次列表的遍历操作

通过RandomAccess接口可以知道List是否支持快速随机访问，基于数组的List都实现了，而基于链表的没有，因此当需要通过索引下标对List进行随机访问时，不要使用LinkedList


## Map

**HashMap、Hashtable：**  
Hashtable大部分方法做了同步，而HashMap没有；Hashtable不允许key或value为null值，而HashMap可以；最后在内部算法上，它们对key的hash算法和hash值到内存索引的映射算法不同。但是总体而言，HashMap、Hashtable与同步的HashMap(使用Collections.synchronizedMap()产生)的性能性差无几

简单地说，HashMap就是将key做hash算法，然后将hash值映射到内存地址，直接去的key所对应的数据。在HashMap中，底层数据结构使用的是数组，所谓的内存地址即数组的下标索引

hash冲突：数组内的元素采用Entry类对象(Node)，每个Entry指向另外一个Entry(超过8个则树形化为红黑树)

影响性能的因素：hash算法，初始大小，负载因子

**LinkedHashMap：**  
HashMap的一个缺点是它的无序性，如果希望元素保存输入时的顺序，需要使用LinkedHashMap。LinkedHashMap可以提供两种类型的顺序：一是元素插入时的顺序，二是最近访问的顺序，可以通过构造函数指定排序行为。内部结构上LinkedHashMap继承了HashMap，在其基础上又在内部增加了一个链表用来存放元素的顺序。实现了LinkedHashMap.Entry，增加了before和after属性用以记录某一表项的前驱和后继，并构成循环链表。在按访问顺序排序模式中，get()方法(同put、remove一样)也会修改链表结构，因此在迭代器访问时使用会抛出异常，需要注意

**TreeMap：**  

TreeMap采用完全不同的Map实现，有更强大的功能，且实现了SortedMap接口，但是性能略微低于HashMap。与LinkedHashMap的排序不同，TreeMap是按照元素的固有顺序排序的(Comparator或Comparable决定)，可以在构造器中传入一个Comparator，也可以使用实现了Comparable接口的key。TreeMap内部实现是基于红黑树的，是一种平衡查找树，性能优于平衡二叉树。如果要将排序功能加入HashMap，建议使用TreeMap而不是自行实现排序，又增加开发成本，又可能成为性能瓶颈


## Set

Set接口没有在Collection接口上增加额外操作，且Set集合内元素是不能重复的，所有的Set实现都是对应的Map的一种封装而已。HashSet对应HashMap，LinkedHashSet对应LinkedHashMap，TreeSet对应TreeMap



# NIO

NIO与旧的基于流的I/O方法相对，表示新的一套Java I/O标准，具有以下特性：  
1、为所有的原始类型提供(Buffer)缓存支持  
2、使用Java.nio.charset.Charset作为字符集编码解码解决方案  
3、增加通道(Channel)对象，作为新的原始I/O对象  
4、支持锁和内存映射文件的文件访问接口  
5、提供了基于Selector的异步网络I/O  
与流式的I/O不同，NIO是基于块(Block)的，它以块作为基本单位处理数据。在NIO中，最重要的两个组件是缓冲Buffer和通道Channel

NIO的实现中，Buffer是一个抽象类，JDK为每一种Java原生类型都创建了一个Buffer。除了ByteBuffer外，其他每一种Buffer都具有完全一样的操作，因为ByteBuffer多用于标准I/O操作的接口。在NIO中和Buffer配合使用的Channel，是一个双向通道，既可读又可写，有点类似Stream，但是Stream是单向的。程序中不能直接堆Channel进行读写操作，必须通过Buffer来进行

Buffer的3个重要参数：位置(position)、容量(capactiy)、上限(limit)，提供了rewind()、clear()、flip()来重置和清空Buffer状态的方法。还提供了标签(mark)，像书签一样随时记录当前位置，然后在任意时刻回来(reset)，同时也可以使用duplicate()进行复制原缓冲区，在处理一些复杂的Buffer数据很有用处，因为新生成的缓冲区和原缓冲区共享相同的内存数据，且任意一方的数据改动都是可见的，只是独立维护了各自的position、limit和mark，加大了代码的灵活性，使用slice()方法进行缓冲区分片，就是创建新的子缓冲区，和父缓冲区共享数据，这有助于将系统模块化。可以使用asReadOnlyBuffer()方法得到一个只读缓冲区，共享数据但保证数据不被修改。NIO还提供了处理结构化数据的方法，称之为散射(Scattering)和聚集(Gathering)，散射将数据读入一组Buffer中而不仅仅是一个，聚集与之相反，将数据写入一组Buffer中

NIO提供了将文件映射到内存的方法进行I/O操作，它比常规基于流的I/O操作快很多，主要由FileChannel.map()方法实现，返回一个MappedByteBuffer，它是ByteBuffer的子类，可以大大提高读取和写入文件的速度



# 引用类型

Java提供了4个级别的引用：强引用、软引用、弱引用和虚引用。这4个级别中，只有强引用FinalReference类是包内可见，其他3种引用类型均为public，可以在程序中直接使用

强引用：可以直接访问目标对象，且强引用所指向的对象在任何时候都不会被系统回收，但是可能会导致内存泄露

软引用：除了强引用外，最强的引用类型。持有软引用的对象，不会被JVM很快回收，JVM会根据当前堆的使用情况来判断何时回收。当堆的使用率临近阀值时才会去回收软引用对象，因此可以用于实现对内存敏感的Cache

弱引用：在系统GC时，只要发现弱引用，不管堆空间是否足够，都会对对象进行回收。但是由于垃圾回收器线程的线程通常级别很低，因此并不一定很快发现持有弱引用的对象。软引用和弱引用都适合保存一些可有可无的缓存数据

虚引用：所有引用类型中最弱的一个。持有虚引用的对象和没有引用几乎是一样的，随时都可能被垃圾回收器回收，当试图通过虚引用get方法取得强引用时会失败，且虚引用必须和引用队列一起使用，它的作用在于跟踪垃圾回收过程

WeakHashMap：是HashMap的一种实现，它使用弱引用作为内部数据的存储方案，可以作为简单的缓存表解决方案。它会在系统内存紧张时使用弱引用，自动释放掉持有弱引用的内存数据，如果存放在WeakHashMap中的key都存在强引用，那么就相当于一个HashMap，所以如果希望使用，不要在其他地方强引用WeakHashMap的key，否则这些key将不会被回收，WeakHashMap也就无法正常释放它们所占用的表项



# 小技巧

1、慎用异常：try-catch语句对系统性能性而言是非常糟糕的，如果被用到循环之中，会给系统带来极大损害

2、位运算代替乘除法：位运算是最为高效的，可以代替部分算术运算来提高系统运行速度，最典型的就是乘除运算

3、代替switch：使用一个连续的数组代替switch的返回值，通过%余运算和下标访问提高性能

4、展开循环：是极端情况下使用的优化手段，加大步长，在循环内做多次同样的操作，减少循环次数可以提高系统性能

5、使用arraycopy()：数组复制的高效API，因为System.arraycopy()是native函数

6、使用Buffer进行I/O操作：除了NIO外，进行I/O操作时，如果合理利用缓冲，可以有效提高I/O的性能

7、使用clone代替new：new关键字在创建轻量级对象时非常快，但是对于重量级对象，由于构造函数可能存在的复杂耗时操作，可能会花费较长时间，使用Object.clone()方法可以绕过对象构造函数，快速复制一个对象实例，默认下clone生成的实例为浅拷贝(普通对象拥有相同的引用)



引用：《Java程序性能优化》
