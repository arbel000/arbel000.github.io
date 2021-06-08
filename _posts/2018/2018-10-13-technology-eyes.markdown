---
layout:     post
title:      "技术之瞳-读书笔记"
date:       2018-10-13
author:     "ZhouJ000"
header-img: "img/in-post/2018/post-bg-2018-headbg.jpg"
catalog: true
tags:
    - 读书笔记
--- 

<font id="last-updated">最后更新于：2018-10-13</font>


# 计算机科学

#### 计算机网络

IP协议(以及相关协议)、ISO七层模型

网络速率单位bit/s，存储容量单位B，需要注意转换

**TCP状态转换图(来自互联网)**
![tcp_status](/img/in-post/2018/10/tcp_status.jpg)

BDP(带宽 X 时延 = 缓冲区大小)  
TCP协议的一个关键特性是可靠传输，抽象过程:  
1、发送端把待发送数据存入发送缓冲区  
2、网络设备发送数据  
3、接收端接收数据，同时返回一个确认收到数据的ACK信息  
4、发送端受到ACK信息之后确认数据已被收到，缓冲区的已确定数据删除(等待时长是一个网络RTT<往返时间>)

在长距离(>50km)传输中网络RTT主要是光信号的传输过程，网络设备和主机软件栈处理消耗的时间其实已经占比很小。光在介质中的传输速率还需要除以该介质的折射率。一般光信号在光纤中传播，同时光信号在光纤中是折线路程传输，这样综合2x100000km/s信息传输速率，工程上一般认为100km的RTT为1ms

一般操作系统中TCP发送缓冲区默认值是4MB左右(有一定自调节能力)  
1、对于终端用户的互联网应用，虽然RTT很高，但是带宽不会很高，所以缓冲区大小也不会很大  
2、对于数据中心应用。虽然带宽非常高(1Gbit/s很常见，10Gbit/s也在普及)，但是机房网络好，RTT非常低(0.1ms级别)
3、在大型网络应用系统中才会出现异地数据中心高带宽的情况，这时候BDP问题需要正确处理

#### 计算机组成原理

一般计算机系统基本组成元素:  用于程序和数据存储的**存储器**、负责程序命令执行的**执行单元**、负责协调整体执行过程的**控制器**、辅助的**I/O设备及接口**

小数部分的二进制转换为: 小数部分循环乘以2取整

LRU淘汰算法(最近最少使用，常用于页面置换算法，是为虚拟页式存储管理服务的)。LRU淘汰策略有两种实现： 缓存访问命中后，是否要这个数据缓存项到LRU的最前端。具体业界的实际策略设计(主要考虑维度有命中率、场景适应、实现复杂度、实现性能等)

<table>
    <tr>
        <th>名称</th>
        <th>名称</th>
        <th>速度</th>
    </tr>
    <tr>
        <td>L1 cache reference</td>
        <td>L1缓存</td>
        <td>0.5 ns</td>
    </tr>
	<tr>
        <td>Branch mispredict</td>
        <td>转移、分支预测</td>
        <td>5 ns</td>
    </tr>
	<tr>
        <td>L2 cache reference</td>
        <td>L2缓存</td>
        <td>7 ns</td>
    </tr>
	<tr>
        <td>Mutex lock/unlock</td>
        <td>互斥锁/解锁</td>
        <td>25 ns</td>
    </tr>
	<tr>
        <td>Main memory reference</td>
        <td>主内存</td>
        <td>100 ns</td>
    </tr>
	<tr>
        <td>Compress 1K bytes with Zippy</td>
        <td>1k字节压缩(Zippy)</td>
        <td>3,000 ns</td>
    </tr>
	<tr>
        <td>Send 2K bytes over 1 Gbps network</td>
        <td>在1Gbps的网络上传送2k字节</td>
        <td>20,000 ns</td>
    </tr>
	<tr>
        <td>SSD random read</td>
        <td>SSD随机读</td>
        <td>150,000 ns</td>
    </tr>
	<tr>
        <td>Read 1 MB sequentially from memory</td>
        <td>从内存顺序读取1MB</td>
        <td>250,000 ns</td>
    </tr>
	<tr>
        <td>Round trip within same datacenter</td>
        <td>同一个数据中心往返RTT</td>
        <td>500,000 ns</td>
    </tr>
	<tr>
        <td>Read 1 MB sequentially from SSD</td>
        <td>从SSD顺序读取1MB</td>
        <td>1,000,000 ns</td>
    </tr>
	<tr>
        <td>Disk seek</td>
        <td>磁盘寻道时间</td>
        <td>10,000,000 ns</td>
    </tr>
	<tr>
        <td>Read 1 MB sequentially from network</td>
        <td>从网络上顺序读取1MB</td>
        <td>10,000,000 ns</td>
    </tr>
	<tr>
        <td>Read 1 MB sequentially from disk</td>
        <td>从磁盘上顺序读取1MB</td>
        <td>30,000,000 ns</td>
    </tr>
	<tr>
        <td>Send packet CA->Netherlands->CA</td>
        <td>发送一个包从美国-荷兰-美国</td>
        <td>150,000,000 ns</td>
    </tr>
</table>

#### 操作系统和分布式

主要负责管理和控制系统的各种资源: 内存资源、计算资源、设备管理、文件系统

内核态，用户态。应用代码从用户态执行模式进入内核态执行模式的方式是通过系统调用达成的:  
1、准备好调用参数  
2、系统调用中断进入内核态  
3、内核态执行  
4、内核态的返回数据复制到用户态  
5、用户态得到调用结果

页式虚拟内存管理(LRU算法)

Linux基于用户(组)作为访问实体的权限控制体系，根据资源、程序所属用户进行权限控制。对于程序还会涉及当前执行用户、程序所属用户

程序文件的权限控制位中有一个setuid标识:
1、setuid置位，那么在执行这个程序的时候按照这个程序的属主进行权限控制
2、setuid未置位，那么在执行这个程序的时候按照当前启动这个程序的用户进行权限控制

"无锁化编程": CAS、RCU..

#### 算法和数据结构

集合、线性结构、树形结构、图结构

#### 编程语言

# 数学算法

#### 逻辑

数字逻辑与数理逻辑

#### 排列组合

分析与归纳能力

#### 概率基础与数理统计

转换为xy图计算

#### 最优化算法

#### 博弈与策略

#### 机器学习


# web前端开发

#### HTTP协议

HTTPS: SSL + HTTPS

HTTP状态码

#### HTML、CSS、JavaScript

#### 数据结构与算法、正则

#### node.js

构建于V8引擎之上，采用事件驱动、非阻塞IO模型，具有轻量、高性能。适用于面向分布式网络的应用程序

#### 前端框架

Jquery、Angular、React、(VUE?)

#### 前端工程化

#### 数据可视化


# 数据分析与挖掘

#### 统计分析

Excel

SPPS、SAS、R、Matlab

Hadoop、Spark、ODPS、Mahout、Xlab等

无论使用哪种工具，算法的原理、问题的本质，还有就是潜在模式的探索、描述、构建和数学归纳等，才是统计分析和数学建模中最核心的部分

#### 商业分析

1. 逻辑思维能力
	+ 规律，事物的完成序列
	+ 事物流动的顺序规律
	+ 事物传递信息，并得到解释的过程
	+ 《金字塔原理》、《逻辑思维》等书籍
2. 商业洞察能力，以及行业信息触角，视野格局
	+ 《哈佛商业评论》、《商业价值》
3. 框架结构和结论抽取提炼能力
4. 统计分析、模型开发能力


# 安全

#### web应用安全

SSL技术加密数据，防火墙，IDS(入侵检测系统)，IPS(入侵防御系统)，身份认证机制

客户端脚本安全、服务端应用安全

#### 系统与网络安全

内核安全，权限控制，系统组件安全，基线检查和网络协议安全等

#### 逆向与测试


# 产品

DAU(daily active user)  日活跃用户数量  
PV(Page View)/UV(Unique Visitor)  人均点击数  
QPS(Query Per Second)  每秒查询率(并发量 / 平均响应时间)  
TPS(Transactions Per Second)  每秒钟处理完的事务次数  
Bounce Rate  跳失率  
CPC(cost per click)  每点击成本  
Time on Site  网站访问时长  
CTR(click through rate)  点击率  

# 交互设计


