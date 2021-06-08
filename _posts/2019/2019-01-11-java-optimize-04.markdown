---
layout:     post
title:      "Java性能优化04-调优工具"
date:       2019-01-11
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

[java项目 CPU占用100%问题](https://zhouj000.github.io/2018/06/24/investigate-cpu-100/)  
[java项目 内存溢出问题](https://zhouj000.github.io/2018/07/14/investigate-out-memery/)  



# Linux命令行工具

top命令，是Linux常用的性能分析工具，能够实时显示系统中各个进程的资源占用状况

sar命令，也是Linux中重要的性能检测工具之一，它可以周期性地对内存和CPU使用情况进行采样

vmstat命令，是一款功能比较齐全的性能检测工具，它可以统计CPU、内存使用情况、swap使用等情况。和sar类似也可以指定采样周期和采样次数

iostat命令，可以提供详尽的I/O信息

pidstat工具，是一个功能强大的性能检测工具，也是Sysstat的组件之一。可以进行CPU使用率监控、I/O使用监控、内存监控


# Windows工具

任务管理器，最为熟知的一款系统工具

perfmon性能监控工具，可以说是Windows下专业级的性能监控工具，不仅可以监控计算机系统的整体运行情况，也可以专门针对某一个进程或线程进行状态监控

Process Explorer，功能强大的进程管理工具，完全可以替代Windows自带的任务管理器

pslist命令行，可以显示进程乃至线程的详细信息


# JDK命令行工具

jps命令，类似Linux下的ps，但它只会列出Java的进程

jstat命令，用于观察Java应用程序运行时信息的工具，可以通过它查看堆信息的详细情况

jinfo命令，可以用来查看正在运行的Java应用程序的扩展参数，甚至还支持在运行时修改部分参数

jmap命令，可以生成Java应用程序的堆快照和对象的统计信息

jhat命令，可以用于分析堆快照内容，使用HTTP服务器展示其分析结果

jstack命令，可以导出Java应用程序的线程堆栈

jstatd命令，是一个RMI服务端程序，它的作用相当于代理服务器，建立本地计算机与远程监控工具的通信。为了启用远程监控，需要配合使用jstatd工具

hprof工具，它不是独立的监控工具，它只是一个Java agent工具，可以用于监控Java应用程序在运行时的CPU信息和堆信息


# JConsole工具

是JDK自带的图形化性能监控工具，可以查看Java应用程序的运行概况、线程的堆栈信息、监控堆信息、性能分析、快照等

![jconsole](/img/in-post/2019/01/jconsole.png)



# Visual VM多合一工具

是一个功能强大的多合一故障诊断和性能监控的可视化工具，集成多种性能统计工具的功能，可以代替jstat、jmap、jhat、jstack甚至是JConsole

![visualvm](/img/in-post/2019/01/visualvm.png)
还可以安装许多插件，比如TDA、BTrace等，在对内存快照分析时，还可以使用OQL语言快速定位所需的资源



# MAT内存分析工具

MAT是基于Eclipse开发的，一款功能强大的Java堆内存分析工具，可以用于查找内存泄露以及查看内存消耗情况。

《[java项目 内存溢出问题](https://zhouj000.github.io/2018/07/14/investigate-out-memery/)》中有关于MAT的使用介绍

浅堆(Shallow Heap)和深堆(Retained Heap)是两个非常重要的概念，它们分别表示一个对象结构所占用的内存大小和一个对象被GC回收后，可以真实释放的内存大小

浅堆是指一个对象所消耗的内存。在32位系统中，一个对象引用会占4个字节，一个int类型会占据4个字节，long会占用8个字节，每个对象头需要占用8个字节。根据堆快照格式不同，对象的大小可能会向8个字节进行对齐。以String对象为例，3个int类型与一个引用类型char[]合计占用内存3X4+4=16个字节，再加上对象头8个字节，因此String对象所占用的空间，即浅堆的大小是24字节。浅堆的大小只与对象的结构有关，与对象的实际内容无关，也就是无论字符串长度是多少，内容是什么，浅堆的大小始终是24字节

深堆概念略复杂，需要了解保留集(Retained Set)。对象A的保留集指当对象A被垃圾回收后，可以被释放的所有的对象集合(包括对象A本身)，即对象A的保留集可以被认为是只能通过对象A被直接或间接访问到的所有对象的集合。深堆是指对象的保留集中所有的对象的浅堆大小之和

MAT还提供了支配树的对象图、GCRoots、内存泄露检测(Leak Suspects)、最大对象报告(Top Consumers)、查找支配者(Immediate Dominators)、线程分析(Thread Details)、集合使用情况分析(Java Collections)以及MAT扩展插件。同时MAT也对OQL语言支持，与VisualVM的OQL不同，MAT的OQL在语法上更接近SQL语句



# JProfile

一款商业产品，主要功能有内存分析、快照分析、CPU分析、线程分析和JVM性能信息收集等

![jprofile](/img/in-post/2019/01/jprofile.png)


引用：《Java程序性能优化》
