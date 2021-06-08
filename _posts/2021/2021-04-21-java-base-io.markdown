---
layout:     post
title:      "Java基础SE(四) IO"
date:       2021-04-21
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


# IO

Java的IO包主要关注的是从原始数据源的读取以及输出原始数据到目标媒介。典型的数据源和目标媒介：
+ 文件
+ 管道
+ 网络连接
+ 内存缓存
+ Java标准输入、输出、错误输出
	- System.in, System.out, System.error

一些IO类：
+ 字节流
	- InputStream
		+ FileInputStream
		+ FilterInputStream
			- BufferedInputStream
			- DataInputStream
			- PushbackInputStream
		+ ObjectInputStream
		+ PipedInputStream
		+ StringBufferInputStream
		+ ByteArrayInputStream
	- OutputStream
		+ FileOutputStream
		+ FilterOutputStream
			- BufferedOutputStream
			- DataOutputStream
			- PrintStream
		+ ObjectOutputStream
		+ PipedOutputStream
		+ ByteArrayOutputStream
+ 字符流
	- Reader
		+ BufferedReader
		+ InputStreamReader
			- FileReader
		+ StringReader
		+ PipedReader
		+ FilterReader
			- PushbackReader
	- Writer
		+ BufferedWriter
		+ OutputStreamWriter
			- FileWriter
		+ StringWriter
		+ PipedWriter
		+ CharArrayWriter
		+ FilterWriter

![io-system](/img/in-post/2021/04/io-system.png)

Reader是Java IO中所有Reader的基类。Reader与InputStream类似，不同点在于，Reader基于**字符**而非基于字节。所以Reader用于读取文本，而InputStream用于读取**原始字节**。Writer同理

> Java内部使用UTF8编码表示字符串。输入流中一个字节可能并不等同于一个UTF8字符。如果你从输入流中以字节为单位读取UTF8编码的文本，并且尝试将读取到的字节转换成字符，可能会得不到预期的结果

## 常用操作

```java
byte[] content = new byte[1024];
int length;
while ((length = inputStream.read(content)) != -1) {
	outputStream.write(content, 0, length);
}
outputStream.flush();
outputStream.close();
```

```java
Reader reader = new InputStreamReader(new FileInputStream("xxx.txt"), "UTF-8");
int data = reader.read();
while(data != -1){
    // 这里不会造成数据丢失，因为返回的int类型变量data只有低16位有数据，高16位没有数据
    char theChar = (char) data;
    data = reader.read();
}
reader.close();

Writer writer = new OutputStreamWriter(new FileOutputStream("xxx.txt"));
writer.write("Hello World");
writer.close();
```




# NIO

标准的IO基于字节流和字符流进行操作的，而NIO是基于**通道**(Channel)和**缓冲区**(Buffer)进行操作，数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中

当线程从通道读取数据到缓冲区时，线程还是可以进行其他事情。当数据被写入到缓冲区时，线程可以继续处理它。从缓冲区写入通道也类似。Java NIO可以让你非阻塞的使用IO

Java NIO引入了**选择器**的概念，选择器用于监听多个通道的事件(比如：连接打开，数据到达)。因此单个的线程可以监听多个数据通道

与IO的区别：
+ IO是面向流的，NIO是**面向缓冲区**的
+ IO流是阻塞的，NIO是**不阻塞**的
+ NIO选择器允许一个单独线程来监视多个输入通道


## Channels与Buffers

![buffer1](/img/in-post/2021/04/buffer1.png)

数据可以从Channel读到Buffer中，也可以从Buffer写到Channel中，即通道中的数据总是要先读到一个Buffer，或者总是要从一个Buffer中写入。通道可以异步地读写

Channel的一些实现：  
1、FileChannel：从文件中读写数据  
2、DatagramChannel：能通过UDP读写网络中的数据  
3、SocketChannel：能通过TCP读写网络中的数据  
4、ServerSocketChannel：可以监听新进来的TCP连接，对每一个新进来的连接都会创建一个SocketChannel

```java
RandomAccessFile aFile = new RandomAccessFile("xxx.txt", "rw");
FileChannel inChannel = aFile.getChannel();

// 0.分配48字节capacity的ByteBuffer
ByteBuffer buf = ByteBuffer.allocate(48);

// 1.写入数据到Buffer
int bytesRead = inChannel.read(buf);
while (bytesRead != -1) {
    System.out.println("Read " + bytesRead);
	// 2.调用flip()方法，从写模式切换到读模式
    buf.flip();
	// 3.从Buffer中读取数据
    while(buf.hasRemaining()){
        System.out.print((char) buf.get());
    }
	// 4.调用clear()方法或者compact()方法，让它可以再次被写入
    buf.clear();
    bytesRead = inChannel.read(buf);
}
aFile.close();
```

Buffer的一些实现：  
1、ByteBuffer  
2、CharBuffer  
3、DoubleBuffer  
4、FloatBuffer  
5、IntBuffer  
6、LongBuffer  
7、ShortBuffer  
8、MappedByteBuffer

缓冲区本质上是一块可以写入数据，然后可以从中读取数据的内存。这块内存被包装成NIO Buffer对象，并提供了一组方法，用来方便的访问该块内存

![buffers-modes](/img/in-post/2021/04/buffers-modes.png)

capacity和limit的含义取决于Buffer处在读模式还是写模式。不管Buffer处在什么模式，capacity的含义总是一样的

+ capacity
	- Buffer有一个**固定**的大小值，叫"capacity"，只能往里写capacity个类型数据
	- 一旦Buffer满了，需要将其清空(通过读数据或者清除数据)，才能继续写数据往里写数据
+ position
	- 写模式下，position表示**当前**的位置。初始的position值为0，当写入数据后，position会向前移动到下一个可插入数据的Buffer单元。position最大可为capacity – 1
	- 当将Buffer从写模式切换到读模式，position会被**重置**为0. 当从Buffer的position处读取数据时，position向前移动到下一个可读的位置
+ limit
	- 写模式下，limit表示你最多能往Buffer里写多少数据，即**等于capacity**
	- 当将Buffer从写模式切换到读模式，limit表示你最多能读到多少数据，即limit会被设置成**写模式下的position值**

```java
写入数据到Buffer:
int bytesRead = inChannel.read(buf);	// 从Channel读
or
buf.put(127);	// put有很多版本

从Buffer读取数据：
int bytesWritten = inChannel.write(buf);  // 写入channel
or
byte aByte = buf.get();		// get有很多版本
```

`rewind()`方法可以将position设回0，这样可以**重读**Buffer中的所有数据，而limit保持不变

`clear()`方法可以将position设回0，limit被设置成capacity的值。Buffer中的数据并未清除，只是这些标记告诉我们可以从哪里开始往Buffer里写数据，未读的数据将**被遗忘**

`compact()`方法将所有未读的数据拷贝到Buffer起始处，position设到最后一个未读元素正后面，limit属性依然为capacity。这样Buffer准备好写数据，且**不会覆盖**未读的数据

`mark()`方法可以**标记**Buffer中的一个特定position，之后可以通过调用`reset()`方法恢复到这个position

#### Scatter/Gather

分散(scatter)从Channel中读取是指在读操作时将读取的数据写入多个buffer中。聚集(gather)写入Channel是指在写操作时将多个buffer的数据写入同一个Channel。scatter/gather经常用于**需要将传输的数据分开处理的场合**，例如传输一个由消息头和消息体组成的消息，你可能会将消息体和消息头分散到不同的buffer中，这样你可以方便的处理消息头和消息体

```java
ByteBuffer header = ByteBuffer.allocate(128);
ByteBuffer body   = ByteBuffer.allocate(1024);
ByteBuffer[] bufferArray = { header, body };

// 按照buffer在数组中的顺序将从channel中读取的数据写入到buffer，当一个buffer被写满后，channel紧接着向另一个buffer中写
channel.read(bufferArray);

// write data into buffers
// 按照数组顺序写入，注意只有position和limit之间的数据才会被写入，因此与Scatter相反，Gathering能较好的处理动态消息
channel.write(bufferArray);
```

#### 通道间传输

如果两个通道中有一个是FileChannel，那你可以直接将数据从一个channel传输到另外一个channel

```java
RandomAccessFile fromFile = new RandomAccessFile("xxx.txt", "rw");
FileChannel fromChannel = fromFile.getChannel();

RandomAccessFile toFile = new RandomAccessFile("xxx.txt", "rw");
FileChannel toChannel = toFile.getChannel();

long position = 0;
long count = fromChannel.size();

// 将数据从源通道传输到FileChannel中
toChannel.transferFrom(position, count, fromChannel);
or
// 将数据从FileChannel传输到其他的channel中
fromChannel.transferTo(position, count, toChannel);
```

### FileChannel

FileChannel是一个连接到文件的通道，可以通过文件通道读写文件。FileChannel无法设置为非阻塞模式，总是运行在阻塞模式下

```java
RandomAccessFile aFile = new RandomAccessFile("xxx.txt", "rw");
FileChannel inChannel = aFile.getChannel();

// 读取
ByteBuffer buf = ByteBuffer.allocate(48);
// 有时可能需要在FileChannel的某个特定位置进行数据的读/写操作
// long pos = channel.position();
// channel.position(pos +123);
// 表示了有多少字节被读到了Buffer中，-1表示到了文件末尾
int bytesRead = inChannel.read(buf);

// 写入
buf.clear();
buf.put("xxxxxx".getBytes());
buf.flip();
// 无法保证write()方法一次能向FileChannel写入多少字节，需要循环调用
while(buf.hasRemaining()) {
	channel.write(buf);
}

// 最后关闭
channel.close();
```

`FileChannel.truncate(1024);`可以截取一个文件到指定长度，后面部分将被删除，此例为截取文件的前1024个字节

`FileChannel.force()`方法将通道里尚未写入磁盘的数据强制写到磁盘上。出于性能考虑，操作系统会将数据缓存在内存中，所以无法保证写入到FileChannel里的数据一定会即时写到磁盘上。`force()`方法有一个boolean类型的参数，指明是否同时将文件元数据(权限信息等)写到磁盘上

### SocketChannel

SocketChannel是一个连接到TCP网络套接字的通道。可以通过打开一个SocketChannel并连接到服务器；或一个新连接到达ServerSocketChannel时创建一个，这两种方法来创建SocketChannel

```java
SocketChannel socketChannel = SocketChannel.open();
socketChannel.connect(new InetSocketAddress("http://xxx.com", 80));

// 阻塞模式的 读取与写入 与FileChannel相同
socketChannel.close();
```

SocketChannel可以设为非阻塞(非阻塞模式与选择器搭配会工作的更好)，在异步模式下调用：
```java
SocketChannel socketChannel = SocketChannel.open();
socketChannel.configureBlocking(false);
socketChannel.connect(new InetSocketAddress("http://xxx.com", 80));

// 在尚未写出任何内容时可能就返回了，因此在循环中做事
while(! socketChannel.finishConnect() ){
    //wait, or do something else...
}
```

### DatagramChannel

DatagramChannel是一个能收发UDP包的通道。因为UDP是无连接的网络协议，所以不能像其它通道那样读取和写入。它发送和接收的是数据包

```java
DatagramChannel channel = DatagramChannel.open();
channel.socket().bind(new InetSocketAddress(9999));

// 接受数据
ByteBuffer buf = ByteBuffer.allocate(48);
buf.clear();
// 将接收到的数据包内容复制到指定的Buffer. 如果Buffer容不下收到的数据，多出的数据将被丢弃
channel.receive(buf);

// 发送数据
ByteBuffer buf = ByteBuffer.allocate(48);
buf.clear();
buf.put("xxxx".getBytes());
buf.flip();
// 发送xxxx到xxx.com的UDP端口9999，UDP在数据传送方面没有任何保证
int bytesSent = channel.send(buf, new InetSocketAddress("xxx.com", 9999));
```

由于UDP是无连接的，连接到特定地址并不会像TCP通道那样创建一个真正的连接。而是锁住DatagramChannel，让其只能从特定地址收发数据
```java
channel.connect(new InetSocketAddress("xxx.com", 80));

// 和用传统的通道一样。只是在数据传送方面没有任何保证
int bytesRead = channel.read(buf);
int bytesWritten = channel.write(but);
```


## Selectors

![overview-selectors](/img/in-post/2021/04/overview-selectors.png)

向Selector注册Channel，然后调用它的select()方法。这个方法会一直阻塞到某个注册的通道有事件就绪。一旦这个方法返回，线程就可以处理这些事件

> 现代的操作系统和CPU在多任务方面表现的越来越好，所以多线程的开销随着时间的推移，变得越来越小了

```java
Selector selector = Selector.open();

// 必须处于非阻塞模式，*FileChannel不能切换到非阻塞模式
channel.configureBlocking(false);
// 对多个事件感兴趣：int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;
SelectionKey key = channel.register(selector, Selectionkey.OP_READ);
```

可以监听四种不同类型的事件：
+ Connect：SelectionKey.OP_CONNECT
+ Accept：SelectionKey.OP_ACCEPT
+ Read：SelectionKey.OP_READ
+ Write：SelectionKey.OP_WRITE

返回的SelectionKey对象包含了一些属性：
+ interest集合
+ ready集合
+ Channel
+ Selector
+ 附加的对象(可选)
	- `selectionKey.attach(theObject);`
	- `SelectionKey key = channel.register(selector, SelectionKey.OP_READ, theObject);`

注册完成后，就可以选择select返回**感兴趣的事件**了：
+ `int select()`
	- 阻塞到至少有一个通道在你注册的事件上就绪了，返回的int值表示有多少通道已经就绪
+ `int select(long timeout)`
	- 多设置了超时时间
+ `int selectNow()`
	- 不会阻塞立即返回，没有通道可选将直接返回零

当确定有通道就绪后，就可以访问"已选择键集"中的就绪通道了，最后遍历处理：
```java
while(true) {
	int readyChannels = selector.select();
	if(readyChannels == 0) continue;

	Set selectedKeys = selector.selectedKeys();
	Iterator keyIterator = selectedKeys.iterator();
	while(keyIterator.hasNext()) {
		SelectionKey key = keyIterator.next();
		if(key.isAcceptable()) {
			// a connection was accepted by a ServerSocketChannel.
		} else if (key.isConnectable()) {
			// a connection was established with a remote server.
		} else if (key.isReadable()) {
			// a channel is ready for reading
		} else if (key.isWritable()) {
			// a channel is ready for writing
		}
		// Selector不会自己从已选择键集中移除SelectionKey实例，必须在处理完后自己移除
		keyIterator.remove();
	}
}
```

`SelectionKey.channel()`返回的通道需要转型成你要处理的类型，比如ServerSocketChannel或SocketChannel等

`Selector.wakeup()`方法可以让阻塞在`select()`方法上的线程会立马返回，如果当前没有阻塞，则下一个调用`select()`方法的线程会立即醒来

`close()`方法会关闭该Selector，且使注册到该Selector上的所有SelectionKey实例无效。不过通道本身并不会关闭

## 非阻塞服务

一个非阻塞式服务器需要时不时检查输入的消息来判断是否有任何新的完整的消息发送过来。服务器可能会在一个或多个完整消息发来之前就检查了多次。同样，一个非阻塞式服务器需要时不时检查是否有任何数据需要写入。服务器需要检查是否有任何相应的连接准备好将该数据写入它们。只在第一次排队消息时检查是不够的，因为消息可能被部分写入

所有这些非阻塞服务器最终都需要定期执行的三个"管道"，在循环中重复执行：
+ 读取管道(The read pipeline)，用于检查是否有新数据从开放连接进来的
+ 处理管道(The process pipeline)，用于所有任何完整消息
+ 写入管道(The write pipeline)，用于检查是否可以将任何传出的消息写入任何打开的连接

![non-blocking-server-9](/img/in-post/2021/04/non-blocking-server-9.png)
![non-blocking-server-10](/img/in-post/2021/04/non-blocking-server-10.png)

## pipe

管道是2个线程之间的单向数据连接，Pipe有一个source通道和一个sink通道，从source通道读取，被写到sink通道

![pipe](/img/in-post/2021/04/pipe.jpg)

```java
Pipe pipe = Pipe.open();

// // 向管道写数据
Pipe.SinkChannel sinkChannel = pipe.sink();
ByteBuffer buf = ByteBuffer.allocate(48);
buf.clear();
buf.put("hello world".getBytes());
buf.flip();
while(buf.hasRemaining()) {
    sinkChannel.write(buf);
}

// 从管道读取数据
Pipe.SourceChannel sourceChannel = pipe.source();
ByteBuffer buf = ByteBuffer.allocate(48);
int bytesRead = sourceChannel.read(buf);
```

## Path

一个Path实例代表文件系统的路径，路径可以指向文件或目录。路径可以是绝对路径，也可以是相对路径。`java.nio.file.Path`与`java.io.File`类在很多方面类似

```java
// Windows 绝对路径
Path path = Paths.get("c:\\xxx\\xxx.txt");
// Unix 绝对路径，在Windows中/开头会被解释为相对于当前驱动器
Path path = Paths.get("/home/xxx/xxx.txt");

// 相对路径
Path file = Paths.get("d:\\xxx", "xx\\xx\\xxx.txt");
```

relativize创建一个新路径：
```java
Path basePath = Paths.get("/a");
Path path = Paths.get("/a/b/c/file.txt");

// b/c/file.txt
Path basePathToPath = basePath.relativize(path);
// ../../..
Path pathToBasePath = path.relativize(basePath);
```



# 操作系统IO

用户态-系统态；阻塞-非阻塞；同步-异步

IO操作分了两个过程：**等待 + 数据拷贝**

IO模型：
+ **阻塞式IO**
	- 发起IO调用，若不具备IO条件，则等待IO条件具备。具备则数据拷贝完毕后返回
+ **非阻塞式IO**
	- 发起IO调用，若不具备条件则立即报错返回，通常是循环发起调用。若具备IO条件，则拷贝数据完毕后返回
+ **事件/信号驱动IO**
	- 先定义IO信号处理方式，若IO条件具备，直接信号通知进程，发起调用，拷贝数据后返回。
	- 流程控制较难，也是一种异步，因为拷贝是异步的
+ **IO多路复用**
	- 一种IO事件监控。同时对大量的描述符进行事件(描述符的可读/可写/异常)，默认阻塞监控，判断监控描述符是否具备IO条件。如果具备(就绪时)进行返回，对就绪的IO进行操作
	- 是高并发的处理模型
	- select、poll、epoll都是实现对大量描述符进行事件监控的操作
+ **异步IO：AIO**
	- 定义信号处理，发起异步IO调用，自己直接返回，之后让别人等待条件具备(等待和数据拷贝都不用自己完成)，IO条件具备后，数据拷贝也由别人完成，最后信号通知进程：IO已经完成，可以对数据直接进行操作
	
#### select、poll、epoll

+ select
	- 遵循POSIX标准，可以**跨平台**
	- select监控的超时等待时间更加精细(微秒级别)
	- select所能监控的描述符是有上限的，Linux下默认1024，取决于`__FD_SETSIZE`
	- select实现监控原理是在内核中进行轮询遍历状态，因此性能会跟着描述符增多而下降
	- select监控每次返回时都会修改监控集合，需要用户每次监控前重新添加描述符到集合中
	- select不会直接告诉用户哪一个描述符事件就绪，只是告诉用户有就绪事件，需要用户遍历查找
+ poll
	- 采用事件结构的方式对描述符进行监控，简化了多个事件集合的监控方式
	- 描述符的具体监控无上限
	- 不能跨平台。所以poll已经逐渐在历史舞台上淡出
	- poll采用轮询遍历判断就绪，性能随着描述符增多而性能下降
	- poll也不会告诉用户具体就绪的描述符，需要用户进行轮询判断
+ epoll
	- Linux下**性能最高**的IO多路转接模型，也是采用事件结构的形式对描述符进行监控
	- 就绪的描述符对应事件拷贝一份到用户态，直接告诉用户有**哪些描述符就绪**
	
select，poll实现需要自己不断**轮询**所有fd集合，直到设备就绪，期间可能要睡眠和唤醒多次交替。而epoll其实也需要调用epoll_wait不断轮询就绪链表，期间也可能多次睡眠和唤醒交替，但是它是设备就绪时，调用**回调函数**，把就绪fd放入就绪链表中，并唤醒在epoll_wait中进入睡眠的进程。虽然都要**睡眠和交替**，但是select和poll在"醒着"的时候要遍历整个fd集合，而epoll在"醒着"的时候只要判断一下**就绪链表**是否为空就行了，这节省了大量的CPU时间。这就是回调机制带来的性能提升	

select，poll每次调用都要把fd集合从用户态往内核态拷贝一次，epoll通过**mmap**把内核空间和用户空间映射到同一块内存，省去了拷贝的操作

#### 同步阻塞IO型

![io1](/img/in-post/2021/04/io1.jpg)

#### 非阻塞IO模型

![io2](/img/in-post/2021/04/io2.jpg)

#### IO复用模型

![io3](/img/in-post/2021/04/io3.jpg)

#### 异步IO模型

![io4](/img/in-post/2021/04/io4.jpg)

#### 信号驱动IO模型

![io5](/img/in-post/2021/04/io5.jpg)

#### 比较

![io-compare](/img/in-post/2021/04/io-compare.png)

#### 事件驱动：Reactor模型

![reactor](/img/in-post/2021/04/reactor.png)

+ 事件驱动
+ 可以处理一个或多个输入源
+ 通过Service Handler同步的将输入事件(Event)采用多路复用分发给相应的Request Handler(多个)处理
	- 同步的等待多个事件源到达(采用`select()`实现)
	- 将事件多路分解以及分配相应的事件服务进行处理，这个分派采用server集中处理(dispatch)
	- 分解的事件以及对应的事件服务应用从分派服务中分离出去(handler)

多Reactor多线程模型：
![reactor2](/img/in-post/2021/04/reactor2.png)

+ Reactor
	- 负责监听和分配事件，将I/O事件分派给对应的Handler。新的事件包含连接建立就绪、读就绪、写就绪等
+ Acceptor
	- 处理客户端新连接，并分派请求到处理器链中
+ Handler
	- 将自身与事件绑定，执行非阻塞读/写任务，完成channel的读入，完成处理业务逻辑后，负责将结果写出channel。可用资源池来管理

#### 异步IO：Proactor模型

Proactor是一种异步I/O模型，在Proactor中直接由事件分发者处理一个事件的读写，而实际的工作由操作系统完成。显然和reactor的区别就是：reactor是有事件就绪就调用注册的函数进行读写，而Proactor是由OS处理完后，才调用处理者处理。Reactor模式属于同步非阻塞I/O的网络通信模型，而Proactor运属于异步非阻塞I/O的网络通信模型

![proactor](/img/in-post/2021/04/proactor.png)

+ Epoll
	- 是"事件分离器"对就绪事件的发现方式，有select、poll与epoll三种方式
	- epoll采用的是回调方式，而不是轮询方式
	- 当出现大批量的读/写事件切换时，epoll的效率会远远低于poll。因为epoll需要进行大量的用户空间到内核空间的切换，而poll仅需要在用户空间做简单的位运算即可完成
	- epoll完全属于Linux，虽然其它系统平台也有epoll的支持，但并不完全相同
+ Proactor
	- 是一种网络通信模型，该模型中就不存在"事件分离器"
	- Reactor模型中具有"事件分离器"

#### MMAP

在LINUX中我们可以使用mmap用来在进程虚拟内存地址空间中分配地址空间，创建和物理内存的映射关系

映射关系可以分为：
+ **文件映射**
	- 磁盘文件映射进程的虚拟地址空间，使用文件内容初始化物理内存
+ **匿名映射**
	- 初始化全为0的内存空间

映射关系是否共享又分为：
+ **私有映射(MAP_PRIVATE)**
	- 多进程间数据共享，修改不反应到磁盘实际文件，是一个copy-on-write(写时复制)的映射方式
+ **共享映射(MAP_SHARED)**
	- 多进程间数据共享，修改反应到磁盘实际文件中

因此两两组合有四种映射：
+ **私有文件映射**
	- 多个进程使用同样的物理内存页进行初始化，但是各个进程对内存文件的修改不会共享，也不会反应到物理文件中
+ **私有匿名映射**
	- mmap会创建一个新的映射，各个进程不共享，这种使用主要用于分配内存(malloc分配大内存会调用mmap)
	- 例如开辟新进程时，会为每个进程分配虚拟的地址空间，这些虚拟地址映射的物理内存空间各个进程间读的时候共享，写的时候会copy-on-write
+ **共享文件映射**
	- 多个进程通过虚拟内存技术共享同样的物理内存空间，对内存文件的修改会反应到实际物理文件中，他也是进程间通信(IPC)的一种机制
+ **共享匿名映射**
	- 这种机制在进行fork的时候不会采用写时复制，父子进程完全共享同样的物理内存页，这也就实现了父子进程通信(IPC).

值得注意的是，**mmap只是在虚拟内存分配了地址空间**(并没有将文件内容加载到物理页上)，只有在**第一次**访问虚拟内存的时候(产生"缺页")才**分配物理内存**(只加载缺页，不过受操作系统一些调度策略影响会加载比所需的多)

+ write
	1. 进程(用户态)将需要写入的数据直接copy到对应的mmap地址(内存copy)
	2. 若mmap地址未对应物理内存，则产生缺页异常，由内核处理
	3. 若已对应，则直接copy到对应的物理内存
	4. 由操作系统调用，将脏页回写到磁盘(通常是异步的)

因为物理内存是有限的，mmap在写入数据超过物理内存时，操作系统会进行**页置换**，根据淘汰算法，将需要淘汰的页置换成所需的新页，所以mmap对应的内存是**可以**被淘汰的。而若内存页是"脏"的，则操作系统会先将数据回写磁盘再淘汰。这样，就算mmap的数据远大于物理内存，操作系统也能很好地处理，不会产生功能上的问题

![mmap-read](/img/in-post/2021/04/mmap-read.png)
	
+ read
	- mmap要比普通的read系统调用少了一次copy的过程。因为read调用，进程是无法直接访问kernel space的，所以在read系统调用返回前，内核需要将数据从内核**复制到**进程指定的buffer。但mmap之后，进程可以**直接访问mmap的数据(page cache)**

优点：
+ 对文件的读取操作**跨过了页缓存**，减少了数据的拷贝次数，用内存读写取代I/O读写，提高了文件读取效率
+ 实现了用户空间和内核空间的高效交互方式。两空间的各自修改操作可以**直接反映在映射的区域内**，从而被对方空间及时捕捉
+ 提供进程间**共享内存及相互通信**的方式。不管是父子进程还是无亲缘关系的进程，都可以将自身用户空间映射到同一个文件或匿名映射到同一片区域。从而通过各自对映射区域的改动，达到进程间通信和进程间共享的目的。同时，如果进程A和进程B都映射了区域C，当A第一次读取C时通过缺页从磁盘复制文件页到内存中；但当B再读C的相同页面时，虽然也会产生缺页异常，但是不再需要从磁盘中复制文件过来，而可直接使用已经保存在内存中的文件数据
+ 可用于实现高效的**大规模数据传输**。内存空间不足，是制约大数据操作的一个方面，解决方案往往是借助硬盘空间协助操作，补充内存的不足。但是进一步会造成大量的文件I/O操作，极大影响效率。这个问题可以通过mmap映射很好的解决。换句话说，但凡是需要用磁盘空间代替内存的时候，mmap都可以发挥其功效

缺点：
+ 文件如果很小，是小于4096字节的，比如10字节，由于内存的最小粒度是页，而进程虚拟地址空间和内存的映射也是以页为单位。虽然被映射的文件只有10字节，但是对应到进程虚拟地址区域的大小需要满足整页大小，因此mmap函数执行后，实际映射到虚拟内存区域的是4096个字节，11~4096的字节部分用零填充。因此会**浪费内存空间**
+ 对变长文件不适合，**文件无法完成拓展**，因为mmap到内存的时候，你所能够操作的范围就确定了
+ 如果更新文件的操作很多，会**触发大量的脏页回写**及由此引发的**随机IO**上。所以在随机写很多的情况下，mmap方式在效率上不一定会比带缓冲区的一般写快

总结来说，常规文件操作为了提高读写效率和保护磁盘，使用了**页缓存机制**。这样造成读文件时需要先将文件页从磁盘**拷贝**到页缓存中，由于页缓存处在**内核空间**，不能被用户进程直接寻址，所以还需要将页缓存中数据页**再次拷贝**到内存对应的**用户空间**中。这样，通过了**两次**数据拷贝过程，才能完成进程对文件内容的获取任务。写操作也是一样，待写入的buffer在内核空间不能直接访问，必须要先**拷贝**至内核空间对应的**主存**，再**写回磁盘**中(延迟写回)，也是需要两次数据拷贝。

而使用mmap操作文件中，创建新的**虚拟内存区域**和建立文件磁盘地址和虚拟内存区域**映射**这两步，没有任何文件拷贝操作。而之后访问数据时发现内存中并无数据而发起的**缺页异常过程**，可以通过已经建立好的映射关系，只使用**一次数据拷贝**，就从磁盘中将数据传入内存的用户空间中，供进程使用



扩展：  
[操作系统 I/O 全流程详解](https://www.cnblogs.com/cxuanBlog/p/13156493.html)  
[深度分析mmap](https://blog.csdn.net/qq_33611327/article/details/81738195)  

