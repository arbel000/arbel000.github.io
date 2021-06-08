---
layout:     post
title:      "重看RocketMQ源码(一)"
date:       2021-05-10
author:     "ZhouJ000"
header-img: "img/in-post/2021/post-bg-2021-headbg.jpg"
catalog: true
tags:
    - mq
--- 



# 概念

![mq0](/img/in-post/2021/05/mq0.png)
![mq00](/img/in-post/2021/05/mq00.png)

+ **异步**：在一对多调用时由消息系统通知。如下单核心流程环节太多，性能较差
+ **解耦**：和第三方系统耦合在一起，性能存在抖动的风险。解决不同重要程度/能力级别系统之间依赖
+ **削峰填谷**：解决瞬时写压力大于应用服务能力导致消息丢失、系统奔溃等问题，例如秒杀活动
+ **失败重试**：业务调用失败风险
+ **延时消息**：比如关闭过期订单的时候，存在扫描大量订单数据的问题
+ **监听BinLog发送到MQ**：数据同步到其他NoSQL

RocketMQ优势：
+ 支持事务型消息：消息发送和DB操作保持两方的最终一致性
	- rabbitmq和kafka不支持
+ 支持结合RocketMQ的多个系统之间数据最终一致性
	- 前提：多方事务，二方事务
+ 支持18个级别的延迟消息
	- rabbitmq和kafka不支持
+ 支持指定次数和时间间隔的失败消息重发
	- kafka不支持，rabbitmq需要手动确认
+ 支持consumer端tag过滤，减少不必要的网络传输
	- rabbitmq和kafka不支持
+ 支持重复消费
	- rabbitmq不支持，kafka支持

![mq1](/img/in-post/2021/05/mq1.png)
![clipboard](/img/in-post/2021/05/clipboard.png)

+ Name Server：是一个**几乎无状态**节点，可集群部署，节点之间**无任何信息同步**。每10s**定期清理**超过2分钟未上报心跳的broker
+ Broker：部署相对复杂，Broker分为Master与Slave，一个Master可以对应多个Slave，但是一个Slave只能对应一个Master，Master 与Slave的对应关系通过指定**相同的**BrokerName，**不同的**BrokerId来定义，BrokerId为0表示Master，非0表示Slave。Master和Slave都可以部署多个。每个Broker与Name Server集群中的**所有**节点建立**长连接**，每隔30s**注册**Topic信息到所有Name Server
+ Producer：与Name Server集群中的其中一个节点(随机选择)建立长连接。每30s从Name Server取Topic路由信息，并向提供Topic服务的Broker **Master**建立**长连接**，每30s向Master发送心跳。Producer**完全无状态**，可集群部署
	- Producer每隔30s(由ClientConfig的pollNameServerInterval)从Name server获取所有topic队列的最新情况，这意味着如果Broker不可用，Producer最多30s能够感知，在此期间内发往Broker的所有消息都会**失败**
	- Producer每隔30s(由ClientConfig中heartbeatBrokerInterval决定)向所有关联的broker发送心跳，Broker每隔10s中扫描所有存活的连接，如果Broker在2分钟内没有收到心跳数据，则**关闭与Producer的连接**
+ Consumer：与Name Server集群中的其中一个节点(随机选择)建立长连接，每30s从Name Server取Topic路由信息，并向提供 Topic服务的Broker **Master/Slave**建立**长连接**，每30s向Master/Slave发送心跳。Consumer既可以从Master订阅消息，也可以从Slave订阅消息，订阅规则由Broker配置决定
	- Consumer每隔30s从Name server获取topic的最新队列情况，这意味着Broker不可用时，Consumer最多最需要30s才能感知
	- Consumer每隔30s(由ClientConfig中heartbeatBrokerInterval决定)向所有关联的broker发送心跳，Broker每隔10s扫描所有存活的连接，若某个连接2分钟内没有发送心跳数据，则**关闭连接**；并向该Consumer Group的**所有Consumer发出通知**，Group内的Consumer**重新分配队列**，然后继续消费
	- 当Consumer得到master宕机通知后，转向slave消费，slave不能保证master的消息100%都同步过来了，因此会有少量的消息**丢失**。但是一旦master恢复，未同步过去的消息会被**最终消费**掉

最佳部署实践：
+ 双主双从，同步复制异步刷盘
	- 异步刷盘ASYNC_FLUSH模式，flushCommitLogTimed要改为true，否则还是会实时刷盘
	- CommitLog异步刷盘默认刷盘间隔：500ms
	- ConsumeQueue默认刷盘间隔：1s

# 消息发送
	
#### 获取topic路由数据	

![getTopic](/img/in-post/2021/05/getTopic.png)

#### 负载均衡

**选择队列策略**：默认为所有队列轮询

**故障转移**：对之前失败的broker，避让一段时间。开启故障转移，则会选择上次消息投递延迟较小的队列，避免选中高延迟和发送失败的broker。如果当前没有无故障的queue，从延迟最短的一半broker中轮询

例如，如果上次请求的latency超过550Lms，就避让3000ms；超过1000ms，就退避60000ms

## 创建topic

#### 自动创建topic

允许自动创建的开关配置在BrokerConfig中，通过autoCreateTopicEnable字段进行控制

1. 生产者第一次发送消息，如果topic在nameServer中不存在
2. 第二次请求会设置`isDefault=true`，使用默认`topic = "TBW102"`从nameServer获取路由信息
3. 拉到broker信息后，构建本地缓存`realTopic(真实的Topic) -> TBW102`的queue信息
4. 使用realTopic往TBW102的queue中发送消息
5. broker接收到消息后，创建realTopic(真实的Topic被创建)信息加到本地缓存topicConfigTable
6. broker定时发送心跳将topicConfigTable发送给nameserver进行注册
7. 如果在心跳期间其他broker没有被生产者使用TBW102选中发消息，那么以后所有该topic的消息，都将发送到这台broker上，如果该topic消息量非常大，会造成某个broker上负载过大，这样的消息存储就达不到负载均衡的效果了

当broker接收到消息后，会在msgCheck方法中调用createTopicInSendMessageMethod方法，将topic的信息塞进topicConfigTable缓存中，并且broker会定时发送心跳将topicConfigTable发送给nameserver进行注册

#### 预先创建

预先创建需要通过mqadmin提供的topic相关命令进行创建

**顺序消息Queue的数量尽量提前预分配**，虽然可以在后期动态增加，但是可能会破坏Key和Queue之间对应关系打乱消费顺序

#### 通过broker模式创建与通过集群模式创建

用集群模式去创建topic时，集群里面每个broker的queue的数量相同，当用单个broker模式去创建topic时，每个broker的queue数量可以不一致

## 消息存储

**ReputMessageService**不停地分发请求并异步构建ConsumeQueue（逻辑消费队列）和IndexFile（索引文件）数据

![rms](/img/in-post/2021/05/rms.png)

#### ConsumeQueue

ConsumeQueue（逻辑消费队列）作为消费消息的索引，保存了指定Topic下的队列消息在CommitLog中的起始物理偏移量offset，消息大小size和消息Tag的HashCode值

**consumequeue文件**采取定长设计，每一个条目共20个字节，分别为8字节的commitlog物理偏移量、4字节的消息长度、8字节tag hashcode，单个文件由30W个条目组成，可以像数组一样随机访问每一个条目，每个ConsumeQueue文件大小约5.72M

#### IndexFile

![IndexFile](/img/in-post/2021/05/IndexFile.png)

IndexFile（索引文件）提供了一种可以通过key或时间区间来查询消息的方法

IndexFile的底层存储设计为在文件系统中实现HashMap结构，故rocketmq的索引文件其底层实现为hash索引

文件名fileName是以创建时的时间戳命名的，文件大小是固定的，等于`40+500W*4+2000W*20=420000040`个字节大小。如果消息设置了KEYS属性（多个KEY以空格分隔），会用`topic + “#” + KEY`来做索引

根据topic和key找到IndexFile索引文件中的一条记录，根据其中的commitLog offset从CommitLog文件中读取消息的实体内容

#### MMAP

![mmap](/img/in-post/2021/05/mmap.png)

页缓存（PageCache)是OS对文件的缓存，用于加速对文件的读写。一般来说，程序对文件进行顺序读写的速度几乎接近于内存的读写速度，主要原因就是由于OS使用PageCache机制对读写访问操作进行了性能优化，将一部分的内存用作PageCache。对于数据的写入，OS会先写入至Cache内，随后通过异步的方式由pdflush内核线程将Cache内的数据刷盘至物理磁盘上。对于数据的读取，如果一次读取文件时出现未命中PageCache的情况，OS从物理磁盘上访问读取文件的同时，会顺序对其他相邻块的数据文件进行预读取

在RocketMQ中，ConsumeQueue逻辑消费队列存储的数据较少，并且是顺序读取，在page cache机制的预读取作用下，Consume Queue文件的读性能几乎接近读内存，即使在有消息堆积情况下也不会影响性能。而对于CommitLog消息存储的日志数据文件来说，读取消息内容时候会产生较多的随机访问读取，严重影响性能。如果选择合适的系统IO调度算法，比如设置调度算法为“Deadline”（此时块存储采用SSD的话），随机读的性能也会有所提升

RocketMQ主要通过MappedByteBuffer对文件进行读写操作。其中，利用了NIO中的FileChannel模型将磁盘上的物理文件直接映射到用户态的内存地址中（这种Mmap的方式减少了传统IO将磁盘文件数据在操作系统内核地址空间的缓冲区和用户应用程序地址空间的缓冲区之间来回进行拷贝的性能开销），将对文件的操作转化为直接对内存地址进行操作，从而极大地提高了文件的读写效率（正因为需要使用内存映射机制，故RocketMQ的文件存储都使用定长结构来存储，方便一次将整个文件映射至内存）

#### 刷盘机制

![refreash](/img/in-post/2021/05/refreash.png)

#### 堆外内存池

Mmap+PageCache的方式，读写消息都走的是pageCache，这样子读写都在pagecache里面不可避免会有锁的问题，在并发的读写操作情况下，会增加缺页中断，内存加锁，污染页的回写

启用"读写"分离，消息发送时消息先追加到DirectByteBuffer(堆外内存)中，然后在异步刷盘机制下，会将DirectByteBuffer中的内容提交到PageCache，然后刷写到磁盘。消息拉取时，直接从PageCache中拉取，实现了读写分离，减轻了PageCaceh的压力

![mapfile](/img/in-post/2021/05/mapfile.png)

#### 异常恢复

根据abort文件,判断是否需要异常恢复

## 消息消费

总流程

![push-modle-sell](/img/in-post/2021/05/push-modle-sell.jpg)

#### ReBalance

![ReBalance](/img/in-post/2021/05/ReBalance.png)

+ AllocateMessageQueueAveragely：平均分配（默认）
+ AllocateMessageQueueAveragelyByCircle：轮询分配
+ AllocateMessageQueueConsistentHash：一致性hash分配

#### offset上报

+ 每次上报treeMap第一个节点key，防止未消费的offset被覆盖
+ 每5s保存消费进度
+ broker保存offset维度`topic@group -> queueId -> offset`

![offset](/img/in-post/2021/05/offset.png)

#### 解决Ack卡进度

每隔15分钟清理，treeMap中超过15分钟未commit的过期消息，并发送重试消息sendMessageBack进行重试

#### 消息过滤

Tag过滤:  
1、Broker端进行tag hash过滤  
2、Consumer端进行tag值过滤

设计亮点：
+ **Message Tag**存储Hashcode，是为了在Consume Queue定长方式存储，节约空间
+ 过滤过程中不会访问Commit Log数据，可以保证堆积情况下也能高效过滤
+ 即使存在Hash冲突，也可以在Consumer端进行**修正**，保证万无一失

SQL92的过滤:  
SQL expression的构建和执行由**rocketmq-filter模块**负责的。每次过滤都去执行SQL表达式会影响效率，所以RocketMQ使用了BloomFilter避免了每次都去执行。SQL92的表达式上下文为消息的属性

#### 长轮询

如果一个消息拉取请求未拉取到消息，Broker允许等待30s的时间，只要这段时间内有新消息到达，将直接返回给消费端

#### 一些坑

+ 消息消费15分钟以上还未成功，可能导致无限重复消费
	- 解决方法：异步去处理
	- 原因：超时重试，在处理的消息，会关闭，重新发一个固定时间的重试消息(重试队列默认15分钟发一次)
		+ 异常重试
		+ 超时重试

+ 同一个消费组，必须保持订阅关系一致
	- TAG不一致：复现
		1. 启动消费者1，消费组为group1，订阅topicA的消息，tag设置为`tag1 || tag2`
		2. 启动消费者2，消费组也为group1，也订阅topicA的消息，但是tag设置为tag3
		3. 启动生产者，生产者发送含有tag1,tag2,tag3的消息各10条
		4. 消费者1没有收到任何消息，消费者2收到部分消息
	- TAG不一致：原因
		+ 同一个consumer group的订阅关系，保存在Client RebalanceImpl类的Map中。key为topic
		+ 不同的消费者启动后，依次注册订阅关系，因为tag不一样，导致Map中同一topic的tag被覆盖。比如：消费者1订阅tag1，消费者2订阅tag2。最后map中只保存tag2
		+ 过滤的核心是tag，tag被更新，过滤条件被改变。服务端过滤后只返回tag2的消息
		+ 客户端接收消息后，再次过滤。先启动的消费者1订阅tagA，但是服务端返回tag2，所以消费者1收不到任何消息。消费者2能收到一半的消息（集群模式，假设消息平均分配，另外一半分给消费者1）
	- TOPIC不一致

NameServer的假死导致路由信息无法更新：  
一台物理机上分别部署了nameserver，broker两个进程。其中一台机器(192.168.3.100)的内存出现故障，导致机器重启，但Linux操作系统由于重启需要自检等因素，整个重启过程竟然持续了将近10分钟，客户端的发送超时持续10分钟

![ns-er1](/img/in-post/2021/05/ns-er1.jpg)

由于机器内存故障触发重启并且需要自检等因素，造成nameserver，broker无法再处理请求但底层TCP连接并未断开，超时后返回，但客户端并不会关闭与故障机器nameserver的TCP连接，不会触发切换NameServer，等到机器重新启动成功后，TCP连接断开，故障机器重启完成后感知路由信息变化，故障恢复

![ns-er2](/img/in-post/2021/05/ns-er2.png)

#### 延时消息

延时级别：1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h

![later](/img/in-post/2021/05/later.png)

#### 消息重试

topic：%RETRY_topic%

+ 并发、集群消费
	- 发送重试消息：延时级别= 重试次数+3
	- 重试消息发送失败：延时5s重新消费
+ 并发、广播消费
	- 不会重试
+ 顺序消费
	- 终止当前队列消费，延时1s重新消费(队列阻塞消费，1s重试一次)
	- 重试次数大于等于最大次数(默认16)，发送重试消息（broker对于重试次数超过16次的消息会放入死信队列）
	- 重试消息发送失败：继续阻塞重试

#### 死信队列

条件：消息重试次数大于等于最大次数(默认16)，算上第一次消费，总共16次

topic：`%DLQ% + consumerGroup`

## 事务消息

![trs-mq1](/img/in-post/2021/05/trs-mq1.png)
![trs-mq2](/img/in-post/2021/05/trs-mq2.png)
![trs-mq3](/img/in-post/2021/05/trs-mq3.png)
![trs-mq4](/img/in-post/2021/05/trs-mq4.png)

#### RemotingCommand协议

![RemotingCommand](/img/in-post/2021/05/RemotingCommand.png)

+ 4字节长度
+ 4字节序列化类型+header长度
	- 序列化类型：默认RemotingSerializable，使用fastjson
+ header
+ 消息体

#### Header格式

| Header字段 |           类型          |                         Request说明                  | Response说明 | 
| :--------- | :---------------------: | :--------------------------------------------------: | :----------: |
| code       | int                     | 请求操作码，应答方根据不同的请求码进行不同的业务处理 | 应答响应码。0表示成功，非0则表示各种错误 |
| language   | LanguageCode            | 请求方实现的语言                                     | 应答方实现的语言 |
| version    | int                     | 请求方程序的版本                                     | 应答方程序的版本 |
| opaque     | int                     | 相当于requestId，在同一个连接上的不同请求标识码，与响应消息中的相对应 | 应答不做修改直接返回 |
| flag       | int                     | 区分是普通RPC还是onewayRPC得标志                     | 区分是普通RPC还是onewayRPC得标志 |
| remark     | String                  | 请传输自定义文本信息                                 | 传输自定义文本信息 |
| extFields  | HashMap<String, String> | 请求方程序的版本                                     | 应答方程序的版本 |

#### Remoting通信类结构

![remoting-trs](/img/in-post/2021/05/remoting-trs.png)

#### Reactor线程模型

![mq-reactor](/img/in-post/2021/05/mq-reactor.png)
1、**Boss线程池**(NioEventLoopGroup，线程数为1，线程名称为NettyBoss_1)，负责监听TCP网络连接请求(OP_ACCEPT)，建立好连接，创建SocketChannel，并注册到selector上  
2、**Worker线程池**(NioEventLoopGroup或EpollEventLoopGroup，线程数默认为3)，负责监听OP_READ事件，当网络数据可读以后，**提交任务**到executor线程池  
3、**Executor线程池**(DefaultEventExecutorGroup，线程数默认为8)，真正执行业务逻辑之前需要进行SSL验证、编解码、空闲检查、网络连接管理  
4、**业务线程池**，根据RomotingCommand的业务请求码code去**processorTable本地缓存**中找到对应的processor，然后封装成Runnable任务后，提交给对应**业务processor的处理线程池**来执行

![mq-reactor2](/img/in-post/2021/05/mq-reactor2.png)
1、Reactor主线程在端口上监听Producer建立连接的请求，建立长连接  
2、Reactor线程池并发的监听多个连接的请求是否到达  
3、Worker线程池请求并发的对多个请求进行预处理  
4、业务线程池并发的对多个请求进行磁盘读写业务操作

![mq-reactor3](/img/in-post/2021/05/mq-reactor3.png)

## 常见问题

#### 解决拆包粘包

如何区分一个整包消息，通常有如下4种做法：
+ 固定长度，例如每120个字节代表一个整包消息，不足的前面补位。解码器在处理这类定常消息的时候比较简单，每次读到指定长度的字节后再进行解码；FixedLengthFrameDecoder
+ 通过回车换行符区分消息，例如 HTTP 协议。这类区分消息的方式多用于文本协议；LineBasedFrameDecoder 
+ 通过特定的分隔符区分整包消息；DelimiterBasedFrameDecoder
+ 通过在协议头或消息头中设置长度字段来标识整包消息。LengthFieldBasedFrameDecoder

RocketMQ使用Netty中的自定义长度解码器LengthFieldBasedFrameDecoder解码器。在消息头中定义长度字段，标识消息的总长度，用以判断消息是否整包

### RocketMQ怎么保证不丢消息

![mq3](/img/in-post/2021/05/mq3.png)

+ **发送阶段**
	- 使用事务消息
+ **消息写入阶段**
	- 同步刷盘
	- 同步复制(双写)
+ **消费阶段**
	- offset上报机制，是上报treeMap第一个节点。防止未消费的offset被覆盖
	- 消费成功才上报

## 主从同步

#### 元数据复制

![meta-copy](/img/in-post/2021/05/meta-copy.png)

#### CommitLog复制

![cml-copy](/img/in-post/2021/05/cml-copy.png)

1. 首先启动Master并在指定端口监听
2. 客户端启动，主动连接Master，建立TCP连接
3. 客户端以每隔5s的间隔时间向服务端拉取消息，如果是第一次拉取的话，先获取本地commitlog文件中最大的偏移量，以该偏移量向服务端拉取消息
4. 服务端解析请求，并返回一批数据给客户端
5. 客户端收到一批消息后，将消息写入本地commitlog文件中，然后向Master汇报拉取进度，并更新下一次待拉取偏移量
6. 然后重复第3步

![cml-copy2](/img/in-post/2021/05/cml-copy2.png)

#### 主从同步阻塞服务

![ms-zs](/img/in-post/2021/05/ms-zs.png)


## 参数调优

+ flushCommitLogTimed
	- 默认值：False
	- 默认值不合理，异步刷盘这个参数应该设置成 true。默认1s刷一次，频繁刷盘，对性能影响极大
+ deleteWhen
	- 默认值：04
	- 几点删除过期文件的时间，删除文件时有很多磁盘读，这个默认值是合理的，有条件的话还是建议低峰删除
+ sendMessageThreadPoolNums
	- 默认值：1
	- 处理生产消息的线程数，这个线程干的事情很多，建议设置为 2～4，但太多也没有什么用。因为最终写 commit log 的时候只有一个线程能拿到锁
+ useReentrantLockWhenPutMessage
	- 默认值：False
	- 如果前一个参数设置比较大，这个最好设置为 true，避免高负载下自旋锁空转消耗 CPU
+ sendThreadPoolQueueCapacity
	- 默认值：10000
	- 处理生产消息的队列大小，默认值可能有点小，比如5万TPS(异步发送)的情况下，卡200ms就会爆。设置比较小的数字可能是担心有大量大消息撑爆内存(比如100K的话，1万个的消息大概占用1G内存，也还好)，具体可以自己算，如果都是小消息，可以把这个数字改大。可以修改Broker参数限制Client发送大消息
+ brokerFastFailureEnable
	- 默认值：True
	- Broker端快速失败（限流），和下面两个参数配合。这个机制可能有争议，client设置了超时时间，如果client还愿意等，并且sendThreadPoolQueue还没有满，不应该失败，sendThreadPoolQueue满了自然会拒绝新的请求。但如果Client设置的超时时间很短，没有这个机制可能导致消息重复。可以自行决定是否开启。理想情况下，能根据Client设置的超时时间来清理队列是最好的
+ waitTimeMillsInSendQueue
	- 默认值：200
	- 200ms很容易导致发送失败，建议改大，比如1000ms
+ osPageCacheBusyTimeOutMills
	- 默认值：1000
	- Page cache超时时间，如果内存比较多，比如32G以上，建议改大点

#### 百万消息堆积解决方式	

如果消费者依赖的服务宕机，导致消息无法成功消费，造成大量消息堆积：  
1、MessageQueue已经足够。如果MessageQueue有16个，消费者实例只有2台，则可以临时申请14台机器，启用16个消费者实例同时消费  
2、使用新的topic增加MessageQueue数量。如果MessageQueue数量比较少，则需要临时修改消费者系统代码，把消息写入一个新的Topic，这个topic有16个MessageQueue，然后再部署16个消费者实例同时消费


### 对比Kafka

+ 存储结构
+ 稀疏索引
+ 时间轮（对比Timer、ScheduledThreadPoolExecutor、rocketmq延时消息）

批处理打包发送机制：多条消息打包成一个batch。多个batch打包成一个request。减少网络通信，提高吞吐量

![kafka1](/img/in-post/2021/05/kafka1.jpg)

Kafka保证不丢消息：
+ `request.required.acks`的默认值即为1，代表消息被leader接收之后就返回成功。可以配置`acks = all/-1`，代表所有ISR副本都要接收到该消息之后才返回成功，需要配合`min.insync.replicas`设定ISR中的最小副本数
+ 生产者端设置：`producer.type=async/sync`，默认是sync。同步发送消息
+ Kafka consumer默认是自动提交位移，每5s提交一次。当consumer fetch了一些数据但还没有完全处理掉的时候，刚好到commit interval出发了提交offset操作，接着consumer crash掉了，会导致丢消息。关闭自动提交位移，在消息被完整处理之后再手动提交位移

![kafka2](/img/in-post/2021/05/kafka2.png)

+ Acceptor线程(Acceptor(mainReactor))：1个线程。新连接建连再分发
+ 网络线程(Processor(subReactor))：3个线程。监听其管理的连接，当事件到达之后，读取封装成Request，并将Request放入共享请求队列中
+ IO线程：8个线程。不断的从该队列中取出请求，执行真正的处理，处理完之后将响应发送到对应的Processor的响应队列中

### 对比

消息写入：
+ RocketMQ：所有topic存储在一个文件，顺序写
+ kafka：每个topic对应的每个patition一个文件，topic数量上升会导致顺序写降级为随机写

mmap：
+ rocketMQ：CommitLog、CosumerQueue都采用了mmap
+ kafka：.index文件mmap，.log文件写到堆外内存

发送消息:
+ RocketMQ：mmap+write的方式，并且通过预热来减少大文件mmap `>` 因为缺页中断产生的性能问题
+ Kafka：sendfile

事务消息:
+ RocketMQ：解决的问题是，确保执行本地事务和发消息这两个操作，要么都成功，要么都失败。并且增加了一个事务反查的机制，来尽量提高事务执行的成功率和数据一致性
+ Kafka：解决的问题是，确保在一个事务中发送的多条消息，要么都成功，要么都失败

消息重试：
+ RocketMQ：自动延迟重试
+ kafka：需要关闭自动提交offset。不支持延迟重试，也没有死信队列

消息过滤：
+ RocketMQ：Tag过滤
+ kafka：不支持消息按tag过滤

消息搜索：
+ RocketMQ：基于indexFile，在producer发消息时指定消息的key，之后可以根据key来搜索这条消息。原理其实就是个基于磁盘实现的hashMap
+ kafka：不支持消息搜索

什么时候Kafka不合适
+ 业务希望个别消费失败以后可以重试，并且不堵塞后续其它消息的消费
+ 业务希望消息可以延迟一段时间再投递
+ 业务需要发送的时候保证数据库操作和消息发送是一致的（也就是事务发送）
+ 为了排查问题，有的时候业务需要一定的单个消息查询能力
什么时候选择RocketMQ
+ 吞吐量高：单机吞吐量可达十万级
+ 可用性高：分布式架构
+ 消息可靠性高：经过参数优化配置，消息可以做到0丢失
+ 功能支持完善：MQ功能较为完善，还是分布式的，扩展性好
+ 支持10亿级别的消息堆积：不会因为堆积导致性能下降
+ 源码是java：方便我们查看源码了解它的每个环节的实现逻辑，并针对不同的业务场景进行扩展
+ 可靠性高：天生为金融互联网领域而生，对于要求很高的场景，尤其是电商里面的订单扣款，以及业务削峰，在大量交易涌入时，后端可能无法及时处理的情况
+ 稳定性高：RoketMQ在上可能更值得信赖，这些业务场景在阿里双11已经经历了多次考验

总结： RocketMQ提供了丰富的消息检索功能、事务消息、消息消费重试、定时消息等

