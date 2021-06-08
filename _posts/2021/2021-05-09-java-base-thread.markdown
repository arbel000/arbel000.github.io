---
layout:     post
title:      "Java基础SE(三) 线程与并发"
date:       2021-05-09
author:     "ZhouJ000"
header-img: "img/in-post/2021/post-bg-2021-headbg.jpg"
catalog: true
tags:
    - java
--- 


[Java基础SE(一) 数据类型与关键字](https://zhouj000.github.io/2021/04/11/java-base-base/)  
[Java基础SE(二) 集合](https://zhouj000.github.io/2021/02/26/java-base-collections/)  
[Java基础SE(三) 线程与并发](https://zhouj000.github.io/2021/05/09/java-base-thread/)  
[Java基础SE(四) IO](https://zhouj000.github.io/2021/04/21/java-base-io/)  

[Java基础: JVM(九) Java内存模型](https://zhouj000.github.io/2019/07/09/java-base-jmm/)  
[Java性能优化02-并行优化](https://zhouj000.github.io/2019/01/08/java-optimize-02/)  



# 线程

进程是操作系统调度和分配资源的基本单位，进程之间的通信需要通过专门的系统机制，比如消息、socket和管道来完成。而线程是比进程更小的执行单位，每个线程拥有自己的栈和寄存器等资源数据，多个线程之间共享进程的代码、数据和文件

线程的优点如下：  
1、创建一个线程比创建一个进程的代价要小  
2、线程的切换比进程间的切换代价小  
3、充分利用多处理器  
4、数据共享(数据共享使得线程之间的通信比进程间的通信更高效)  

在Java中创建线程有3种方式：  
1、继承Thread类  
2、实现Runnable接口  
3、实现Callable接口

守护线程，可以在创建线程后将其`setDaemon(true)`即可。比如JVM的垃圾回收。内存管理都是守护线程等，当所有线程退出后，守护线程也将结束

## 线程状态

![thread_status](/img/in-post/2021/05/thread_status.jpg)

线程状态可以分为：  
1、**新建**(new)  
2、**就绪状态**/可运行状态(Runnable)  
3、**运行状态**(Running)  
4、**阻塞状态**(Blocked)：等待阻塞(wait进入等待队列)、同步阻塞(同步锁被占用进入锁池)、其他阻塞(sleep/join/IO阻塞)  
5、**死亡状态**(Dead)

其中sleep方法会休眠一段时间(进入阻塞)，但**不释放**对象锁，到时间后进入就绪状态，并不保证能马上运行。`yield`方法类似让出资源(cpu时间片)，让优先级更高的线程执行，自己不会阻塞而是进入**就绪**状态，让系统的线程调度器重新调度器重新调度一次，因此也有可能之后还是自己执行。`wait`方法则会让线程进入阻塞状态，当前线程释放对象锁，进入等待队列，直到有线程`notify/notifyAll`唤醒。而当一个线程需要等另一个(或多个)线程先执行完再继续执行时可以使用join方法，当前线程阻塞，但不释放对象锁。使用interrupt方法可以中断线程，将"中断标记"设置为true，如果该线程正处于阻塞状态则将"中断标记"立即清除为"false"并抛出InterruptedException异常。`interrupted`方法判断当前线程是否处于中断状态，返回后会清除中断状态，而`isInterrupted`方法则只是判断中断状态并不会清除


java调度算法：  
1、选择当前可运行线程中优先级最高的线程运行(机会大)  
2、拥有同样优先级的线程：采用round-robin的方式  
PS、java是preemptive抢占式的，但不同平台的实现并不一定保证这点  
3、sleep休眠/wait/notify/notifyAll  
4、让步yield：当前运行线程让出CPU资源  
5、合并join


# 多线程

并发(concurrency)和并行(parallellism)都是完成多任务更加有效率的方式，但还是有一些区别的：  
1、并发：**交替**做不同事情的能力，**CPU时间片**执行，体现在不同的代码块交替执行，强调在一个时间段内同时执行  
2、并行：**同时**做不同事情的能力，**多核**执行，体现在不同的代码块同时执行

对于多线程执行，如果没有共享变量，那大家都相安无事，相当于各个单线程执行，线程安全。而如果多个线程间有共享变量，又由于Java内存模型的通信机制，就会发生数据不同步、死锁等问题

+ 上下文切换
	- 线程从运行状态切换到阻塞状态或者等待状态的时候需要将线程的运行状态保存，线程从阻塞状态或者等待状态切换到运行状态的时候需要加载线程上次运行的状态
	- 线程的运行状态从保存到再加载就是一次上下文切换，而上下文切换的开销是非常大的，而我们知道CPU给每个线程分配的时间片很短，通常是几十毫秒(ms)，那么线程的切换就会很频繁
+ 死锁
+ 资源限制的挑战
	- 计算机硬件资源或软件资源限制了多线程的运行速度

同步方式：
+ synchronized方法/代码块
+ ReentrantLock
+ ThreadLocal
+ volatile关键字
+ concurrent.atomic包下原子变量
+ wait/notifyAll

并发的2个关键问题：线程间如何通信和线程间如何同步：  
1、命令式编程中通信有2种：共享内存和消息传递。Java的并发采用共享内存，线程间通信总是隐式进行  
2、同步指程序中用于控制不同线程间操作发生相对顺序的机制，在共享内存并发模型中，同步是显式进行的，必须显式指定某个方法或代码需要在线程之间互斥执行


#### 伪共享

为了解决计算器中主内存与CPU之间运行速度差的问题，会在CPU与主内存中添加一级或多级高速缓冲存储器(cache)(L1/L2)，这个cache是集成在CPU内部的，所以也叫CPU Cache。在Cache内部是按行存储的，其中一行成为一个Cache行，其是Cache与主内存进行交换数据的单位，Cache行的大小一般为2的幂次数字节。当CPU访问某个变量时，会先去CPU Cache查看，如果没有再到主内存获取，并将该变量所在内存区域的一个Cache行大小的内存复制进Cache中，由于存放在Cache行的是内存块而不是变量，所以可能有多个变量存储到一个Cache行中。如果多个线程同时修改一个缓存行里的多个变量时，由于同时只能有一个线程操作缓存行，所以相比将每个变量放到一个缓存行，性能就会有所下降，这就是伪共享

伪共享的产生原因是多个变量放在一个缓存行，这在单线程下对代码执行是更有利的，执行更快，多线程下相反。JDK 8之前是通过字节填充的方式解决的，创建一个变量时使用填充字段填充其缓存行，避免同一个缓存行中有多个变量。在JDK 8提供了一个sun.misc.Contended注解来解决伪共享问题，需要注意的是这个注解默认只使用于Java核心类，比如rt包下的类，我们自己创建的类要使用则需要`-XX:-RestrictContended`自定义宽度


## CAS

CAS即compare and swap，是JDK提供的非阻塞原子性操作，它通过硬件保证了比较 —— 更新操作的原子性。Unsafe类中有3个`compareAndSwap*`方法。JDK的rt.jar包的Unsafe类提供了硬件级别的原子性操作，其中的方法都是native方法，使用JNI的方式访问本地C++实现库，提供的这些功能包括裸内存的申请/释放/访问，低层硬件的atomic/volatile支持，创建未初始化对象等。扩展了Java语言表达能力，便于在高层Java代码里实现原本在更底层C中实现的核心库功能。许多广泛使用的高性能开发库都是基于Unsafe类开发，比如 Netty、Hadoop、Kafka 等

虽然Unsafe很强大，可以直接操作内存，但是由于安全性考虑无法直接获取，但是可以通过反射拿到
```java
@CallerSensitive
public static Unsafe getUnsafe() {
	Class var0 = Reflection.getCallerClass();
	// 是不是Bootstrap类加载器加载的
	if (!VM.isSystemDomainLoader(var0.getClassLoader())) {
		throw new SecurityException("Unsafe");
	} else {
		return theUnsafe;
	}
}
```

扩展：  
[Unsafe 相关整理](https://www.jianshu.com/p/2e5b92d0962e)  

CAS操作是乐观锁，每次不加锁而是假设没有冲突去完成某项操作，如果因为冲突失败就重试，直到成功为止。而synchronized就是悲观锁，线程一旦得到锁，其他需要锁的线程会挂起。尽管JDK 1.6为Synchronized做了优化，增加了从偏向锁到轻量级锁再到重量级锁的过度，但是在最终转变为重量级锁之后，性能仍然较低

CAS机制当中使用了3个基本操作数：内存地址V，旧的预期值A，要修改的新值B。更新一个变量的时候，只有当变量的预期值A和内存地址V当中的实际值相同时，才会将内存地址V对应的值修改为B。无论哪种情况，都会在CAS指令之前返回该位置的值(一些特殊情况下将仅返回CAS是否成功)

CAS的问题：  
1、在并发量比较高的情况下，如果许多线程反复尝试更新某一个变量，却又一直更新不成功，循环往复，会给CPU带来很大的压力  
2、只保证一个共享变量的原子性操作，不能保证代码块/多个共享变量的原子性  
3、ABA问题(解决思路：使用版本号)


## 原子操作类

原子操作，有针对CPU指令级别的，和针对高级语言级别的。原子操作实质并不是不可分割，本质在于多个资源之间有一致性的要求，操作的中间态对外不可见

concurrent包中提供了一些原子类，在其atomic包下，例如AtomicInteger、AtomicLong、AtomicIntegerArray、AtomicReference、LongAdder等

```java
// AtomicInteger
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;
private volatile int value;

static {
	try {
		valueOffset = unsafe.objectFieldOffset
			(AtomicInteger.class.getDeclaredField("value"));
	} catch (Exception ex) { throw new Error(ex); }
}

public final int getAndSet(int newValue) {
	return unsafe.getAndSetInt(this, valueOffset, newValue);
}

public final boolean compareAndSet(int expect, int update) {
	return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```
由此可见，内部就是使用了volatile + CAS保证的。根据对象和偏移量获取当前值，与希望的值比较，相等的话进行更新操作

```java
// AtomicIntegerArray
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final int base = unsafe.arrayBaseOffset(int[].class);
private static final int shift;
private final int[] array;

static {
	int scale = unsafe.arrayIndexScale(int[].class);
	if ((scale & (scale - 1)) != 0)
		throw new Error("data type scale not a power of two");
	shift = 31 - Integer.numberOfLeadingZeros(scale);
}

private long checkedByteOffset(int i) {
	if (i < 0 || i >= array.length)
		throw new IndexOutOfBoundsException("index " + i);
	return byteOffset(i);
}

private static long byteOffset(int i) {
	return ((long) i << shift) + base;
}

public final int getAndSet(int i, int newValue) {
	return unsafe.getAndSetInt(array, checkedByteOffset(i), newValue);
}
```
数组的略有不同，因为要计算出每个元素正确的偏移量

JDK 8新增的原子操作类LongAdder等。对于以前非阻塞的原子性操作类(AtomicLong等)在高并发下大量线程竞争更新同一个原子变量，不断循环尝试CAS，浪费CPU性能。新增的原子性递增或者递减类LongAdder用来克服在高并发下使用AtomicLong的缺点

LongAdder类与AtomicLong类的区别在于高并发时前者将对单一变量的CAS操作分散为对数组cells中多个元素的CAS操作，取值时进行求和；而在并发较低时仅对base变量进行CAS操作，与AtomicLong类原理相同

```java
transient volatile Cell[] cells;
transient volatile long base;

// Contended解决伪共享
@sun.misc.Contended static final class Cell {
	volatile long value;
	Cell(long x) { value = x; }
	final boolean cas(long cmp, long val) {
		return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
	}

	// Unsafe mechanics
	private static final sun.misc.Unsafe UNSAFE;
	private static final long valueOffset;
	static {
		try {
			UNSAFE = sun.misc.Unsafe.getUnsafe();
			Class<?> ak = Cell.class;
			valueOffset = UNSAFE.objectFieldOffset
				(ak.getDeclaredField("value"));
		} catch (Exception e) {
			throw new Error(e);
		}
	}
}
```

修改值：
```java
public void add(long x) {
	Cell[] as; long b, v; int m; Cell a;
	// 并发量不大时，直接CAS，与AtomicLong一样
	if ((as = cells) != null || !casBase(b = base, b + x)) {
		boolean uncontended = true;
		// 当竞争激烈到一定程度无法对base进行累加操作时，会对cells数组中某个元素进行更新
		if (as == null || (m = as.length - 1) < 0 ||
			(a = as[getProbe() & m]) == null ||
			!(uncontended = a.cas(v = a.value, v + x)))
			// 用一个死循环对cells数组中的元素进行操作：
			// 当要更新的位置的元素为空时插入新的cell元素，否则在该位置进行CAS的累加操作
			// 如果CAS操作失败并且数组大小没有超过核数就扩容cells数组当要更新的位置的元素为空时插入新的cell元素
			// 否则在该位置进行CAS的累加操作，如果CAS操作失败并且数组大小没有超过核数就扩容cells数组
			longAccumulate(x, null, uncontended);
	}
}

// 对Striped64的BASE的值(base值所对应的内存偏移量)进行累加并返回是否成功，
final boolean casBase(long cmp, long val) {
	return UNSAFE.compareAndSwapLong(this, BASE, cmp, val);
}
```
取值：
```java
public long longValue() {
	return sum();
}

// 返回的是base和cells数组中所有元素的和
public long sum() {
	Cell[] as = cells; Cell a;
	long sum = base;
	if (as != null) {
		for (int i = 0; i < as.length; ++i) {
			if ((a = as[i]) != null)
				sum += a.value;
		}
	}
	return sum;
}
```

总结：底层就是volatile(内存可见性)和CAS(原子性)共同作用的结果

## 锁

锁分类：
+ 按处理方式
	- 乐观锁：相对于悲观锁而言，采取了更加宽松的加锁机制
		+ CAS
		+ 版本号控制
	- 悲观锁：具有强烈的独占和排他特性
		+ 关系型数据库的行锁，表锁，读锁，写锁
		+ synchronized
+ 按抢占方式
	- 公平锁
		+ 队列FIFO
	- 非公平锁
		+ 默认ReentrantLock对象
+ 按持有
	- 独占锁：排他锁，写锁，不能与其他锁并存
	- 共享锁：读锁，可共享一把锁访问数据，但是不能修改

选择因素：
+ 响应效率
+ 冲突频率
+ 重试代价

### synchronized

+ 可见性：语义保证了共享变量的可见性
	- 线程加锁前：需要将工作内存清空，从而保证了工作区的变量副本都是从主存中获取的最新值
	- 线程解锁前；需要将工作内存的变量副本写回到主存中
+ 有序性：保证了有序性
	- 使用阻塞的同步机制，共享变量只能同时被一个线程操作，所以JMM不用像volatile那样考虑加内存屏障去保证synchronized多线程情况下的有序性，因为CPU在单线程情况下是保证了有序性的
+ 原子性：保证了原子性
	- 使用阻塞的同步机制，共享变量加锁了，在对共享变量进行读/写操作的时候是原子性的

**synchronized同步代码块**是通过加`monitorenter`和`monitorexit`指令实现的
	
> 每个对象都有个监视器锁(monitor)，当monitor被占用的时候就代表对象处于锁定状态，而monitorenter指令的作用就是获取 monitor的所有权(计数为1，锁重入+1)，monitorexit的作用是释放monitor的所有权(计数-1直到0)
	
**synchronized同步方法**，常量池中比普通的方法多了个`ACC_SYNCHRONIZED`标识，JVM就是根据这个标识来实现方法的同步。当调用方法的时候，调用指令会检查方法是否有ACC_SYNCHRONIZED标识，有的话线程需要先获取monitor，获取成功才能继续执行方法，方法执行完毕之后，线程再释放monitor，同一个monitor同一时刻只能被一个线程拥有

所以，synchronized同步代码块是需要JVM通过字节码显式的去**获取和释放monitor**实现同步；而synchronized同步方法只是检查`ACC_SYNCHRONIZED`标志是否被设置，不需要JVM去显式的实现。实际上这两个同步方式实际都是通过获取monitor和释放monitor来实现同步的，而monitor的实现依赖于底层操作系统的**mutex**互斥原语，而操作系统实现线程之间的切换的时候需要从用户态转到内核态，这个转成过程开销比较大

![monitorout](/img/in-post/2021/05/monitorout.jpg)

线程尝试获取monitor的所有权，如果获取失败说明monitor被其他线程占用，则将线程加入到的同步队列中，等待其他线程释放monitor，当其他线程释放monitor后，有可能刚好有线程来获取monitor的所有权，那么系统会将monitor的所有权给这个线程，而不会去唤醒同步队列的第一个节点去获取，所以synchronized是**非公平锁**。如果线程获取monitor成功则进入到monitor中，并且将其进入数+1


### 锁膨胀

+ 对象
	- 对象头
		+ Mark Word(存储对象自身的运行时数据)：结构不是固定的
			- 哈希吗
			- GC分代年龄
			- 对象分代年龄
			- 锁状态标志
			- 线程持有的锁
			- 偏向线程ID
			- 偏向时间戳
		+ 类型指针
		+ 数组长度(只有数组对象才有)
	- 实例数据：对象的实际数据，即程序中定义的各种类型的字段内容
	- 对齐填充：对齐方式为8字节整数倍对齐

![markword](/img/in-post/2021/05/markword.jpg)

在32位虚拟机下，`Mark Word`的结构和数据可能为以下5种中的一种：
![mw32](/img/in-post/2021/05/mw32.jpg)

在64位虚拟机下，`Mark Word`的结构和数据可能为以下2种中的一种：
![mw64](/img/in-post/2021/05/mw64.jpg)

总结：是否偏向锁和锁标志位这两个标识和synchronized的锁膨胀息息相关

java虚拟机的实现中每个对象都有一个对象头，用于保存对象的系统信息。对象头中有一个mark word的部分，它是实现锁的关键。在32位系统中，它为32位的数据；在64位系统中，它为64位的数据。它是一个多功能的数据区，可以存放对象的哈希值、对象年龄、锁的指针等信息。一个对象是否占用锁，占有哪个锁，就记录在这个mark word中

以32位系统为例，普通对象的对象头比如：  
`hash:25 ------------>| age:4 biased_lock:1 lock:2`  
它表示mark word中有25位比特表示对象的哈希值，4位比特表示对象的年龄，1位比特表示**是否偏向锁**，2位比特表示**锁的信息**

当对象处于普通的**未锁定**状态时，格式为:  
`[header      |0|01] unlocked`  
前29位表示对象的哈希值、年龄等信息。倒数第3位位为0，最后两位为01，表示未锁定，与偏向锁最后2位相同，因此虚拟机通过**倒数第3位**比特来区分是否为偏向锁

**偏向锁**是程序没有竞争时、取消之前已经获得锁的线程同步操作，即某一锁被线程获取后，进入偏向模式，CAS操作在对象头存储锁偏向的**线程ID**，该线程再次请求这个锁时，无需进行相关同步操作(CAS操作来加锁和解锁)，只需简单的测试一下对象头的Mark Word里是否存储着指向当前线程的偏向锁。对于偏向锁的对象，其对象头记录获得锁的线程：  
`[javaThread* | epoch | age |1|01]`  
前23位表示持有偏向锁的线程，后续2位表示偏向锁的时间戳，4位比特表示对象年龄，年龄后1位为1表示偏向锁，最后2位位01表示可偏向/未锁定。当获得锁的线程再次尝试获取锁时，通过mark word的线程信息可以判断当前线程是否持有偏向锁。当然在锁竞争激烈的场景下反而得不到优化，可以使用`-XX:-UseBiasedLocking`禁用偏向锁

![basicObjectLock](/img/in-post/2021/05/basicObjectLock.png)

偏向锁失败后，java虚拟机让线程申请**轻量级锁**。轻量级锁在虚拟机内部使用一个称为**BasicObjectLock**的对象实现，这个对象内部由一个**BasicLock对象**和一个**持有该锁的Java对象指针**组成。BaseicObjectLock对象放置在Java栈的栈帧中，在BasicLock对象内部还维护着**displaced_header**字段，它用于备份对象头部的**mark word**。当对象处于轻量级锁定时其头部mark word为:  
`[ptr         |00] locked`  
末尾2位为00。此时整个mark word为指向BasicLock对象的**指针**，由于BasicObjectLock对象在线程栈中，因此该指针必然**指向持有该锁的线程栈空间**，即存放在获得锁的线程栈中的该对象真实对象头。当需要判断某一线程是否持有该对象锁时，也只需简单判断**对象头的指针**是否在当前线程的栈地址范围即可。同时BasicLock对象的displaced_header备份了原对象的mark word内容。BasicObjectLock对象的obj字段则指向该对象
```
markOop mark = obj -> mark();
lock -> set_displaced_header(mark);
if (mark == (markOop)Atomic::cmpxchg_ptr(lock, ojb()->mark_addr(), mark)) {
	TEVENT(slow_enter: release stacklock);
	return;
}
```
首先BasicLock通过set_displaced_header方法备份原对象的mark word。然后使用CAS操作尝试将BasicLock的地址复制到对象头的mark word，如果复制成功则枷锁成功，如果失败则认为加锁失败，那么轻量级锁有可能膨胀为重量级锁

![weight-lock](/img/in-post/2021/05/weight-lock.jpg)

+ Mark Word：默认存储对象的HashCode，分代年龄和锁标志位信息。它会根据对象的状态复用自己的存储空间
+ Klass Point：对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例

当轻量级锁失败，虚拟机就会使用**重量级锁**，当对象处于重量级锁定时对象的mark word为：  
`[ptr         |10] monitor`  
末尾2位为10。整个mark word表示**指向monitor对象的指针**，在轻量级锁处理失败后，虚拟机会执行以下操作：  
```
lock -> set_displaced_header(markOopDesc::unused_mark());
ObjectSynchronizer::inflate(THREAD, obj())-> enter(THREAD);
```
首先**废弃**前面BasicLock备份的对象头信息，然后正式启用重量级锁。启用过程分为两步：首先通过`inflate()`方法进行**锁膨胀**，其目的是获得对象的**ObjectMonitor**，然后使用`enter()`方法尝试**进入该锁**。在`enter()`方法调用中，线程很可能会在操作系统层面被挂起，如果这样，线程间切换和调度的成本会比较高。因此锁膨胀后虚拟机会做最后的争取，希望线程可以快速进入**临界区**而避免被操作系统挂起，一种较为有效的方式就是使用**自旋锁**。自旋锁可以使线程在没有获得锁时不被挂起，而转去执行一个空循环，在若干空循环后线程如果可以获得锁则继续执行，如果依旧不能获取才会挂起。JDK1.7以后不用设置参数，默认启用，且自选次数由虚拟机自行调整

![spz](/img/in-post/2021/05/spz.jpg)

重量级锁的实现是基于底层操作系统的mutex互斥原语的，这个开销是很大的。所以JVM对synchronized做了优化，JVM先利用对象头实现锁的功能，如果线程的竞争过大则会将锁升级(膨胀)为重量级锁，也就是使用monitor对象

![sch2](/img/in-post/2021/05/sch2.jpg)
![sch3](/img/in-post/2021/05/sch3.jpg)

![sdb](/img/in-post/2021/05/sdb.jpg)

扩展：  
[深入分析synchronized原理和锁膨胀过程(二)](https://ddnd.cn/2019/03/22/java-synchronized-2/)  
[深入分析Synchronized原理](https://www.cnblogs.com/aspirant/p/11470858.html)


#### 锁分离

在某些情况下，可以对一组独立对象上的锁进行分解，这种情况称为锁分段。例如JDK 7的ConcurrencyHashMap是有一个包含16个锁的数组实现，每个锁保护所有散列桶的1/16，其中第N个散列桶由第(N mod 16)个锁来保护。假设所有关键字都均匀分布，那么相当于把锁的请求减少到原来的1/16

锁分段的劣势在于，与采用单个锁来实现独占访问相比，要获取多个锁来实现独占访问将更加困难并且开销更高，比如计算size、重hash


### AQS

抽象同步队列AQS(锁的底层支持/条件变量的支持/自定义同步器)。AbstractQueuedSynchronizer主要功能分为两类：独占功能和共享功能

Node为AQS维护的一个双向链表队列，存储了当前线程，节点类型，队列中前后元素的变量，还有个waitStatus的变量描述节点的状态
```java
// 内部类Node
static final class Node {
	/** Marker to indicate a node is waiting in shared mode */
	static final Node SHARED = new Node();
	/** Marker to indicate a node is waiting in exclusive mode */
	static final Node EXCLUSIVE = null;

	volatile int waitStatus;
	volatile Node prev;
	volatile Node next;
	volatile Thread thread;
	Node nextWaiter;
}
```
并发时，会有许多Node节点，每个节点代表了一个线程的状态，有的线程放弃了竞争，有的线程在等待条件满足后才恢复执行。一共有4种状态：  
1、节点取消(CANCELLED:1)  
2、节点等待触发(SIGNAL:-1，只有这个状态当前节点才能被挂起)  
3、节点等待条件满足(CONDITION:-2)  
4、节点状态需要向后传播(PROPAGATE:-3)

获取：
```java
/**
 * The synchronization state.
 */
private volatile int state;

public final void acquire(int arg) {
	// 尝试获取
	if (!tryAcquire(arg) &&
		// 1. addWaiter创建独占锁节点(线程+节点类型)，加入到队列尾部，如果有并发则自旋去修改
		// 2. acquireQueued将线程挂起
		acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
		// 中断
		selfInterrupt();
}

// 留给子类实现，由于有两类功能，因此非抽象方法让所有子类都去实现
// 子类可通过state判断
protected boolean tryAcquire(int arg) {
	throw new UnsupportedOperationException();
}

final boolean acquireQueued(final Node node, int arg) {
	boolean failed = true;
	try {
		boolean interrupted = false;
		for (;;) {
			// 获取node的前一个节点p
			final Node p = node.predecessor();
			// 如果节点p是头节点，说明是队列中第一个"有效的"节点，因此尝试获取tryAcquire
			if (p == head && tryAcquire(arg)) {
				// 成功后将自己设为头节点，清空thread和prev
				setHead(node);
				p.next = null; // help GC
				failed = false;
				return interrupted;
			}
			// 确认/更新获取失败节点的状态，是否被取消(跳过)，是否已准备好挂起(返回true)
			if (shouldParkAfterFailedAcquire(p, node) &&
				// 借助JUC包下的LockSopport类的静态方法Park挂起当前线程
				parkAndCheckInterrupt())
				interrupted = true;
		}
	} finally {
		if (failed)
			// 取消请求，将当前节点从队列中移除
			cancelAcquire(node);
	}
}
```
AQS的队列为FIFO队列，所以每次被CPU唤醒，且当前线程不是头节点的位置，也是会被挂起的。AQS通过这样的方式，实现了竞争的排队策略

释放：
```java
public final boolean release(int arg) {
	// 同样是由子类实现
	if (tryRelease(arg)) {
		Node h = head;
		if (h != null && h.waitStatus != 0)
			// 找到AQS的头节点，唤醒它的下一个节点
			unparkSuccessor(h);
		return true;
	}
	return false;
}

private void unparkSuccessor(Node node) {
	int ws = node.waitStatus;
	if (ws < 0)
		compareAndSetWaitStatus(node, ws, 0);

	Node s = node.next;
	if (s == null || s.waitStatus > 0) {
		s = null;
		for (Node t = tail; t != null && t != node; t = t.prev)
			if (t.waitStatus <= 0)
				s = t;
	}
	if (s != null)
		LockSupport.unpark(s.thread);
}
```
因此，AQS独占功能的实现主要就是使用标志位+队列的方式，记录锁、竞争、释放等一系列独占的状态，因为在AQS的层面state可以表示锁，也可以表示其他状态，它并不关心它的子类把它变成一个什么工具类，只是提供了维护一个独占状态。甚至可以说，AQS只是维护一个状态，一个控制各个线程何时可以访问的状态，它只对状态负责，而这个状态表示什么含义，由子类自己去定义

通过CountDownLatch计数器工具类(类似于一个"栏栅"，在CountDownLatch的计数为0前，调用await方法的线程将一直阻塞)，可以看下AQS的共享功能的实现：
```java
// 例如输入5，代表初始化5个，countDown为0后await返回
public CountDownLatch(int count) {
	if (count < 0) throw new IllegalArgumentException("count < 0");
	this.sync = new Sync(count);
}

private static final class Sync extends AbstractQueuedSynchronizer {
	Sync(int count) {
		// 很明显就是修改AQS的state
		setState(count);
	}
}
```
先看下await方法：
```java
public void await() throws InterruptedException {
	sync.acquireSharedInterruptibly(1);
}

public final void acquireSharedInterruptibly(int arg) throws InterruptedException {
	// 检查下线程是否被中断
	if (Thread.interrupted())
		throw new InterruptedException();
	if (tryAcquireShared(arg) < 0)
		// 获取共享锁失败，即state不为0，意味着计数器还不为0，将当前线程放入到队列中去
		doAcquireSharedInterruptibly(arg);
}

// await方法的获取方式更像是在获取一个独占锁，不过为0时可以多个线程调用，同时等待返回
protected int tryAcquireShared(int acquires) {
	return (getState() == 0) ? 1 : -1;
}

private void doAcquireSharedInterruptibly(int arg) throws InterruptedException {
	// 标示这是一个共享节点
	final Node node = addWaiter(Node.SHARED);
	boolean failed = true;
	try {
		for (;;) {
			final Node p = node.predecessor();
			// 如果新建节点的前一个节点，就是Head，说明当前节点是AQS队列中等待获取锁的第一个节点，按照FIFO可直接尝试获取锁
			if (p == head) {
				int r = tryAcquireShared(arg);
				if (r >= 0) {
					// 获取成功，需要将当前节点设置为AQS队列中的第一个节点。队列的头节点表示正在获取锁的节点
					setHeadAndPropagate(node, r);
					p.next = null; // help GC
					failed = false;
					return;
				}
			}
			// 检查下是否需要将当前节点挂起
			if (shouldParkAfterFailedAcquire(p, node) &&
				parkAndCheckInterrupt())
				throw new InterruptedException();
		}
	} finally {
		if (failed)
			cancelAcquire(node);
	}
}

private void setHeadAndPropagate(Node node, int propagate) {
	Node h = head; // Record old head for check below
	setHead(node);

	if (propagate > 0 || h == null || h.waitStatus < 0 ||
		(h = head) == null || h.waitStatus < 0) {
		Node s = node.next;
		// 将当前节点的下一个节点取出来，如果符合条件，做releaseShared操作
		// 因为共享功能和独占功能不同，独占有且只有一个线程(持有锁的线程重复调用lock会被包装成多个节点在AQS队列中)能获取锁
		// 而共享状态是可以被共享的，AQS队列中的其他节点也应能第一时间知道状态的变化。因此节点自身获取共享锁成功后，唤醒下一个共享类型节点的操作，实现共享状态的向后传递
		if (s == null || s.isShared())
			doReleaseShared();
	}
}

private void doReleaseShared() {
	for (;;) {
		Node h = head;
		if (h != null && h != tail) {
			int ws = h.waitStatus;
			// 如果头节点是SIGNAL(-1)，则它可能正在等待一个信号，或在等待被唤醒，因此做两件事：
			// 1.重置waitStatus标志位  2.重置成功后, 唤醒下一个节点 
			if (ws == Node.SIGNAL) {
				if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
					continue;            // loop to recheck cases
				unparkSuccessor(h);
			}
			// 如果头节点的waitStatus是出于重置状态waitStatus==0的，将其设置为"传播"状态，即需要将状态向后一个节点传播
			else if (ws == 0 &&
					 !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
				continue;                // loop on failed CAS
		}
		if (h == head)                   // loop if head changed
			break;
	}
}
```
对于`doAcquireShared(忽略中断)`，AQS提供了类似的实现。另外还有
```java
// 响应中断。每次循环时，会检查当前线程的中断状态，以实现对线程中断的响应
doAcquireSharedInterruptibly
// 限制等待时间，超出时间后，方法返回
doAcquireSharedNanos

if (nanosTimeout <= 0L)
	return false;
// 算出超时时间
final long deadline = System.nanoTime() + nanosTimeout;
// 自旋...
for (;;) {
	// tryAcquireShared... 
	nanosTimeout = deadline - System.nanoTime();
	// 超时返回
	if (nanosTimeout <= 0L)
		return false;
	// 如果超时时间小于自旋时间(1000ns)，则不需要挂起线程
	if (shouldParkAfterFailedAcquire(p, node) &&
		nanosTimeout > spinForTimeoutThreshold)
		LockSupport.parkNanos(this, nanosTimeout);
}
```
然后看下countDown：
```java
public void countDown() {
	sync.releaseShared(1);
}

public final boolean releaseShared(int arg) {
	// 尝试去释放锁，同样由子类实现
	// 如果state为0，对CountDownLatch来说：所有的子线程已经执行完毕，可以唤醒调用await()方法的线程了，而这些线程正在 AQS的队列中，且被挂起
	if (tryReleaseShared(arg)) {
		doReleaseShared();
		return true;
	}
	return false;
}

// CountDownLatch，尝试-1
protected boolean tryReleaseShared(int releases) {
	// Decrement count; signal when transition to zero
	for (;;) {
		int c = getState();
		if (c == 0)
			return false;
		int nextc = c-1;
		if (compareAndSetState(c, nextc))
			return nextc == 0;
	}
}
```
如果获取共享锁失败后，将请求共享锁的线程封装成Node对象放入AQS的队列中，并挂起Node对象对应的线程，实现请求锁线程的等待操作。待共享锁可以被获取后，从头节点开始，依次唤醒头节点及其以后的所有共享类型的节点。实现共享状态的传播


### 独占锁ReentrantLock

ReentrantLock类实现了Lock接口，并提供了与synchronized相同的互斥性和内存可见性，它的底层是通过AQS来实现多线程同步的。与内置锁相比ReentrantLock不仅提供了更丰富的加锁机制，而且在性能上也不逊色于内置锁。提供了定时的锁等待，可中断的锁等待，公平锁，以及实现非块结构的加锁。而synchronized可以执行一些优化，例如对线程封闭的锁对象的锁消除优化，通过增加锁的粒度来消除内置锁的同步，因此当需要一些高级功能时才应该使用ReentrantLock

```java
public class ReentrantLock implements Lock, java.io.Serializable {
    private final Sync sync;
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
}
```
这里补完上面QAS的独占功能部分的子类实现。加锁部分：
```java
public void lock() {
	// FairSync或NonfairSync重写的lock方法
	sync.lock();
}

// FairSync
final void lock() {
	acquire(1);
}

protected final boolean tryAcquire(int acquires) {
	final Thread current = Thread.currentThread();
	// 获取AQS中的标志位
	int c = getState();
	if (c == 0) {
		// 如果队列中没有其他线程，说明没有线程正在占有锁！
		if (!hasQueuedPredecessors() &&
			// 修改一下状态位，这里的acquires是在lock的时候传递来的，即1
			compareAndSetState(0, acquires)) {
			// 如果CAS操作状态更新成功则代表当前线程获取锁，将当前线程设到AQS的一个变量中，说明这个线程拿走了锁
			// exclusiveOwnerThread = thread;
			setExclusiveOwnerThread(current);
			return true;
		}
	}
	// 如果不为0说明锁已经被拿走了，但因为ReentrantLock是重入锁，所以还要判断获取锁的线程是不是当前请求锁的线程
	else if (current == getExclusiveOwnerThread()) {
		// 是的情况下，acquires累加在state上
		int nextc = c + acquires;
		if (nextc < 0)
			throw new Error("Maximum lock count exceeded");
		setState(nextc);
		return true;
	}
	return false;
}


// NonfairSync
final void lock() {
	if (compareAndSetState(0, 1))
		// 如果CAS操作状态更新成功则代表当前线程获取锁，将当前线程设到AQS的一个变量中，说明这个线程拿走了锁
		setExclusiveOwnerThread(Thread.currentThread());
	else
		acquire(1);
}
```
释放锁：
```java
public void unlock() {
	// 实际执行AQS的release方法
	sync.release(1);
}

protected final boolean tryRelease(int releases) {
	int c = getState() - releases;
	if (Thread.currentThread() != getExclusiveOwnerThread())
		throw new IllegalMonitorStateException();
	boolean free = false;
	if (c == 0) {
		free = true;
		setExclusiveOwnerThread(null);
	}
	setState(c);
	return free;
}
```

#### ReentrantReadWriteLock 

![wrlock](/img/in-post/2021/05/wrlock.png)
分别有ReadLock和WriteLock处理不同场景的锁定。写锁使用独占功能，读锁使用共享功能

···java
// r主要与读锁配套使用
static final class HoldCounter {
	// 计数，表示某个读线程重入的次数
	int count = 0;
	// 获取当前线程的TID属性的值，用来唯一标识一个线程
	final long tid = getThreadId(Thread.currentThread());
}

static final class ThreadLocalHoldCounter extends ThreadLocal<HoldCounter> {
	// 重写初始化方法，获取的都是该HoldCounter值
	public HoldCounter initialValue() {
		return new HoldCounter();
	}
}

abstract static class Sync extends AbstractQueuedSynchronizer {
	// 设置了本地线程计数器和AQS的状态state
	Sync() {
		readHolds = new ThreadLocalHoldCounter();
		setState(getState()); // ensures visibility of readHolds
    }
}
···

同步状态在重入锁的实现中是表示被同一个线程重复获取的次数，仅仅表示是否锁定，而不用区分是读锁还是写锁。而读写锁需要在同步状态(这个整型变量)上维护多个读线程和一个写线程的状态

因此读写锁对于同步状态的实现是在一个整型变量上通过**按位切割**使用：将变量切割成两部分，高16位表示读，低16位表示写
![wrlock2](/img/in-post/2021/05/wrlock2.png)

写锁的lock和unlock就是独占式同步状态的获取与释放，因此只要看tryAcquire和tryRelease方法的实现即可。类似于写锁，读锁的lock和unlock的实际实现对应Sync的tryAcquireShared和tryReleaseShared方法

总结：一个线程要想同时持有写锁和读锁，必须先获取写锁再获取读锁；写锁可以**降级**为读锁；读锁不能**升级**为写锁

扩展：  
[ReentrantReadWriteLock读写锁详解](https://www.cnblogs.com/xiaoxi/p/9140541.html)


#### StampedLock锁

[Java多线程进阶（十一）—— J.U.C之locks框架：StampedLock](https://segmentfault.com/a/1190000015808032)  


### 分布式锁

C(一致性)、A(可用性)、P(分区容错性)

为保证最终一致性，需要很多技术方案支持，比如分布式事务、分布式锁

针对分布式锁：
+ 保证集群中，同一方法/资源在同一时间只能被一台机器上的一个线程执行
+ 锁最好是可重入锁(避免死锁)
+ 锁最好是阻塞锁(根据业务需要考虑)
+ 有高可用的获取和释放锁功能(原子的，网络或宕机时清除锁)
+ 获取/释放锁的性能要好
+ 基于数据库：锁表。依赖数据库(多库)、锁无失效时间(定时任务)、非阻塞(自旋/for update)、非可重入(记录)
+ 基于put方法：memcached的add，redis的setnx(不能设超时，需要额外EXPIRE设置，或lua脚本)(推荐redlock)
+ 基于ZooKeeper：某方法加锁，在对应节点目录下生成一个唯一的瞬时有序节点，只需判断最小节点访问。释放时删除节点(监听器)


### 线程池

> 创建线程和线程池(通过ThreadFactory)时要指定与业务相关的名称

直接看线程池ThreadPoolExecutor
```java
public ThreadPoolExecutor(int corePoolSize, // 核心线程池大小
						  int maximumPoolSize, // 最大线程池大小
						  long keepAliveTime, // 线程最大空闲时间
						  TimeUnit unit, // 时间单位
						  BlockingQueue<Runnable> workQueue, // 线程等待队列
						  ThreadFactory threadFactory, // 线程创建工厂
						  RejectedExecutionHandler handler) { // 拒绝策略
	if (corePoolSize < 0 ||
		maximumPoolSize <= 0 ||
		maximumPoolSize < corePoolSize ||
		keepAliveTime < 0)
		throw new IllegalArgumentException();
	if (workQueue == null || threadFactory == null || handler == null)
		throw new NullPointerException();
	this.acc = System.getSecurityManager() == null ?
			null :
			AccessController.getContext();
	this.corePoolSize = corePoolSize;
	this.maximumPoolSize = maximumPoolSize;
	this.workQueue = workQueue;
	this.keepAliveTime = unit.toNanos(keepAliveTime);
	this.threadFactory = threadFactory;
	this.handler = handler;
}
```
BlockingQueu用于存放提交的任务，队列的实际容量与线程池大小相关联：
+ 如果当前线程池任务线程数量小于核心线程池数量，执行器总是优先创建一个任务线程
+ 如果当前线程池任务线程数量大于核心线程池数量，执行器总是优先从线程队列中取一个空闲线程(加入队列)
+ 如果当前线程池任务线程数量大于核心线程池数量，且队列中无空闲任务线程，将会创建一个任务线程，直到超出maximumPoolSize，如果超时maximumPoolSize，则任务将会被拒绝

主要有三种队列策略：
+ 直接握手队列
	- 一个很好的默认选择是SynchronousQueue，它将任务交给线程而不需要保留，如果没有线程立即可用运行它，那么排队任务的尝试将失败，因此将构建新的线程
	- 通常需要无限制的maximumPoolSizes来避免拒绝新提交的任务
	- 适合具有内部依赖关系的请求集中处理的场景，可以避免锁定
	- 注意：当任务持续以平均提交速度>平均处理速度时，会导致线程数量会无限增长问题
+ 无界队列
	- 例如没有预定义容量的LinkedBlockingQueue，在corePoolSize线程繁忙时将所有新任务会放到队列中等待，从而导致maximumPoolSize的值没有任何作用
	- 适合不同任务互不影响、独立的场景
	-注意：当任务持续以平均提交速度>平均处理速度时，会导致队列无限增长
+ 有界队列
	- 例如ArrayBlockingQueue与maximumPoolSizes配置防止资源耗尽，但是需要调试，相互权衡
	- 使用大队列和较小的maximumPoolSizes可以最大限度地减少CPU使用率，操作系统资源和上下文切换开销，但会导致人为的低吞吐量。如果任务经常被阻塞(比如I/O限制)，那么系统可以调度比我们允许的更多的线程
	- 使用小队列通常需要较大的maximumPoolSizes，这会使CPU更繁忙，但可能会遇到不可接受的调度开销，这也会降低吞吐量

ThreadPoolExecutor为提供了每个任务执行前后提供了钩子方法，重写`beforeExecute(Thread，Runnable)`和`afterExecute(Runnable，Throwable)`方法来操纵执行环境

几种默认线程池
```java
ExecutorService es = Executors.newFixedThreadPool(10);

// 适用场景：可用于Web服务瞬时削峰，但需注意长时间持续高峰情况造成的队列阻塞
public static ExecutorService newFixedThreadPool(int nThreads) {
	return new ThreadPoolExecutor(nThreads, nThreads, // 固定核心线程
								  0L, TimeUnit.MILLISECONDS,
								  new LinkedBlockingQueue<Runnable>()); // 无界阻塞队列
}

// 适用场景：快速处理大量耗时较短的任务，如Netty的NIO接受请求时可使用
public static ExecutorService newCachedThreadPool() {
	return new ThreadPoolExecutor(0, Integer.MAX_VALUE,	// 线程数量几乎无限制
								  60L, TimeUnit.SECONDS,
								  new SynchronousQueue<Runnable>()); // 同步队列，入队出队必须同时传递
}

// 支持任务周期性调度的线程池。对比Timer而言是多线程、任务之间不影响、相对时间、异常无影响
public ScheduledThreadPoolExecutor(int corePoolSize) {
	super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
		  new DelayedWorkQueue());
}
```

1、工作线程与核心线程数比较，队列判断  
2、addWorker创建核心/非核心线程(，执行任务)  
2-1、自旋、CAS、重读ctl等结合，判断是否可以创建worker  
2-2、创建worker，使用ReentrantLock锁，然后并启动线程  
3、runWorker一直loop循环，一个个任务处理，没有任务就去getTask()，可能会阻塞

ThreadPoolExecutor的执行主要围绕Worker，Worker实现了AbstractQueuedSynchronizer并继承了Runnable

[ThreadPoolExecutor执行原理](https://www.jianshu.com/p/23cb8b903d2c)

> 使用线程池的情况下当程序结束时，需要记得调用shutdown()手动关闭线程池，如果希望线程池中的等待队列中的任务不继续执行，可以使用shutdownNow()方法

一个线程不应该由其他线程来强制中断或停止，而是应该由线程自己自行停止。所以`Thread.stop, Thread.suspend, Thread.resume`都已经被废弃了。`Thread.interrupt`的作用其实也不是中断线程，而是**通知线程应该中断了**，具体到底中断还是继续运行，应该由被通知的线程自己处理
+ 如果线程处于被阻塞状态（例如处于sleep, wait, join等状态），那么线程将立即退出被阻塞状态，并抛出一个InterruptedException异常
+ 如果线程处于正常活动状态，那么仅会将该线程的中断标志设置为true而已。被设置中断标志的线程将继续正常运行不受影响
+ 在正常运行任务时，经常检查本线程的中断标志位，如果被设置了中断标志就自行停止线程

所以想要正确的关闭线程池，并不是简单的调用`shutdown`方法那么简单，要考虑到应用场景的需求，如何拒绝新来的请求任务？如何处理等待队列中的任务？如何处理正在执行的任务？想好这几个问题，再确定如何优雅而正确的关闭线程池

### FutureTask

```java
public FutureTask(Callable<V> callable) {
	if (callable == null)
		throw new NullPointerException();
	this.callable = callable;
	this.state = NEW;       // ensure visibility of callable
}

public FutureTask(Runnable runnable, V result) {
	this.callable = Executors.callable(runnable, result);
	this.state = NEW;       // ensure visibility of callable
}
```
调用run方法：
```java
public void run() {
	if (state != NEW ||
		!UNSAFE.compareAndSwapObject(this, runnerOffset, null, Thread.currentThread()))
		return;
	try {
		Callable<V> c = callable;
		if (c != null && state == NEW) {
			V result;
			boolean ran;
			try {
				// 1、获取返回值
				result = c.call();
				ran = true;
			} catch (Throwable ex) {
				result = null;
				ran = false;
				// 2、FutureTask的异常处理
				setException(ex);
			}
			if (ran)
				// 3、设置返回值
				set(result);
		}
	} finally {
		// runner must be non-null until state is settled to
		// prevent concurrent calls to run()
		runner = null;
		// state must be re-read after nulling runner to prevent
		// leaked interrupts
		int s = state;
		if (s >= INTERRUPTING)
			handlePossibleCancellationInterrupt(s);
	}
}

protected void setException(Throwable t) {
	if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
		outcome = t;
		// 设置为异常状态
		UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL);
		finishCompletion();
	}
}

protected void set(V v) {
	if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
		outcome = v;
		// 设置为正常结束状态
		UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
		finishCompletion();
	}
}
```
再看下get阻塞方法：
```java
public V get() throws InterruptedException, ExecutionException {
	int s = state;
	// 未完成：status为NEW和COMPLETING，则进入阻塞状态，等待完成
	if (s <= COMPLETING)
		s = awaitDone(false, 0L);
	return report(s);  //判断处理返回值
}

private V report(int s) throws ExecutionException {
	Object x = outcome;
	// 根据state判断线程处理状态，并对outcome返回结果进行强转。
	if (s == NORMAL)
		return (V)x;
	if (s >= CANCELLED)
		throw new CancellationException();
	throw new ExecutionException((Throwable)x); //在主线程中抛出异常
}
```


## 其他

### Threadlocal

ThreadLocal的作用是提供线程内的局部变量，这种变量在线程的生命周期内起作用，减少同一个线程多个方法或组件间一些公共变量的传递的复杂度

每个线程都有`ThreadLocal.ThreadLocalMap threadLocals`(当前线程的ThreadLocalMap，主要存储该线程自身的ThreadLocal)和`ThreadLocal.ThreadLocalMap inheritableThreadLocals`(自父线程集成而来的ThreadLocalMap，主要用于父子线程间ThreadLocal变量的传递)映射表，ThreadLocalMap是类Map结构，是ThreadLocal的内部静态类，采用线性探测的开放地址法解决hash冲突。当线程第一次调用ThreadLocal的set或者get方法的时候才会创建他们。这个映射表的key是ThreadLocal实例本身，value是真正需要存储的object。这样设计的好处是让Map的Entry数量变少了，否则要存Thread的数量之和，现在只要存ThreadLocal的数量之和即可

```java
// ThreadLocalMap
private Entry[] table;

// 是用ThreadLocal的弱引用为key
static class Entry extends WeakReference<ThreadLocal<?>> {
	/** The value associated with this ThreadLocal. */
	Object value;
	Entry(ThreadLocal<?> k, Object v) {
		super(k);
		value = v;
	}
}

ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
	table = new Entry[INITIAL_CAPACITY];
	int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
	table[i] = new Entry(firstKey, firstValue);
	size = 1;
	setThreshold(INITIAL_CAPACITY);
}
```

每个线程的本地变量不是存放在ThreadLocal实例中，而是放在调用线程的ThreadLocals变量里面。也就是说，ThreadLocal类型的本地变量是存放在**具体的线程空间**上，其本身相当于一个装载本地变量的工具壳，通过set方法将value添加到调用线程的threadLocals中，当调用线程调用get方法时候能够从它的threadLocals中取出变量。如果调用线程一直不终止，那么这个本地变量将会一直存放在他的threadLocals中，所以不使用本地变量的时候需要调用**remove方法**将threadLocals中删除不用的本地变量

![trl](/img/in-post/2021/05/trl.jpg)

接下来看Threadlocal的set方法：
```java
public void set(T value) {
	// 获取当前线程（调用者线程）
	Thread t = Thread.currentThread();
	// 以当前线程作为key值，去查找对应的线程变量，找到对应的map
	ThreadLocalMap map = getMap(t);
	if (map != null)
		map.set(this, value);
	else
		createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
	return t.threadLocals;
}

void createMap(Thread t, T firstValue) {
	t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```
然后看下get方法：
```java
public T get() {
	Thread t = Thread.currentThread();
	ThreadLocalMap map = getMap(t);
	if (map != null) {
		ThreadLocalMap.Entry e = map.getEntry(this);
		if (e != null) {
			T result = (T)e.value;
			return result;
		}
	}
	// 执行到此处，threadLocals为null，初始化当前线程的threadLocals变量
	return setInitialValue();
}

private Entry getEntry(ThreadLocal<?> key) {
	int i = key.threadLocalHashCode & (table.length - 1);
	Entry e = table[i];
	if (e != null && e.get() == key)
		return e;
	else
		return getEntryAfterMiss(key, i, e);
}
```


使用ThreadLocal不当可能会导致内存泄漏。ThreadLocalMap使用ThreadLocal的弱引用作为key，如果一个ThreadLocal没有外部强引用引用它，Key就会在GC时被回收。这样就会导致ThreadLocalMap中出现key为null的Entry，而value还存在着强引用，只有thead线程退出以后，value的强引用链条才会断掉。如果当前线程迟迟不结束，这些key为null的Entry的value就会有一条一直存在的强引用，让它永远不会被回收：`ThreadLocalRef -》Thread -》ThreadLocalMap -》Entry -》Value`

JDK在处理上有加入保护措施，get方法时会擦除null位置的Entry，在向下个位置查询过程中会擦除所有遇到key为null的value，set方法也会这样，但是必须手动调用。因此，一些时候需要主动调用remove方法手动删除不再需要的ThreadLocal。JDK建议将ThreadLocal变量定义为`private static`的，让其生命周期更长，由于一直存在ThreadLocal的强引用，就不会被回收ThreadLocal，就可以在任何时候根据ThreadLocal的弱引用访问到Entry的value，并remove它，防止内存泄露


InheritableThreadLocal继承自ThreadLocal。由于ThreadLocal设计之初就是为了绑定当前线程，如果希望当前线程的ThreadLocal能够被子线程使用，InheritableThreadLocal就应运而生

InheritableThreadLocal类重写了ThreadLocal的3个函数：
```java
 /**
 * 该函数在父线程创建子线程，向子线程复制InheritableThreadLocal变量时使用
 */
protected T childValue(T parentValue) {
	return parentValue;
}

/**
 * 由于重写了getMap，操作InheritableThreadLocal时，
 * 将只影响Thread类中的inheritableThreadLocals变量，
 * 与threadLocals变量不再有关系
 */
ThreadLocalMap getMap(Thread t) {
   return t.inheritableThreadLocals;
}

/**
 * 类似于getMap，操作InheritableThreadLocal时，
 * 将只影响Thread类中的inheritableThreadLocals变量，
 * 与threadLocals变量不再有关系
 */
void createMap(Thread t, T firstValue) {
	t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
}
```
创建子线程
```java
/**
 * 初始化一个线程.
 * 此函数有两处调用，
 * 1、init()，不传AccessControlContext，inheritThreadLocals=true
 * 2、		  传递AccessControlContext，inheritThreadLocals=false
 */
private void init(ThreadGroup g, Runnable target, String name,
				  long stackSize, AccessControlContext acc,
				  boolean inheritThreadLocals) {
	// ...
	// 采用默认方式产生子线程时，inheritThreadLocals=true；若此时父线程inheritableThreadLocals不为空，则将父线程inheritableThreadLocals传递至子线程，子线程将parentMap中的所有记录逐一复制至自身线程
	if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
	// ...
}
```

#### ThreadLocalRandom

ThreadLocalRandom:是JDK 7之后提供并发产生随机数，能够解决多个线程发生的竞争争夺。ThreadLocalRandom不是直接用new实例化，而是第一次使用其静态方法current()。`Math.random()`改变到ThreadLocalRandom，我们不再有从多个线程访问同一个随机数生成器实例的争夺，取代以前每个随机变量实例化一个随机数生成器实例，我们可以每个线程实例化一个


### 使用中

在高并发、高流量、响应时间要求比较小的系统中，同步打印日志已经满足不了需求了。将日志写入磁盘的操作是业务线程同步调用完成的，那么把日志任务放入一个队列后直接返回，然后使用一个线程专门负责从队列中获取日志任务，并将其写入磁盘。这就是logback提供的异步日志打印模型。logback的异步日志打印模型是一个多生产者、单消费者模型，提供队列把同步日志打印转换成了异步

每个SimpleDateFormat实例里面都有一个Calendar对象。SimpleDateFormat之所以是线程不安全的，是因为Calendar线程不安全。后者之所以是线程不安全的，是因为其中存放日期数据的变量都是线程不安全的，比如fields、time等

对需要复用但是会被下游修改的参数要进行深复制


### Quartz

Job+Trigger+Scheduler

Scheduler任务调度器：ThreadPool+Jobstore  
Trigger触发器：定义任务调度的时间规则

QuartzSchedulerThread：一个协调的thread，起来后`while(true)`，去线程池拿当前可用的线程(workerThread)，如果没拿到`wait(500)`，如果拿到，就从JobStore里获取可以执行的JOB，让workerThread去`run`

workerThread：`run`完后通知QuartzSchedulerThread，继续去找可用线程，然后再找trigger和Job，让这个thread去执行它们



### concurrent其他

Semaphore信号量：控制特定资源同一时间被访问个数

CyclieBarrier：阻塞调用的线程，直到条件满足，阻塞线程同时被打开

![concurrent-package](/img/in-post/2021/05/concurrent-package.png)