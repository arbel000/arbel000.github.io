---
layout:     post
title:      "Java性能优化02-并行优化"
date:       2019-01-08
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



# 并行设计模式

## Future模式

JDK内置了Future模式，最为重要的模块是FutrueTask类，它实现了Runnable接口，作为单独的线程运行。在其run()方法中，通过Sync内部类，调用Callback接口，并维护Callable接口的返回对象。当使用FutureTask.get()方法时，将返回Callback接口的返回对象。除了基本的功能外，JDK内置的Futrue模式还可以取消Futrue任务或设置超时时间

## Master-Worker模式

是常用的并行模式之一，核心思想是系统由两类进程协作工作：Master进程和Worker进程。Master进程负责接收和分配任务，Worker进程负责处理子任务。当各个Worker进程将子任务处理完成后，将结果返回给Master进程。这种模式的好处是能够将一个大任务分解为若干个小任务，并行执行，从而提高系统的吞吐量。而对于请求者来说，任务提交后Master会分配任务并返回，并不需要等待全部完成后返回，其处理过程是异步的

Master-Worker模式是一种使用多线程进行数据处理的结构。多个Worker进程协作处理用户请求，Master进程负责维护Worker进程，并整合最终处理结果。一种简单的实现就是Master作为主进程，维护一个Worker进程队列、子任务队列和子结果集

## Guarded Suspension模式

意为保护暂停，其核心思想是仅当服务进程准备好时，才提供服务。在服务端短时间收到大量请求时，超过了处理能力，这时候将请求进行排队，由服务端一个接一个处理，这样就保证请求不丢失，又能避免同时处理太多请求而崩溃。Guarded Suspension模式可以确保系统仅在有能力处理某个任务时，才会处理。没有能力处理任务时，它将暂存任务信息，等待系统空闲。普通的Guarded Suspension模式不能返回处理结果，结合Future模式，就可以进行扩展，构造一个可以携带返回值的Guarded Suspension

## 不变模式

具有强一致性和不变性，天然的多线程友好。基本上去除setter等修改方法，将所有属性设为私有并final，确保没有子类可以重载(final class)，拥有一个可以创建完整对象的构造函数。就可以实现一个不变对象，它是不需要被同步的，在需求允许下，不变模式可以提高系统的并发性能和并发量

## 生产者-消费者模式

其核心组件是共享内存缓冲区，作为生产者和消费者间的通信桥梁，同时由于缓冲区的存在，使得生产和消费速度间存在时间差


# 线程池

JDK提供了一套Executor框架，其中ThreadPoolEecutor表示一个线程池，Executors类则扮演线程池工厂的角色。ThreadPoolEecutor类实现了Executor接口，因此通过这个接口任何Runnable对象都可以被ThreadPoolEecutor调度

![executor](/img/in-post/2019/01/executor.png)
```java
public ThreadPoolExecutor(int corePoolSize,	// 核心线程数
			  int maximumPoolSize,	// 最大线程数
			  long keepAliveTime,	// 多余的空闲线程存活时间
			  TimeUnit unit,	// 时间单位
			  BlockingQueue<Runnable> workQueue,	// 任务队列
			  ThreadFactory threadFactory,	// 线程工厂
			  RejectedExecutionHandler handler)	// 拒绝策略
```
几种BlockingQueue：  
**1、直接提交队列**：SynchronousQueue，没有容量，每一个插入操作都需要等待一个相应的删除操作，SynchronousQueue不保存任务，总是将任务提交给线程执行，如果没有空闲线程，则尝试新增新的线程，如果线程数量达到最大值则执行拒绝策略。例如Executors.newCachedThreadPool  
**2、有界队列**：ArrayBlockingQueue。若有新任务需要执行，如果线程池实际线程数小于corePoolSize，则优先创建新线程，若大于corePoolSize则新任务加入等待队列。当队列已满时，若总线程数不大于maximumPoolSize，则创建新的线程执行任务，若大于则执行拒绝策略  
**3、无界任务队列**：LinkedBlockingQueue。与有界队列相比，除非系统资源耗尽，否则无界队列不存在任务入队失败的情况，并且系统线程数到达corePoolSize后就不会继续增加，后续新任务进来又没空闲线程情况下，直接进入队列等待，无界队列将一直增长直到系统内存耗尽。例如Executors.newSingleThreadExecutor和Executors.newFixedThreadPool  
PS、LinkedBlockingQueue中的锁是分离的，生产者的锁PutLock，消费者的锁takeLock，而ArrayBlockingQueue生产者和消费者使用的是同一把锁；LinkedBlockingQueue内部维护的是一个链表结构，而ArrayBlockingQueue内部维护了一个数组；ArrayBlockingQueue在初始化的时候，必须传入一个容量大小的值  
**4、优先任务队列**：带有执行优先级的队列，通过PriorityBlockQueue实现。可以控制任务的执行先后顺序，是一个特殊的无界队列。ArrayBlockingQueue与LinkedBlockingQueue都是按照先进先出算法处理任务的，而PriorityBlockQueue则可以根据任务自身的优先级顺序先后执行(实现Comparable接口)

估算线程池大小：Nthreads = Ncpu<CPU数量> * Ucpu<目标CPU使用率> * (1 + W/C<等待时间与计算时间的比率>)

ThreadPoolExecutor也是一个可以扩展的线程池，它提供了beforeExecute()、afterExecute()和terminated()3个接口对线程池进行控制


# 并发数据结构

## 并发List

Vector或CopyOnWriteArrayList是两个线程安全的List实现，ArrayList不是，如果要使用则需要Collections.synchronizedList(List<T> list)进行包装。CopyOnWriteArrayList会在写操作时复制该对象，很好的利用了对象的不变性，只有在试图改变对象时，总是先获取对象的一个副本，然后对副本进行修改，最后将副本写回。这种实现方式减少锁竞争，从而提高高并发时的读性能，但是牺牲了写的性能。因此在读为主的场景中，CopyOnWriteArrayList优于Vector，而写频繁时，CopyOnWriteArrayList效率不高，应该优先使用Vector

## 并发Set

与List相似，并发Set也有一个CopyOnWriteArraySet，它的内部实现完全依赖于CopyOnWriteArrayList，因此特性与CopyOnWriteArrayList一致，适用于读多写少的场景。在需要并发写的场景下则可以使用Collections.synchronizedSet(Set<T> s)

## 并发Map

也同样可以使用Collections.synchronizedMap(Map<K,V> m)方法，但是高并发下这个Map的性能不是最优的。由于Map是使用频繁的一个数据结构，因此JDK提供了专用于高并发的ConcurrentHashMap，它的get也是无锁的，put操作的锁粒度又小于同步的HashMap

## 并发Queue

并发队列上，JDK提供了两套实现，一个是以ConcurrentLinkedQueue为代表的高性能队列，一个是以BlockingQueue接口为代表的阻塞队列。ConcurrentLinkedQueue通过无锁的方式实现了高并发状态下的高性能，通常性能要好过BlockingQueue。而BlockingQueue的典型使用场景就是生产者-消费者模式中或多线程间的数据共享，提供了一种读写阻塞等待的机制。BlockingQueue主要的两种实现是ArrayBlockingQueue和LinkedBlockingQueue

## 并发Deque

双端队列Double-ended Queue，允许在队列头部或尾部进行出队和入队操作。有3个实现类，LinkedList使用链表实现了双端队列，ArrayDeque使用数组实现双端队列，拥有高效的随机访问性能，在数据量不大的情况下表现更好，但是它们都不是安全的。LinkedBlockingDeque是一个线程安全的双端队列实现，内部使用了链表结构，由于没有进行读写锁的分离，因此同一时间只能有一个线程对其进行操作，因此实际应用场景中性能很低


# 并发控制方法

## Java内存模型与volatile

在Java中，每个线程都有一块工作内存区，其中存放被所有线程共享的主内存中变量的拷贝。线程只能操作自己的工作内存，不能直接操作主内存的数据，每次使用共享变量需要从主内存中进行拷贝到工作内存，完成后将变量值从工作内存写回主内存。一个线程可以执行使用(use)、赋值(assign)、装载(load)、存储(store)、锁定(lock)、解锁(unlock)。而主内存可以执行读()、写()、锁定(lock)、解锁(unlock)，每一个操作都是原子的。特殊的，double和long类型变量是非原子操作，如果一个double或long变量没有声明volatile，则变量在进行read或write操作时，主内存会把它当做两个32位的read或write操作，则两个操作间是可以分开的，因此32位系统中，必须对double或long进行同步

虽然线程或主内存的操作是原子的，但是主内存和线程的工作内存间的数据传送并不满足原子性。即当数据从主内存复制到工作内存时，必定出现2个动作：主内存执行的read操作，工作内存执行的相应的load操作。当数据从工作内存拷贝到主内存时，同样也有2个动作：工作内存执行的store操作，主内存执行的相应的write操作

由于每个线程都有自己的工作内存区，因此一个线程改变自己工作内存中的数据，对其他线程是不可见的。为此，可以使用volatile关键字迫使所有线程均读写主内存中的对应变量(强制同步)，使得volatile变量在多线程间可见。声明volatile的变量保证：1、其他线程对变量的修改，可以即时反应在当前线程中。2、确保当前线程对volatile变量的修改，能即时写回共享主内存中，并被其他线程所见。3、使用volatile声明的变量，编译器会保证其有序性(内存屏障)

## synchronized关键字

最常用的同步方法，并且在JVM不断优化下，与非公平锁的差距缩小，同时synchronized更加简洁明了，可读性和可维护性更好

synchronized可以锁定一个对象的方法，也可以构造同步代码块。此外synchronized还可以用于static函数，相当于将锁加到当前的class对象上，所有对该方法的调用都必须得到class对象的锁

虽然synchronized可以保证对象或代码的线程安全，但是为了实现多线程间的交互，还需要使用Object对象的wait()和notify()方法

## ReentrantLock可重入锁

ReentrantLock比内部锁synchronized拥有更强大的功能，可以中断，可以定时。ReentrantLock提供了公平锁和非公平锁两种锁，公平锁的实现代价更大，所以无特殊需要优先选择非公平锁，而synchronized提供锁也不是绝对公平的。在ReentrantLock使用完毕后，需要手动释放锁

## ReadWriteLock读写锁

ReadWriteLock是读写分离锁，可以有效帮助减少锁竞争，提升系统性能。读写锁允许多个线程同时读，但是考虑到数据完整性，写写操作和读写操作间依然是需要互相等待和持有锁的。如果系统中，读操作次数远大于写操作次数，读写锁就可以发挥最大的功效

## Condition对象

线程间的协调工作光有锁是不够的，业务层可能会有更复杂的协作逻辑，Condition对象就可以用于协调多线程间的复杂操作。Condition是与锁相关联的，通过Lock接口的Condition newCondition()方法可以生成一个与锁绑定的Condition实例

Condition接口提供的基本方法有：  
1、await()方法会使当前线程等待，同时释放当前锁，当其他线程中使用signal()或signalAll()方法时，线程会重新获得锁并继续执行。或者当线程被中断时，也能跳出等待  
2、awaitUninterruptibly()方法与await()方法基本相同，但是它并不会在等待过程中响应中断  
3、singal()方法用于唤醒一个等待中的线程  
4、singalAll()方法唤醒所有在等待中的线程

BlockingQueue在实现队列满时让生产者等待，又在队列空时让消费者等待中，就与Condition紧密相关

## Semaphore信号量

信号量为多线程协作提供了更为强大的控制方法。广义上信号量是对锁的扩展。无论是内部锁synchronized还是重入锁ReentrantLock，一次都只允许一个线程访问一个资源，而信号量却可以指定多个线程同时访问某一个资源。在构造信号量对象时，必须要指定信号量的准入数，即同时能申请多少个许可。当每个线程每次只申请一个许可时，将相当于指定了同时有多少个线程可以访问某一个资源

信号量使用acquire()方法尝试获得一个准入的许可，若无法获得，则线程会等待直到有线程释放一个许可或当前线程被中断。acquireUninterruptibly()方法和acquire()方法一样只是不响应中断。tryAcquire()尝试获得一个许可，成功返回true失败返回false，它立即返回不会等待。release()用于在线程访问资源结束后，释放一个许可，以使得其他等待许可的线程可以进行资源访问

## ThreadLocal线程局部变量

ThreadLocal是一种多线程间并发访问变量的解决方案，与synchronized等加锁方式不同，ThreadLocal完全不提供锁，而是以空间换时间的方式，为每个线程提供变量的独立副本，以保障线程安全，因此它不是一种数据共享的解决方案


# 锁的性能和优化

## 死锁

死锁问题是多线程特有的问题，是线程间切换消耗系统性能的一种极端情况。在死锁时线程间相互等待，而又不释放自己的资源，导致无穷无尽的等待，让系统任务永远无法完成。需要坚决避免和杜绝，可以通过线程dump查看死锁情况

一般情况出现死锁满足以下条件：  
1、互斥条件  
2、请求与保持条件  
3、不剥夺条件  
4、循环等待条件

## 减小锁持有时间

在锁竞争中，单个线程对锁的持有时间和系统性能有着直接关系，因此尽可能减少对某个锁的占有时间，可以减少线程间互斥的可能。一个较为优化的方案就是，只有在必要时进行同步

## 减少颗粒度

减小锁的粒度也是一种削弱多线程间锁竞争的有效手段，比如对资源进行分段拆分，这样就不用对整个资源获取全局锁。减小颗粒度，就是缩小锁定对象的范围，从而减少锁冲突的可能性，进而提高系统性能

## 读写分离锁代替独占锁

在读多写少的场景下，读写锁ReadWriteLock可以提高系统的性能，这是减少颗粒度的一种特殊情况。在对象没有锁的情况下，才能获得对象写锁；在写锁释放前，也无法在对象上附加任何锁(读/写锁)

## 锁分离

读写锁思想的延伸就是锁分离。使用类似分离的思想，也可以对独占锁进行分离，一个典型的例子就是LinkedBlockingQueue的实现

在LinkedBlockingQueue的实现中，take()和put()方法分别实现了从队列中取得数据和往队列中增加数据的功能。虽然两个方法都对队列进行了修改操作，但由于LinkedBlockingQueue是基于链表的，因此两个操作分别作用于队列的头和尾，理论上两者并不冲突。因此在JDK的实现上，用了两把锁分离了take()和put()操作

## 自旋锁

自旋锁可以使线程在没有取得锁后，不被挂起，转而去执行一个空循环(即自旋)，在若干的空循环后，若线程获得了锁则继续进行，若线程依然不能获得锁，才会被挂起。使用自旋锁后，线程被挂起的概率就会相对减少，线程执行的连贯性相对加强。因此对那些锁竞争不是很激烈，锁占用时间很短的并发线程，有着积极的作用

JVM虚拟机也提供-XX+UseSpinning参数来开启自旋锁，使用-XX:PreBlockSpin参数设置自旋次数

## 其他优化

锁粗化：在虚拟机遇到一连串的对同一锁不断进行请求和释放的操作时，便会把所有的锁操作整合成对锁的一次请求，从而减少对锁的请求同步次数。锁粗化的思想与减少锁持有时间是相反的，但在不同场合权衡下，它们的效果并不相同

锁消除：是JVM在即时编译时，通过对运行上下文扫描，去除不可能存在共享资源竞争的锁。通过锁清除，可以节省毫无意义的请求锁时间。比如使用了Vector，StringBuffer等内置工具类，里面的同步方法有时是不需要的，JVM虚拟机可以在运行时，基于逃逸分析技术，捕获到这些不可能存在竞争缺申请锁竞争的代码段，并消除这些不必要的锁，从而提高系统性能。逃逸分析和锁清除分别可以使用JVM参数-XX:+DoEscapeAnalysis和-XX:+EliminateLocks开启(锁消除必须工作在-server模式下)

锁偏向：如果程序没有竞争，则取消之前已经取得锁的线程同步操作。也就是说若某一锁被线程获取后，便进入偏向模式，当线程再次请求这个锁时，无需进行相关的同步操作，节省操作时间。如果在此期间有其他线程进行了锁操作，则锁退出偏向模式。在JVM中可以使用-XX:+UseBiasedLocking开启。但是偏向锁在竞争激烈的场景下没有优化效果，此时不仅得不到性能优化，还会有损系统性能


# 无锁的并行计算

在高并发时，对锁的激烈竞争可能会成为系统瓶颈。为此可以采取一种称为非阻塞同步的方法，依然保证数据和程序在高并发环境下保持多线程间的一致性

## 非阻塞的同步/无锁

基于锁的同步方式，也是一种阻塞式的线程间同步方式，无论使用信号量、重入锁或内部锁，受到核心资源限制，不同线程竞争时，都会不可避免相互等待，从而阻塞当前线程

最简单的一种非阻塞式同步就是ThreadLocal，每个线程拥有各自独立的副本，并行计算时无需互相等待

CAS算法是一种基于比较并交换的无锁并发控制方法。与锁的实现相比，无锁算法的设计和实现都会复杂很多，但由于其非阻塞性，对死锁问题天然免疫，而且线程间相互影响也远远小于基于锁的方式。更为重要的是无锁方式没有锁竞争带来的系统开销，也没线程间频繁调度切换带来的开销，因此拥有更好的性能

CAS包含3个参数(V,E,N)，V表示要更新的变量，E表示预期值，N表示新值。仅当V值等于E值时，才会将V的值设为N，如果V值和E值不相等，则说明已经有其他线程做了更新，则当前线程什么都不做。最后CAS返回当前V的真实值。当多个线程同时使用CAS操作同一个变量时，只有一个会胜出，其余均会失败，失败的线程不会被挂起，仅被告知，并且允许再次尝试，也允许失败线程放弃操作

硬件层面大部分现代处理器都已支持原子化的CAS指令

## 原子操作

在JDK的java.util.concurrent.atomic包下，有一组使用无锁算法实现的原子操作类，主要由AtomicInteger、AtomicInegerArray、AtomicLong、AtomicLongArray和AtomicReference等。它们分别封装对整数、整数数组、长整数、长整数数组和普通对象的线程安全操作

它们就是使用了CAS算法进行工作的，会使用一个无穷循环进行CAS冲突处理，即当前线程受其他线程影响失败时，会不断尝试直到成功

## Amino框架

Amino框架是无锁并行框架，是Apache的一个分支项目，提供了可用于线程安全的、基于无锁算法的一些数据结构，同时还内置了一些多线程调度模式

Amino提供了一些基础集合类，比如List、Set等集合类，均采用无锁的方式实现，封装了复杂的无锁算法。比如LockFreeList(链表)、LockFreeVector(数组)、LockFreeSet

除了简单的无锁集合类，Amino还提供了无锁的树结构，LockFreeBSTree是一颗无锁且线程安全的二叉搜索树。Amino还提供了更为复杂的数据结构——图。主要提供了有向图和无向图两种数据结构

Amino还提供了一些非常有用的并行开发模式实现，比如Master-Worker模式，提供了两套实现，一种是静态的，另一种是动态的实现


# 协程

与进程相比，线程是一个较为轻量级的并行程序解决方案。但是对于高并发程序而言，线程对系统资源的占用量依旧不小，则也限制了系统的并发数。为了进一步提升系统并发数，可以对线程做进一步分割，即所谓的协程。无论是进程、线程还是协程，在逻辑层它们都对应一个任务，以执行一段代码逻辑。当使用协程实现一个任务时，协程并不完全占据一个线程，当一个协程处于等待状态时，它便会把CPU交给该线程内的其他协程。与线程相比，协程间的切换更为轻便，因此具有更低的操作系统成本和更高的任务并发性

## Kilim框架

协程并不被Java语言原生支持，因此在Java中使用协程，可以使用协程框架，比如Kilim框架



引用：《Java程序性能优化》
