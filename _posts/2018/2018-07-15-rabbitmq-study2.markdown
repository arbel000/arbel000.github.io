---
layout:     post
title:      "RabbitMQ(二) 持久化与消息确认"
date:       2018-07-15
author:     "ZhouJ000"
header-img: "img/in-post/2018/post-bg-2018-headbg.jpg"
catalog: true
tags:
    - 高并发
    - mq
--- 

<font id="last-updated">最后更新于：2019-01-15</font>

[RabbitMQ(一) 基础概念](https://zhouj000.github.io/2019/01/15/rabbitmq-study1/)  
[RabbitMQ(二) 持久化与消息确认](https://zhouj000.github.io/2018/07/15/rabbitmq-study2/)  



RabbitMQ Java Client: 5.3.0  
spring-rabbit: 1.3.9.RELEASE  
spring-boot-starter-amqp: 1.5.3.RELEASE

# 持久化 

## exchange持久化

如果不设置exchange的持久化对消息的可靠性来说没有什么影响，但是如果exchange不设置持久化，那么当broker服务重启之后，exchange将不复存在，那么既而发送方rabbitmq producer就无法正常发送消息。所以一般会将exchange也持久化。

只需要将durable设为true即可。
```
/**
 * @param exchange the name of the exchange  > 交换机名称
 * @param type the exchange type  > 交换机类型
 * @param durable true if we are declaring a durable exchange (the exchange will survive a server restart)  
   > 持久化标志
 * @param autoDelete true if the server should delete the exchange when it is no longer in use
 * @param internal true if the exchange is internal, i.e. can't be directly
 * published to by a client.
 * @param arguments other properties (construction arguments) for the exchange
 * @return a declaration-confirm method to indicate the exchange was successfully declared
 * @throws java.io.IOException if an error is encountered
 */
Exchange.DeclareOk exchangeDeclare(String exchange,
	BuiltinExchangeType type,
	boolean durable,
	boolean autoDelete,
	boolean internal,
	Map<String, Object> arguments) throws IOException;
	
	
void exchangeDeclareNoWait(String exchange, BuiltinExchangeType type, boolean durable,
        boolean autoDelete, boolean internal, Map<String, Object> arguments) throws IOException;
Exchange.DeclareOk exchangeDeclarePassive(String name) throws IOException;
```


## queue持久化

queue的持久化是通过durable=true来实现的。
```java
Connection connection = new ConnectionFactory().newConnection();
Channel channel = connection.createChannel();
channel.queueDeclare("queue.persistent.name", true, false, false, null);
```
```
/**
 * Declare a queue
 * @see com.rabbitmq.client.AMQP.Queue.Declare
 * @see com.rabbitmq.client.AMQP.Queue.DeclareOk
 * @param queue the name of the queue  > 队列的名称
 * @param durable true if we are declaring a durable queue (the queue will survive a server restart)  >持久化标志
 * @param exclusive true if we are declaring an exclusive queue (restricted to this connection)
   > 排他队列，如果一个队列被声明为排他队列，该队列仅对首次申明它的连接可见，并在连接断开时自动删除。这里需要注意三点：
   1.排他队列是基于连接可见的，同一连接的不同信道是可以同时访问同一连接创建的排他队列；
   2.“首次”，如果一个连接已经声明了一个排他队列，其他连接是不允许建立同名的排他队列的，这个与普通队列不同；
   3.即使该队列是持久化的，一旦连接关闭或者客户端退出，该排他队列都会被自动删除的，这种队列适用于一个客户端发送读取消息的应用场景。
 * @param autoDelete true if we are declaring an autodelete queue (server will delete it when no longer in use)
   > 自动删除，如果该队列没有任何订阅的消费者的话，该队列会被自动删除。这种队列适用于临时队列。
 * @param arguments other properties (construction arguments) for the queue
 * @return a declaration-confirm method to indicate the queue was successfully declared
 * @throws java.io.IOException if an error is encountered
 */
Queue.DeclareOk queueDeclare(String queue, boolean durable, boolean exclusive, boolean autoDelete,
							 Map<String, Object> arguments) throws IOException;
```

queueDeclare相关的有4种方法，分别是：
```java
Queue.DeclareOk queueDeclare() throws IOException;
Queue.DeclareOk queueDeclare(String queue, boolean durable, boolean exclusive, boolean autoDelete,
                             Map<String, Object> arguments) throws IOException;
void queueDeclareNoWait(String queue, boolean durable, boolean exclusive, boolean autoDelete,
                        Map<String, Object> arguments) throws IOException;

// 可以用来检测一个queue是否已经存在。如果该队列存在，则会返回true；如果不存在，就会返回异常，但是不会创建新的队列
Queue.DeclareOk queueDeclarePassive(String queue) throws IOException;
```

## message持久化

如果要在重启后保持消息的持久化必须设置消息是持久化的标识。MessageProperties.PERSISTENT_TEXT_PLAIN：  
`channel.basicPublish("exchange.persistent", "persistent", MessageProperties.PERSISTENT_TEXT_PLAIN, "persistent_test_message".getBytes());`

```
/**
 * @param exchange the exchange to publish the message to  > 交换机名称
 * @param routingKey the routing key  > 路由键
 * @param mandatory true if the 'mandatory' flag is to be set
   > 当mandatory标志位设置为true时，如果exchange根据自身类型和消息routeKey无法找到一个符合条件的queue，
   那么会调用basic.return方法将消息返回给生产者(Basic.Return + Content-Header + Content-Body)；当mandatory设置为false时，出现上述情形broker会直接将消息扔掉。
 * @param immediate true if the 'immediate' flag is to be
 * set. Note that the RabbitMQ server does not support this flag.
   > 当immediate标志位设置为true时，如果exchange在将消息路由到queue(s)时发现对于的queue上么有消费者，
   那么这条消息不会放入队列中。当与消息routeKey关联的所有queue(一个或者多个)都没有消费者时，该消息会通过basic.return方法返还给生产者。
   !! 在RabbitMQ3.0以后的版本里，去掉了immediate参数的支持
 * @param props other properties for the message - routing headers etc  消息属性字段，比如消息头部信息等等
 * @param body the message body  消息主体部分
 * @throws java.io.IOException if an error is encountered
 */
void basicPublish(String exchange, String routingKey, boolean mandatory, boolean immediate, BasicProperties props, byte[] body)
		throws IOException;

body可用com.fasterxml.jackson.databind.ObjectMapper将String转为byte数组。
```

BasicProperties的定义: 
```
public BasicProperties(
		String contentType, // 消息类型如：text/plain
		String contentEncoding, // 编码
		Map<String,Object> headers,
		Integer deliveryMode, // 1:nonpersistent不持久化 2:persistent持久化
		Integer priority, // 优先级
		String correlationId,
		String replyTo, // 反馈队列
		String expiration, // expiration到期时间
		String messageId,
		Date timestamp,
		String type,
		String userId,
		String appId,
		String clusterId)
```
MessageProperties.PERSISTENT_TEXT_PLAIN的定义：就是将deliveryMode设置为了2
```
public static final BasicProperties PERSISTENT_TEXT_PLAIN =
        new BasicProperties("text/plain",
                            null,
                            null,
                            2,
                            0, null, null, null,
                            null, null, null, null,
                            null, null);
```
另一种配置方式：
```
AMQP.BasicProperties.Builder builder = new AMQP.BasicProperties.Builder();
builder.deliveryMode(2);
AMQP.BasicProperties properties = builder.build();
```

注意：  
**设置了队列和消息的持久化之后，当broker服务重启的之后，消息依旧存在。单只设置队列持久化，重启之后消息会丢失；单只设置消息的持久化，重启之后队列消失，既而消息也丢失。单单设置消息持久化而不设置队列的持久化显得毫无意义。**



# 消息确认

## producer端确认机制

当消息的发布者在将消息发送出去之后，消息到底有没有正确到达broker代理服务器呢？如果不进行特殊配置的话，默认情况下发布操作是不会返回任何信息给生产者的，也就是默认情况下我们的生产者是不知道消息有没有正确到达broker的，如果在消息到达broker之前已经丢失的话，持久化操作也解决不了这个问题，因为消息根本就没到达代理服务器。

RabbitMQ为我们提供了两种方式：
1. 通过AMQP事务机制实现，这也是AMQP协议层面提供的解决方案
2. 通过将channel设置成confirm模式来实现

### 事务

> 事务机制会带来大量的多余开销，并会导致吞吐量下降250%

RabbitMQ中与事务机制有关的方法有三个：txSelect(), txCommit()以及txRollback(), txSelect用于将当前channel设置成transaction模式，txCommit用于提交事务，txRollback用于回滚事务，在通过txSelect开启事务之后，我们便可以发布消息给broker代理服务器了，如果txCommit提交成功了，则消息一定到达了broker了，如果在txCommit执行之前broker异常崩溃或者由于其他原因抛出异常，这个时候我们便可以捕获异常通过txRollback回滚事务了。

```
try {
	channel.txSelect();
	...
	channel.basicPublish(exchange, routingKey, MessageProperties.PERSISTENT_TEXT_PLAIN, msg.getBytes());
	...
	channel.txCommit();
catch (Exception e) {
	...
	channel.txRollback();
}
```

事务确实能够解决producer与broker之间消息确认的问题，**只有消息成功被broker接受，事务提交才能成功**，否则我们便可以在捕获异常进行事务回滚操作同时进行消息重发，但是使用事务机制的话会降低RabbitMQ的性能，从AMQP协议的层面看是没有更好的方法。  
RabbitMQ为了补救事务带来的问题，提供了一个更好的方案,引入了confirmation机制(Publisher Confirm)，即将channel信道设置成confirm模式。


### Confirm机制

生产者将在channel设置成confirm模式，一旦在channel进入confirm模式，所有在该在channel上面发布的消息都会被指派一个唯一的ID(以confirm.select为基础从1 开始消息计数)，一旦消息被投递到所有匹配的队列之后，broker就会发送一个确认(basic.ack)给生产者(包含消息的唯一ID)，这就使得生产者知道消息已经正确到达目的队列了，如果消息和队列是可持久化的，那么确认消息会将消息写入磁盘之后发出，broker回传给生产者的确认消息中deliver-tag域包含了确认消息的序列号，此外broker也可以设置basic.ack的multiple域，表示到这个序列号之前的所有消息都已经被broker正确的处理。

confirm模式最大的好处在于他是异步的，一旦发布一条消息，生产者应用程序就可以在等信道返回确认的同时继续发送下一条消息，当消息最终得到确认之后，生产者应用便可以通过回调方法来处理该确认消息，如果RabbitMQ因为自身内部错误导致消息丢失或无法成功处理，此时broker将发送basic.nack来代替 basic.ack。在这个情形下，basic.nack中各域值的含义与basic.ack中相应各域含义是相同的，同时requeue域的值应该被忽略。通过nack 一或多条消息，broker表明自身无法对相应消息完成处理，并拒绝为这些消息的处理负责。在这种情况下，生产者应用同样可以在回调方法中处理该nack消息，比如将消息re-publish。

在channel被设置成confirm模式之后，所有被publish的后续消息都将被confirm(即ack)或者被nack一次。但是没有对消息被confirm 的快慢做任何保证，并且同一条消息不会既被confirm又被nack。

##### RabbitMQ实现

生产者通过调用channel的confirmSelect方法将channel设置为confirm模式(发送confirm.select 方法帧)，如果没有设置no-wait标志的话，broker会返回confirm.select-ok表示同意发送者将当前channel信道设置为confirm模式(从3.6版本来看，如果调用了channel.confirmSelect方法，默认情况下是直接将no-wait设置成false的，也就是默认情况下broker是必须回传confirm.select-ok的)。一旦在channel 上使用confirm.select方法，channel就将处于confirm模式。处于transactional模式的channel不能再被设置成confirm模式，反之亦然。

##### 代码实现

confirm机制提供了ConfirmListener和waitForConfirms两种方式。

归纳起来，客户端实现生产者confirm有三种编程方式：
1. 普通confirm模式：每发送一条消息后，调用waitForConfirms()方法，等待服务器端confirm。实际上是一种串行confirm了
2. 批量confirm模式：每发送一批消息后，调用waitForConfirms()方法，等待服务器端confirm
3. 异步confirm模式：提供一个回调方法，服务端confirm了一条或者多条消息后Client端会回调这个方法

```java
/**
普通confirm模式最简单，publish一条消息后，等待服务器端confirm,如果服务端返回false或者超时时间内未返回，客户端进行消息重传。
*/
channel.confirmSelect();
channel.basicPublish(exchangeName, routingKey, MessageProperties.PERSISTENT_TEXT_PLAIN, msg.getBytes());
if(!channel.waitForConfirms()){
    System.out.println("send message failed.");
}

/**
 批量confirm模式稍微复杂一点，客户端程序需要定期（每隔多少秒）或者定量（达到多少条）或者两则结合起来publish消息，然后等待服务器端confirm, 相比普通confirm模式，批量极大提升confirm效率，但是问题在于一旦出现confirm返回false或者超时的情况时，客户端需要将这一批次的消息全部重发，这会带来明显的重复消息数量，并且，当消息经常丢失时，批量confirm性能应该是不升反降的。
*/
try {
	channel.confirmSelect();
	for(int i=0; i < batchCount; i++){
		channel.basicPublish(exchangeName, routingKey, MessageProperties.PERSISTENT_TEXT_PLAIN, msg.getBytes());
	}
	// 该方法会等到最后一条消息得到确认或者得到nack才会结束,会造成当前程序的阻塞
	channel.waitForConfirmsOrDie();
	/*if(!channel.waitForConfirms()){
		System.out.println("send message failed.");
	}*/
catch (Exception e) {
	...
}

/**
异步confirm模式的编程实现最复杂，Channel对象提供的ConfirmListener()回调方法只包含deliveryTag（当前Chanel发出的消息序号），我们需要自己为每一个Channel维护一个unconfirm的消息序号集合，每publish一条数据，集合中元素加1，每回调一次handleAck方法，unconfirm集合删掉相应的一条（multiple=false）或多条（multiple=true）记录。从程序运行效率上看，这个unconfirm集合最好采用有序集合SortedSet存储结构。实际上，SDK中的waitForConfirms()方法也是通过SortedSet维护消息序号的。
*/
SortedSet<Long> confirmSet = Collections.synchronizedSortedSet(new TreeSet<Long>());
channel.confirmSelect();
channel.addConfirmListener(new ConfirmListener() {
	public void handleAck(long deliveryTag, boolean multiple) throws IOException {
		if (multiple) {
			confirmSet.headSet(deliveryTag + 1).clear();
		} else {
			confirmSet.remove(deliveryTag);
		}
	}
	public void handleNack(long deliveryTag, boolean multiple) throws IOException {
		System.out.println("Nack, SeqNo: " + deliveryTag + ", multiple: " + multiple);
		if (multiple) {
			confirmSet.headSet(deliveryTag + 1).clear();
		} else {
			confirmSet.remove(deliveryTag);
		}
	}
});
while(condition) {
	long nextSeqNo = channel.getNextPublishSeqNo();
	channel.basicPublish(exchangeName, routingKey, MessageProperties.PERSISTENT_TEXT_PLAIN, msg.getBytes());
	confirmSet.add(nextSeqNo);
}
```
发送平均速率比较：  
事务模式（tx）：1637.484  
普通confirm模式(common)：1936.032  
批量confirm模式(batch)：10432.45  
异步confirm模式(async)：10542.06  

可以看到事务模式性能是最差的，普通confirm模式性能比事务模式稍微好点，但是和批量confirm模式还有异步confirm模式相比，还是小巫见大巫。批量confirm模式的问题在于confirm之后返回false之后进行重发这样会使性能降低，异步confirm模式(async)编程模型较为复杂，至于采用哪种方式，那是仁者见仁智者见智了。


##### RabbitTemplate
```
// producer的手工确认模式
CachingConnectionFactory connectionFactory = new CachingConnectionFactory(ip, port);
connectionFactory.setUsername(userName);
connectionFactory.setPassword(password);
connectionFactory.setPublisherConfirms(true); // enable confirm mode
...

@Bean
/** 因为要设置回调类，所以应是prototype类型，如果是singleton类型，则回调类为最后一次设置 */
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public RabbitTemplate rabbitTemplate() {
	RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory());
	return template;
}


rabbitTemplate.setMandatory(true);
rabbitTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {
	@Override
	public void confirm(CorrelationData correlationData, boolean ack, String cause) {
		if (!ack) {
			throw new RuntimeException("send error " + cause);
		}
	}
});
// rabbitTemplate.setReturnCallback()
```

rabbitTemplate的发送流程是这样的：
1. 发送数据并返回(不确认rabbitmq服务器已成功接收)
2. 异步的接收从rabbitmq返回的ack确认信息
3. 收到ack后调用confirmCallback函数  
注意：**在confirmCallback中是没有原message的，所以无法在这个函数中调用重发，confirmCallback只有一个通知的作用**

如果在2，3步中任何时候切断连接，我们都无法确认数据是否真的已经成功发送出去，从而造成数据丢失的问题。

解决：  
在rabbitTemplate异步确认的基础上
1. 在本地缓存已发送的message
2. 通过confirmCallback或者被确认的ack，将被确认的message从本地删除
3. 定时扫描本地的message，如果大于一定时间未被确认，则重发  
注意：如果rabbitmq接收到了消息，在发送ack确认时，网络断了，造成客户端没有收到ack，就会重发消息。(相比于丢失消息，重发消息要好解决的多，我们可以在consumer端做到幂等)



### basicPublish的mandatory和immediate

概括来说，mandatory标志告诉服务器至少将该消息route到一个队列中，否则将消息返还给生产者；immediate标志告诉服务器如果该消息关联的queue上有消费者，则马上将消息投递给它，如果所有queue都没有消费者，直接把消息返还给生产者，不用将消息入队列等待消费者了。

获取到没有被正确路由到合适队列的消息:
```
可以通过为channel信道设置ReturnListener监听器来实现

channel.addReturnListener(new ReturnListener() {
	@Override
	public void handleReturn(int replyCode, String replyText, String exchange, String routingKey, AMQP.BasicProperties basicProperties, byte[] body) throws IOException {
		String message = new String(body);
		System.out.println("Basic.return返回的结果是: " + message);
	}
});
```

在RabbitMQ3.0以后的版本里，去掉了immediate参数的支持，发送带immediate标记的publish会返回如下错误：  
“{amqp_error,not_implemented,”immediate=true”,’basic.publish’}”  

版本变化描述:
```
Removal of “immediate” flag 
What changed? We removed support for the rarely-used “immediate” flag on AMQP’s basic.publish. 
Why on earth did you do that? Support for “immediate” made many parts of the codebase more complex, particularly around mirrored queues. It also stood in the way of our being able to deliver substantial performance improvements in mirrored queues. 
What do I need to do? If you just want to be able to publish messages that will be dropped if they are not consumed immediately, you can publish to a queue with a TTL of 0. 
If you also need your publisher to be able to determine that this has happened, you can also use the DLX feature to route such messages to another queue, from which the publisher can consume them. 

大意: immediate标记会影响镜像队列性能，增加代码复杂性，并建议采用“TTL”和“DLX”等方式替代。
```

### 消息在什么时候确认?

从实验来看，消息的确认机制只是确认publisher发送消息到broker，由broker进行应答，不能确认消息是否有效消费。  
而为了确认消息是否被发送给queue，应该在发送消息中启用参数mandatory=true，使用ReturnListener接收未被发送成功的消息。  
接下来就需要确认消息是否被有效消费。publisher端目前并没有提供监听事件，但提供了应答机制来保证消息被成功消费，应答方式： 
+ basicAck：成功消费，消息从队列中删除 
+ basicNack：requeue=true，消息重新进入队列，false被删除 
+ basicReject：等同于basicNack 
+ basicRecover：消息重入队列，requeue=true，发送给新的consumer，false发送给相同的consumer 

> broker将在下面的情况中对消息进行confirm:  
broker发现当前消息无法被路由到指定的queues中(如果设置了mandatory属性，则broker会先发送basic.return)  
非持久属性的消息到达了其所应该到达的所有queue中(和镜像queue中)  
持久消息到达了其所应该到达的所有queue中(和镜像queue中)，并被持久化到了磁盘(被fsync)  
持久消息从其所在的所有queue中被consume了(如果必要则会被acknowledge)

> basicRecover：是路由不成功的消息可以使用recovery重新发送到队列中。  
basicReject：是接收端告诉服务器这个消息我拒绝接收,不处理,可以设置是否放回到队列中还是丢掉，而且只能一次拒绝一个消息,官网中有明确说明不能批量拒绝消息，为解决批量拒绝消息才有了basicNack。  
basicNack：可以一次拒绝N条消息，客户端可以设置basicNack方法的multiple参数为true，服务器会拒绝指定了delivery_tag的所有未确认的消息(tag是一个64位的long值，最大值是9223372036854775807)。


## broker端镜像队列

关键的问题是消息在正确存入RabbitMQ之后，还需要有一段时间（这个时间很短，但不可忽视）才能存入磁盘之中，RabbitMQ并不是为每条消息都做fsync的处理，可能仅仅保存到cache中而不是物理磁盘上，在这段时间内RabbitMQ broker发生crash, 消息保存到cache但是还没来得及落盘，那么这些消息将会丢失。  

那么这个怎么解决呢？  
一种方法可以引入RabbitMQ的mirrored-queue即镜像队列，这个相当于配置了副本，当master在此特殊时间内crash掉，可以自动切换到slave，这样有效的保障了HA, 除非整个集群都挂掉，这样也不能完全的100%保障RabbitMQ不丢消息，但比没有mirrored-queue的要好很多，很多现实生产环境下都是配置了mirrored-queue的。  
另一种可能的方案是在系统panic时或者异常重启时或者断电时，应该给各个应用留出时间去flash cache，保证每个应用都能exit gracefully。

##### 方法

如果RabbitMQ集群是由多个broker节点构成的，那么从服务的整体可用性上来讲，该集群对于单点失效是有弹性的，但是同时也需要注意：尽管exchange和binding能够在单点失效问题上幸免于难，但是queue和其上持有的message却不行，这是因为queue及其内容仅仅存储于单个节点之上，所以一个节点的失效表现为其对应的queue不可用。

引入RabbitMQ的镜像队列机制，将queue镜像到cluster中其他的节点之上。在该实现下，如果集群中的一个节点失效了，queue能自动地切换到镜像中的另一个节点以保证服务的可用性。在通常的用法中，针对每一个镜像队列都包含一个master和多个slave，分别对应于不同的节点。slave会准确地按照master执行命令的顺序进行命令执行，故slave与master上维护的状态应该是相同的。除了publish外所有动作都只会向master发送，然后由master将命令执行的结果广播给slave们，故看似从镜像队列中的消费操作实际上是在master上执行的。

一旦完成了选中的slave被提升为master的动作，发送到镜像队列的message将不会再丢失：publish到镜像队列的所有消息总是被直接publish到master和所有的slave之上。这样一旦master失效了，message仍然可以继续发送到其他slave上。

> RabbitMQ的镜像队列同时支持publisher confirm和事务两种机制。在事务机制中，只有当前事务在全部镜像queue中执行之后，客户端才会收到Tx.CommitOk的消息。同样的，在publisher confirm机制中，向publisher进行当前message确认的前提是该message被全部镜像所接受了。

##### 设置

镜像队列的配置通过添加policy完成，policy添加的命令为:
```
rabbitmqctl set_policy [-p Vhost] Name Pattern Definition [Priority]

-p Vhost： 可选参数，针对指定vhost下的queue进行设置
Name: policy的名称
Pattern: queue的匹配模式(正则表达式)
Definition：镜像定义，包括三个部分ha-mode, ha-params, ha-sync-mode
    ha-mode:指明镜像队列的模式，有效值为 all/exactly/nodes
        all：表示在集群中所有的节点上进行镜像
        exactly：表示在指定个数的节点上进行镜像，节点的个数由ha-params指定
        nodes：表示在指定的节点上进行镜像，节点名称通过ha-params指定
    ha-params：ha-mode模式需要用到的参数
    ha-sync-mode：进行队列中消息的同步方式，有效值为automatic和manual
priority：可选参数，policy的优先级

例如，对队列名称以“queue_”开头的所有队列进行镜像，并在集群的两个节点上完成进行，policy的设置命令为：
rabbitmqctl set_policy --priority 0 --apply-to queues mirror_queue "^queue_" '{"ha-mode":"exactly","ha-params":2,"ha-sync-mode":"automatic"}'
```

[RabbitMQ之镜像队列](https://blog.csdn.net/u013256816/article/details/71097186)

### 消息什么时候需要持久化?

1. 消息本身在publish的时候就要求消息写入磁盘
2. 内存紧张，需要将部分内存中的消息转移到磁盘

#### 消息什么时候刷到磁盘?

1. 写入文件前会有一个Buffer,大小为1M,数据在写入文件时，首先会写入到这个Buffer，如果Buffer已满，则会将Buffer写入到文件(未必刷到磁盘)。  
2. 有个固定的刷盘时间：25ms,也就是不管Buffer满不满，每个25ms，Buffer里的数据及未刷新到磁盘的文件内容必定会刷到磁盘。   
3. 每次消息写入后，如果没有后续写入请求，则会直接将已写入的消息刷到磁盘：使用Erlang的receive x after 0实现，只要进程的信箱里没有消息，则产生一个timeout消息，而timeout会触发刷盘操作。

#### 文件何时删除？

消息保存于/msg_store_persistent/x.rdq文件中，其中x为数字编号，从1开始，每个文件最大为16M，超过这个大小会生成新的文件，文件编号加1。

1. 当所有文件中的垃圾消息(已经被删除的消息)比例大于阈值(GARBAGE_FRACTION = 0.5)时，会触发文件合并操作(至少有三个文件存在的情况下)，以提高磁盘利用率。
2. publish消息时写入内容，ack消息时删除内容(更新该文件的有用数据大小)，当一个文件的有用数据等于0时，删除该文件。

#### 消息索引

每个队列会对队列中的消息维护一个索引，每入队列一个消息，索引加1，索引在持久化时，以2^14个(16384)entry为单位组成一个文件(Segment)。  
rabbit_msg_index模块为每一个Segment维护一个unacked计数，每publish一个消息加1，每ack一个消息减1，当unacked=0时，文件删除。



## consumer端确认机制

从consumer端来说，如果这时autoAck=true，那么当consumer接收到相关消息之后，还没来得及处理或在处理中就crash掉了，因为我们采用no-ack的方式进行确认，也就是说，每次Consumer接到数据后，而不管是否处理完成，RabbitMQ Server会立即把这个Message标记为完成，然后从queue中删除了。那么这样也算数据丢失。这种情况也好处理，RabbitMQ支持消息确认机制，即acknowledgments。只需将autoAck设置为false，然后在正确处理完消息之后进行手动ack(channel.basicAck)。如果Consumer退出了但是没有发送ack，那么RabbitMQ就会把这个Message发送到下一个Consumer。这样就保证了在Consumer异常退出的情况下数据也不会丢失。这里并没有用到超时机制。RabbitMQ仅仅通过Consumer的连接中断来确认该Message并没有被正确处理。也就是说，RabbitMQ给了Consumer足够长的时间来做数据处理。

> 如果一个Consumer异常退出了，它处理的数据能够被另外的Consumer处理，这样数据在这种情况下就不会丢失了

```
String basicConsume(String queue, boolean autoAck, Consumer callback) throws IOException;

QueueingConsumer consumer = new QueueingConsumer(channel);
channel.basicConsume(queueName, false, consumer);
while(true){
	QueueingConsumer.Delivery delivery = consumer.nextDelivery();
	String msg = new String(delivery.getBody());
	// do something with msg. 
	channel.basicAck(delivery.getEnvelope().getDeliveryTag(), false);
	
	// requeue，true重新进入队列  
    //channel.basicNack(envelope.getDeliveryTag(), false, true); 
	// requeue，true重新进入队列,与basicNack差异缺少multiple参数  
    //channel.basicReject(envelope.getDeliveryTag(), true);
	// 重新发送到队列中
	//channel.basicRecover();
}
```

> 如果改为手动而忘记了ack，那么后果很严重。当Consumer退出时，Message会重新分发。然后RabbitMQ会占用越来越多的内存，由于RabbitMQ会长时间运行，因此这个“内存泄漏”是致命的。

```
使用spring-rabbit时，用Listener(实现接口ChannelAwareMessageListener)接受消息设置AcknowledgeMode

@Bean
public SimpleMessageListenerContainer testListenerContainer() {
	SimpleMessageListenerContainer container = new SimpleMessageListenerContainer();
	container.setConnectionFactory(configBase.connectionFactory());
	container.setQueues(this.testQueue());
	container.setExposeListenerChannel(true);
	container.setMaxConcurrentConsumers(1);
	container.setConcurrentConsumers(1);
	container.setMessageListener(myTestMQListener);
	container.setAcknowledgeMode(AcknowledgeMode.MANUAL);
	return container;
}

<rabbit:listener-container connection-factory="connectionFactory" acknowledge="manual">
    <rabbit:listener queues="queue_xxx" ref="MqConsumer"/>
    <rabbit:listener queues="queue_xxx" ref="MqConsumer2"/>
</rabbit:listener-container>
```

```
spring boot 实现：

@RabbitHandler
@RabbitListener(queues = {"${rabbitmq.bill_queueName}"})
public void process(@Payload Message msg, @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag, Channel channel) throws Exception {
	...
}

@RabbitListener(containerFactory = "rabbitListenerContainerFactory", bindings = @QueueBinding(
        value = @Queue(value = "${mq.config.queue}", durable = "true"),
        exchange = @Exchange(value = "${mq.config.exchange}", type = ExchangeTypes.TOPIC),
        key = "${mq.config.key}"), admin = "rabbitAdmin")
```


本文参考：  
[消息中间件（Kafka/RabbitMQ）收录集](https://blog.csdn.net/u013256816/article/details/54743481)  
推荐阅读：  
[RabbitMQ消息可靠性分析](https://blog.csdn.net/u013256816/article/details/79147591)  