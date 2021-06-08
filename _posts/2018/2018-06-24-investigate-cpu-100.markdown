---
layout:     post
title:      "java项目 CPU占用100%问题"
date:       2018-06-24
author:     "ZhouJ000"
header-img: "img/in-post/2018/post-bg-2018-headbg.jpg"
catalog: true
tags:
    - 排查
    - 命令
--- 

<font id="last-updated">最后更新于：2018-06-30</font>

[java项目 CPU占用100%问题](https://zhouj000.github.io/2018/06/24/investigate-cpu-100/)  
[java项目 内存溢出问题](https://zhouj000.github.io/2018/07/14/investigate-out-memery/)

[Java性能优化04-调优工具](https://zhouj000.github.io/2019/01/11/java-optimize-04/)  



> 一个应用占用CPU很高，除了确实是计算密集型应用之外，通常原因都是出现了死循环，死递归和死锁。

> 当us值过高时，表示运行的应用消耗大量的CPU。
java应用造成us高的原因主要是线程一直处于可运行（Runnable）状态，
通常这些线程在执行无阻塞、循环、正则或纯粹的计算等任务造成的；另外一个可能也会造成us高的原因是频繁GC。

> 当sy值高时，表示linux花费了更多的时间在进行java线程切换。
java应用造成这种现象的主要原因是启动的线程比较多，且这些线程多数处于不断的阻塞（例如锁等待，IO等待状态）和执行状态的变化过程中，
这就导致了操作系统要不断地切换执行的线程，产生大量的线程上下文切换。


# 1.确定Java应用进程

### jps

> 一个显示当前所有java进程pid的命令

参数-m： 输出主函数main class传入的参数
````
jps -m

25650 Jps -m
```

参数-l： 输出应用程序main class的完整package名或者应用程序的jar文件完整路径名
```
jps -l

24582 sun.tools.jps.Jps
```

参数-v： 输出传递给JVM的参数
```
jps -v

24482 Jps -Denv.class.path=.:/usr/java/jdk1.7.0_79/lib/dt.jar:/usr/java/jdk1.7.0_79/lib/tool.jar -Dapplication.home=/usr/java/jdk1.7.0_79 -Xms8m
```


## Linux下查看

### top命令

> top命令是Linux下常用的性能分析工具，能够实时显示系统中各个进程的资源占用状况，常用于服务端性能分析

```
查看指定进程下的线程cpu占用比例，分析是具体哪个线程占用率过高：
top -p <PID> -H
```

top命令的结果分为两个部分：
+ 统计信息： 前五行是系统整体的统计信息
+ 进程信息： 统计信息下方类似表格区域显示的是各个进程的详细信息，默认5秒刷新一次

常用命令参数:
+ -u<用户名>： 指定用户名
+ -p<进程号>： 指定进程
+ -n<次数>： 循环显示的次数
+ -H： 显示进程的所有线程
+ -d： 屏幕刷新间隔时间

常用命令交互：
+ 基础操作
    - l(字母)： 切换显示平均负载和启动时间信息
	- m： 切换显示内存信息
	- t： 切换显示进程和CPU状态信息
	- c： 切换显示命令名称和完整命令行
	- 1(数字)： 显示CPU详细信息，每核显示一行
	- d / s： 修改刷新频率，单位为秒
	- < ENTER > / < SPACE >： 刷新显示
	- f： top进入另一个视图，在这里可以编排基本视图中的显示字段
	- h： 可显示帮助界面
+ 进程列表排序 
	- M： 根据驻留内存大小进行排序
	- P： 根据CPU使用百分比大小进行排序
	- T： 根据时间/累计时间进行排序
	- shift + > / shift + <： 向右或左改变排序列

```
  Z,B,E,e   Global: 'Z' colors; 'B' bold; 'E'/'e' summary/task memory scale
  l,t,m     Toggle Summary: 'l' load avg; 't' task/cpu stats; 'm' memory info
  0,1,2,3,I Toggle: '0' zeros; '1/2/3' cpus or numa node views; 'I' Irix mode
  f,F,X     Fields: 'f'/'F' add/remove/order/sort; 'X' increase fixed-width

  L,&,<,> . Locate: 'L'/'&' find/again; Move sort column: '<'/'>' left/right
  R,H,V,J . Toggle: 'R' Sort; 'H' Threads; 'V' Forest view; 'J' Num justify
  c,i,S,j . Toggle: 'c' Cmd name/line; 'i' Idle; 'S' Time; 'j' Str justify
  x,y     . Toggle highlights: 'x' sort field; 'y' running tasks
  z,b     . Toggle: 'z' color/mono; 'b' bold/reverse (only if 'x' or 'y')
  u,U,o,O . Filter by: 'u'/'U' effective/any user; 'o'/'O' other criteria
  n,#,^O  . Set: 'n'/'#' max tasks displayed; Show: Ctrl+'O' other filter(s)
  C,...   . Toggle scroll coordinates msg for: up,down,left,right,home,end

  k,r       Manipulate tasks: 'k' kill; 'r' renice
  d or s    Set update interval
  W,Y       Write configuration file 'W'; Inspect other output 'Y'
  q         Quit
Commands available anytime   -------------
  A       . Alternate display mode toggle, show Single / Multiple windows
  g       . Choose another field group and make it 'current', or change now
            by selecting a number from:  1 =Def; 2 =Job; 3 =Mem; or 4 =Usr
Commands requiring 'A' mode  -------------
  G       . Change the Name of the 'current' window/field group
  a , w   . Cycle through all four windows:  'a' Forward; 'w' Backward
  - , _   . Show/Hide:  '-' Current window; '_' all Visible/Invisible
The screen will be divided evenly between task displays.  But you can make
some larger or smaller, using 'n' and 'i' commands.  Then later you could:
  = , +   . Rebalance tasks:  '=' Current window; '+' Every window
            (this also forces the current or every window to become visible)
```
			
CPU状态信息字段：
+ us： 用户空间占用CPU百分比
+ sy： 内核空间占用CPU百分比
+ ni： 用户进程空间内改变过优先级的进程占用CPU百分比
+ id： 空闲CPU百分比
+ wa： 等待输入输出的CPU时间百分比
+ hi： 硬件中断
+ si： 软件中断

其他：
+ 待


### ps命令

```
ps -ef 是用标准的格式显示进程的(System V风格)
ps aux 是用BSD的格式来显示(BSD 风格)，aux会截断command列

ps aux | grep java

ps H -eo user,pid,ppid,tid,time,%cpu,cmd --sort=%cpu
ps -mp <pid> -o THREAD,tid,time
```

常用命令：
+ -A： 列出所有的行程 
+ a： 显示现行终端机下的所有程序，包括其他用户的进程
+ u： 以用户为主的格式来显示程序状况
+ x： 显示所有程序，不以终端机来区分，可列出较完整信息
+ e： 列出程序时，显示每个程序所使用的环境变量
+ -e： 此参数的效果和指定"A"参数相同
+ -m / m： 显示所有的执行绪
+ 输出格式：
	+ l / -l： 较长、较详细的将该PID 的的信息列出
	+ j / -j： 工作的格式 (jobs format)
	+ -f:  全格式输出，生成一个完整列表
	+ -o： 用户自定义格式

```	
********* simple selection *********  ********* selection by list *********
-A all processes                      -C by command name
-N negate selection                   -G by real group ID (supports names)
-a all w/ tty except session leaders  -U by real user ID (supports names)
-d all except session leaders         -g by session OR by effective group name
-e all processes                      -p by process ID
                                      -q by process ID (unsorted & quick)
T  all processes on this terminal     -s processes in the sessions given
a  all w/ tty, including other users  -t by tty
g  OBSOLETE -- DO NOT USE             -u by effective user ID (supports names)
r  only running processes             U  processes for specified users
x  processes w/o controlling ttys     t  by tty
*********** output format **********  *********** long options ***********
-o,o user-defined  -f full            --Group --User --pid --cols --ppid
-j,j job control   s  signal          --group --user --sid --rows --info
-O,O preloaded -o  v  virtual memory  --cumulative --format --deselect
-l,l long          u  user-oriented   --sort --tty --forest --version
-F   extra full    X  registers       --heading --no-heading --context
                                      --quick-pid
                    ********* misc options *********
-V,V  show version      L  list format codes  f  ASCII art forest
-m,m,-L,-T,H  threads   S  children in sum    -y change -l format
-M,Z  security data     c  true command name  -c scheduling class
-w,w  wide output       n  numeric WCHAN,UID  -H process hierarchy
```

常用字段：
+ 待


### 补充
```
监控java线程数：
ps -eLf | grep java | wc -l

监控网络客户连接数：
netstat -n | grep tcp | grep <侦听端口> | wc -l

输出进程内存的状况，可以用来分析线程堆栈：
pmap <PID>

查看包含内存每个项目的报告，通过-S M或-S k可以指定查看的单位，默认为kb。结合watch命令就可以看到动态变化的报告;
vmstat -s -S M  

查看cpu波动情况，尤其是多核机器上：
mpstat -P ALL 10 

IO实时监控查看所有设备使用率、读写字节数等信息
iostat -P ALL  

IO实时监控只查看均值的：
iostat -c
```



## windows下查看

### pslist命令

```
pslist查看java进程详情：
pslist | findstr java

查看线程信息：
pslist -x <pid>
```

下载pstools压缩包，解压后复制到C:\Windows\System32目录下  

参数：
+ /?： 查看用法
+ -d:  显示线程详情
+ -m:  显示内存详情
+ -x:  显示线程和内存详情


### tasklist命令

> 该工具显示在本地或远程机器上当前运行的进程列表

```
tasklist | findstr java

tasklist /? 查看用法
```

### ProcessExplorer

> 由Sysinternals开发的Windows系统和应用程序监视工具，目前已并入微软旗下。不仅结合了Filemon（文件监视器）和Regmon（注册表监视器）两个工具的功能，还增加了多项重要的增强功能。包括稳定性和性能改进、强大的过滤选项、修正的进程树对话框（增加了进程存活时间图表）、可根据点击位置变换的右击菜单过滤条目、集成带源代码存储的堆栈跟踪对话框、更快的堆栈跟踪、可在 64位 Windows 上加载 32位 日志文件的能力、监视映像（DLL和内核模式驱动程序）加载、系统引导时记录所有操作等。



# 2.查看线程信息

#### 将tid转换为16进制

Linux下:
`printf "%x\n" <tid>`

Windows下：
`计算器进行转换`

## jstack查看(thread dump)

> jstack是java虚拟机自带的一种堆栈跟踪工具。jstack用于打印出给定的java进程ID或core file或远程调试服务的Java堆栈信息，如果是在64位机器上，需要指定选项"-J-d64"，Windows的jstack使用方式只支持以下的这种方式：jstack [-l] pid

```
jstack <pid> | grep <tid> -A 30

jstack [-l] <pid> > ./jstack.log
```

主要分为两个功能： 
- 针对活着的进程做本地的或远程的线程dump
- 针对core文件做线程dump

jstack用于生成java虚拟机当前时刻的**线程快照**。线程快照是**当前java虚拟机内每一条线程正在执行的方法堆栈的集合**，生成线程快照的主要目的是定位线程出现长时间停顿的原因，如**线程间死锁、死循环、请求外部资源导致的长时间等待等**。 线程出现停顿的时候通过jstack来查看各个线程的**调用堆栈**，就可以知道没有响应的线程到底在后台做什么事情，或者等待什么资源。 如果java程序崩溃生成core文件，jstack工具可以用来获得core文件的java stack和native stack的信息，从而可以轻松地知道java程序是如何崩溃和在程序何处发生问题。

```
jstack [ option ] pid
jstack [ option ] executable core
jstack [ option ] [server-id@]remote-hostname-or-IP
```

基本参数：
+ -F： 强制dump线程堆栈信息. 用于进程hung住，jstack <pid>命令没有响应的情况。一般情况不需要使用
+ -l： 长列表. 打印关于锁的附加信息，会使得JVM停顿得长久得多。一般情况不需要使用
+ -m： 打印java和native c/c++框架的所有栈信息，可以打印JVM的堆栈，显示上Native的栈帧，一般应用排查不需要使用

#### 线程状态

>**NEW**： 未启动的。不会出现在Dump中。  
**RUNNABLE**： 在虚拟机内执行的。运行中状态，可能里面还能看到locked字样，表明它获得了某把锁。 这个名字很具有欺骗性，很容易让人误以为处于这个状态的线程正在运行。事实上，这个状态只是表示，线程是可运行的。   
**BLOCKED**(on object monitor)： 受阻塞并等待监视器锁。被某个锁(synchronizers)給block住了，正在等待一个monitor lock。通常情况下，是因为本线程与其他线程公用了一个锁。   
**WATING**： 无限期等待另一个线程执行特定操作。等待某个condition或monitor发生，一般停留在park(), wait(), join() 等语句里。Object.wait()方法只能够在同步代码块中调用。调用了wait()方法后，会释放锁。  
Object.wait 不指定超时时间 # java.lang.Thread.State: WAITING (on object monitor)  
Thread.join with no timeout  
LockSupport.park #java.lang.Thread.State: WAITING (parking)  
**TIMED_WATING**： 有时限的等待另一个线程的特定操作。和WAITING的区别是wait() 等语句加上了时间限制 wait(timeout)。  
Thread.sleep #java.lang.Thread.State: TIMED_WAITING (sleeping)  
Object.wait 指定超时时间 #java.lang.Thread.State: TIMED_WAITING (on object monitor)  
Thread.join with timeout  
LockSupport.parkNanos #java.lang.Thread.State: TIMED_WAITING (parking)  
LockSupport.parkUntil #java.lang.Thread.State: TIMED_WAITING (parking)  
**TERMINATED**： 已退出的。线程终止。

对于 java.lang.Thread.State: WAITING (on object monitor)和java.lang.Thread.State: TIMED_WAITING (on object monitor)，对于这两个状态，是因为调用了Object的wait方法(前者没有指定超时，后者指定了超时)，由于wait方法肯定要在syncronized代码中编写，因此肯定是如类似以下代码导致：
```
synchronized(obj) {
	...
	obj.wait();
	...
}

因此通常的堆栈信息中，必然后一个lock标记：
"CM1" #21 daemon prio=5 os_prio=0 tid=0x00007f02f0d6d800 nid=0x3d48 in Object.wait() [0x00007f02fefef000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        at java.lang.Object.wait(Object.java:502)
        at java.util.TimerThread.mainLoop(Timer.java:526)
        - locked <0x00000000eca75aa8> (a java.util.TaskQueue)
        at java.util.TimerThread.run(Timer.java:505)
```

#### Monitor

Monitor是Java中用以实现线程之间的互斥与协作的主要手段，它可以看成是对象或者Class的锁。每一个对象都有，也仅有一个monitor。下 面这个图，描述了线程和Monitor之间关系，以及线程的状态转换图:
![java Monitor](/img/in-post/2018/6/java-monitor.jpg)

>**进入区(Entrt Set)**： 表示线程通过synchronized要求获取对象的锁。如果对象未被锁住,则迚入拥有者;否则则在进入区等待。一旦对象锁被其他线程释放,立即参与竞争。  
**拥有者(The Owner)**： 表示某一线程成功竞争到对象锁。  
**等待区(Wait Set)**： 表示线程通过对象的wait方法,释放对象的锁,并在等待区等待被唤醒。

#### 调用修饰

表示线程在方法调用时,额外的重要的操作。线程Dump分析的重要信息。修饰上方的方法调用。
>**locked &lt;地址> 目标**： 通过synchronized关键字,成功获取到了对象的锁,成为监视器的拥有者,在临界区内操作。对象锁是可以线程重入的。  
**waiting to lock &lt;地址> 目标**： 通过synchronized关键字,没有获取到了对象的锁,线程在监视器的进入区等待。在调用栈顶出现,线程状态为**Blocked**。  
**waiting on &lt;地址> 目标**： 通过synchronized关键字,成功获取到了对象的锁后,调用了wait方法,进入对象的等待区等待。在调用栈顶出现,线程状态为**WAITING**或**TIMED_WATING**。  
**parking to wait for &lt;地址> 目标**： park是基本的线程阻塞原语,不通过监视器在对象上阻塞。随concurrent包会出现的新的机制,不synchronized体系不同。

#### 线程动作

线程状态产生的原因：
>**runnable**： 状态一般为RUNNABLE。  
**in Object.wait()**： 等待区等待,状态为WAITING或TIMED_WAITING。  
**waiting for monitor entry**： 进入区等待,状态为BLOCKED。  
**waiting on condition**： 等待区等待、被park。  
**sleeping**： 休眠的线程,调用了Thread.sleep()。

Wait on condition 该状态出现在线程等待某个条件的发生。具体是什么原因，可以结合stacktrace来分析。 最常见的情况就是线程处于sleep状态，等待被唤醒；还有等待网络IO。

#### 线程dump的分析工具

+ [IBM Thread and Monitor Dump Analyze for Java](https://www.ibm.com/developerworks/community/groups/service/html/communitystart?communityUuid=2245aa39-fa5c-4475-b891-14c205f7333c)  一个小巧的Jar包，能方便的按状态，线程名称，线程停留的函数排序，快速浏览。
+ [http://spotify.github.io/threaddump-analyzer](http://spotify.github.io/threaddump-analyzer)  Spotify提供的Web版在线分析工具，可以将锁或条件相关联的线程聚合到一起。


# 3.分析

原则：
> 结合代码阅读的推理。需要线程Dump和源码的相互推导和印证。
造成Bug的根源往往不会在调用栈上直接体现,一定格外注意线程当前调用之前的所有调用。

## 线程Dump分析入手点

#### 值得关注信息

死锁： **Deadlock**  
执行中： Runnable  
等待资源： **Waiting on condition**  
等待获取监视器： **Waiting on monitor entry**  
暂停： Suspended  
对象等待中： Object.wait() 或 TIMED_WAITING  
阻塞： **Blocked**  
停止： Parked

daemon： 表示线程是否是守护线程  
prio： 表示我们为线程设置的优先级  
tid： 是java中为这个线程的id  
nid： 是这个线程对应的操作系统本地线程id，每一个java线程都有一个对应的操作系统线程

### 进入区等待

```
"Thread-496" prio=6 tid=0x000000001e8eb000 nid=0x1a40 waiting for monitor entry [0x000000003747e000]
   java.lang.Thread.State: BLOCKED (on object monitor)
	at java.lang.ThreadGroup.threadTerminated(ThreadGroup.java:942)
	- waiting to lock <0x00000007d6083758> (a java.lang.ThreadGroup)
	at java.lang.Thread.exit(Thread.java:755)
```

线程状态 BLOCKED,线程动作 wait on monitor entry,调用修饰 waiting to lock
总是一起出现。表示在代码级别已经存在冲突的调用。必然有问题的代码,需要尽可能减少其发生。
waiting for monitor entry [0x000000003747e000]说明线程是通过synchronized关键字进入了监视器的临界区，并处于"Entry Set"队列，等待monitor，具体实现可以参考[深入分析synchronized的JVM实现](https://www.jianshu.com/p/c5058b6fe8e5)。

### 同步块阻塞

一个线程锁住某对象,大量其他线程在该对象上等待。

```
"Thread-491" prio=6 tid=0x000000001e8b6000 nid=0x2364 runnable [0x000000003703f000]
   java.lang.Thread.State: RUNNABLE
	at java.lang.System.arraycopy(Native Method)
	at java.lang.ThreadGroup.remove(ThreadGroup.java:969)
	- locked <0x00000007d6083758> (a java.lang.ThreadGroup)

"Thread-496" prio=6 tid=0x000000001e8eb000 nid=0x1a40 waiting for monitor entry [0x000000003747e000]
   java.lang.Thread.State: BLOCKED (on object monitor)
	at java.lang.ThreadGroup.threadTerminated(ThreadGroup.java:942)
	- waiting to lock <0x00000007d6083758> (a java.lang.ThreadGroup)
"Thread-495" prio=6 tid=0x000000001e8ea000 nid=0x26f4 waiting for monitor entry [0x00000000369af000]
   java.lang.Thread.State: BLOCKED (on object monitor)
	at java.lang.ThreadGroup.threadTerminated(ThreadGroup.java:942)
	- waiting to lock <0x00000007d6083758> (a java.lang.ThreadGroup)
```
持续运行的IO，IO操作是可以以 RUNNABLE状态达成阻塞。例如:数据库死锁、网络读写。格外注意对IO线程的真实状态的分析。 一般来说,被捕捉到 RUNNABLE的IO调用,都是有问题的。

以下堆栈显示： 线程状态为 RUNNABLE。调用栈在SocketInputStream或SocketImpl上,socketRead0等方法。 调用栈包含了jdbc相关的包。很可能发生了数据库死锁。

```
"Thread-211" daemon prio=6 tid=0x0000000022f1f000 nid=0x37c8 runnable
[0x0000000027cbd000]
java.lang.Thread.State: RUNNABLE
at java.net.SocketInputStream.socketRead0(Native Method)
at java.net.SocketInputStream.read(Unknown Source)
at oracle.net.ns.Packet.receive(Packet.java:240)
at oracle.net.ns.DataPacket.receive(DataPacket.java:92)
at oracle.net.ns.NetInputStream.getNextPacket(NetInputStream.java:172)
at oracle.net.ns.NetInputStream.read(NetInputStream.java:117)
at oracle.jdbc.driver.T4CMAREngine.unmarshalUB1(T4CMAREngine.java:1034)
at oracle.jdbc.driver.T4C8Oall.receive(T4C8Oall.java:588)
```

### 分线程调度的休眠

正常的线程池等待
```
"Thread-211" in Object.wait()
java.lang.Thread.State: TIMED_WAITING (on object monitor)
at java.lang.Object.wait(Native Method)
at com.test.core.impl.WorkingManager.getWorkToDo(WorkingManager.java:322)
- locked <0x0000000313f656f8> (a com.test.core.impl.WorkingThread)
at com.test.core.impl.WorkingThread.run(WorkingThread.java:40)
```
可疑的线程等待
```
"Thread-212" in Object.wait()
java.lang.Thread.State: WAITING (on object monitor)
at java.lang.Object.wait(Native Method)
at java.lang.Object.wait(Object.java:485)
at com.test.core.impl.AcquirableAccessor.exclusive()
- locked <0x00000003011678d8> (a com.test.core.impl.CacheGroup)
at com.test.core.impl.Transaction.lock()
```

#### 入手点总结

wait on monitor entry： 被阻塞的,肯定有问题
runnable ： 注意IO线程
in Object.wait()： 注意非线程池等待

## 死锁分析

```java
	public static Lock lockA = new ReentrantLock();
    public static Lock lockB = new ReentrantLock();

	Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    while(true) {
                        synchronized (lockA) {
                            System.out.println("t1 locked a");
                            Thread.sleep(1000 * 1);
                            synchronized (lockB) {
                                System.out.println("t1 locked b");
                            }
                        }
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
	Thread t2 = new Thread(new Runnable() {
		@Override
		public void run() {
			try {
				while (true) {
					synchronized (lockB) {
						System.out.println("t2 locked b");
						Thread.sleep(1000 * 1);
						synchronized (lockA) {
							System.out.println("t2 locked a");
						}
					}
				}
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	});
	t1.start();
	t2.start();
```
```
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x000000000ae23bc8 (object 0x00000007d6437578, a java.util.concurrent.locks.ReentrantLock),
  which is held by "Thread-0"
"Thread-0":
  waiting to lock monitor 0x000000000ae26668 (object 0x00000007d64375a8, a java.util.concurrent.locks.ReentrantLock),
  which is held by "Thread-1"

Java stack information for the threads listed above:
===================================================
"Thread-1":
	at com.zj.Test$2.run(Test.java:67)
	- waiting to lock <0x00000007d6437578> (a java.util.concurrent.locks.ReentrantLock)
	- locked <0x00000007d64375a8> (a java.util.concurrent.locks.ReentrantLock)
	at java.lang.Thread.run(Thread.java:745)
"Thread-0":
	at com.zj.Test$1.run(Test.java:45)
	- waiting to lock <0x00000007d64375a8> (a java.util.concurrent.locks.ReentrantLock)
	- locked <0x00000007d6437578> (a java.util.concurrent.locks.ReentrantLock)
	at java.lang.Thread.run(Thread.java:745)

Found 1 deadlock.
```

## 其他

虚拟机执行Full GC时,会阻塞所有的用户线程。因此,即时获取到同步锁的线程也有可能被阻塞。 
在查看线程Dump时,首先查看内存使用情况。

对于jstack做的ThreadDump的栈，可以反映如下信息（[源自](https://blog.csdn.net/axman/article/details/7104819)）：

如果某个相同的call stack经常出现， 我们有80%的以上的理由确定这个代码存在性能问题（读网络的部分除外）；
如果相同的call stack出现在同一个线程上（tid）上， 我们很很大理由相信， 这段代码可能存在较多的循环或者死循环；
如果某call stack经常出现， 并且里面带有lock，请检查一下这个lock的产生的原因， 可能是全局lock造成了性能问题；
在一个不大压力的群集里（w<2）， 我们是很少拿到带有业务代码的stack的， 并且一般在一个完整stack中， 最多只有1-2业务代码的stack，
如果经常出现， 一定要检查代码， 是否出现性能问题。
如果你怀疑有dead lock问题， 那么请把所有的lock id找出来，看看是不是出现重复的lock id。

jstack -m 会打印出JVM堆栈信息，涉及C、C++部分代码，可能需要配合gdb命令来分析。

### 频繁GC问题或内存溢出问题

1. 使用jps查看线程ID
2. 使用jstat -gc 3331 250 20 查看gc情况，一般比较关注PERM区的情况，查看GC的增长情况
3. 使用jstat -gccause：额外输出上次GC原因
4. 使用jmap -dump:format=b,file=heapDump 3331生成堆转储文件
5. 使用jhat或者可视化工具（Eclipse Memory Analyzer 、IBM HeapAnalyzer）分析堆情况。
6. 结合代码解决内存溢出或泄露问题。
