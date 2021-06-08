---
layout:     post
title:      "Dubbo(二):架构与SPI"
date:       2018-08-10
author:     "ZhouJ000"
header-img: "img/in-post/2018/post-bg-2018-headbg.jpg"
catalog: true
tags:
    - rpc
    - 分布式
--- 

<font id="last-updated">最后更新于：2018-08-10</font>

[Dubbo(一):从dubbo-demo初探dubbo源码](https://zhouj000.github.io/2018/07/16/dubbo-demo-exploration/)  
[Dubbo(二):架构与SPI](https://zhouj000.github.io/2018/08/10/dubbo-framework-spi/)  
[Dubbo(三):高并发下降级与限流](https://zhouj000.github.io/2019/02/18/dubbo-03/)  
[Dubbo(四):其他常用特性](https://zhouj000.github.io/2019/02/19/dubbo-04/)  
[Dubbo(五):自定义扩展](https://zhouj000.github.io/2019/02/20/dubbo-05/)  



# 关于dubbo

## dubbo是什么

Dubbo是Alibaba开源的一个分布式服务框架，致力于提供高性能和透明化的RPC远程服务调用方案，以及SOA服务治理方案。
[Dubbo官网文档](http://dubbo.apache.org/#!/docs/user/quick-start.md?lang=zh-cn)

## dubbo做了什么

1 . 透明化的远程方法调用，就像调用本地方法一样调用远程方法。  
2 . 服务自动注册与发现，解决各服务间复杂的依赖关系，注册中心基于接口名查询服务提供者的IP地址，并且能平滑上下架服务提供者。  
3 . 软负载均衡和Failover，降低对F5硬件负载均衡器的依赖，也能减少部分成本。  
4 . 服务监控与统计。



# dubbo 架构

## dubbo 层级
![dubbo-framework](/img/in-post/2018/8/dubbo-framework.jpg)
图例说明：
+ 图中左边淡蓝背景的为服务消费方使用的接口，右边淡绿色背景的为服务提供方使用的接口，位于中轴线上的为双方都用到的接口。
+ 图中从下至上分为十层，各层均为单向依赖，右边的黑色箭头代表层之间的依赖关系，每一层都可以剥离上层被复用，其中，Service 和 Config 层为 API，其它各层均为 SPI。
+ 图中绿色小块的为扩展接口，蓝色小块为实现类，图中只显示用于关联各层的实现类。
+  图中蓝色虚线为初始化过程，即启动时组装链，红色实线为方法调用过程，即运行时调时链，紫色三角箭头为继承，可以把子类看作父类的同一个节点，线上的文字为调用的方法。

****

- Bussiness业务
	+ 服务接口层 service: 与实际业务逻辑相关的，根据服务提供方和服务消费方的业务设计对应的接口和实现
- RPC 
	+ 配置层 config: 对外配置接口，以 ServiceConfig, ReferenceConfig 为中心，可以直接初始化配置类，也可以通过 spring 解析配置生成配置类
	+ 服务代理层 proxy: 服务接口透明代理，生成服务的客户端 Stub 和服务器端 Skeleton, 以 ServiceProxy 为中心，扩展接口为 ProxyFactory
	+ 服务注册层 registry: 封装服务地址的注册与发现，以服务 URL 为中心，扩展接口为 RegistryFactory, Registry, RegistryService。可能没有服务注册中心，此时服务提供方直接暴露服务
	+ 集群层 cluster: 封装多个提供者的路由及负载均衡，并桥接注册中心，以 Invoker 为中心，扩展接口为 Cluster, Directory, Router, LoadBalance。将多个服务提供方组合为一个服务提供方，实现对服务消费方来透明，只需要与一个服务提供方进行交互
	+ 监控层 monitor: RPC 调用次数和调用时间监控，以 Statistics 为中心，扩展接口为 MonitorFactory, Monitor, MonitorService
	+ **远程调用层 protocol**: 封装 RPC 调用，以 Invocation, Result 为中心，扩展接口为 Protocol, Invoker, Exporter。Protocol是服务域，它是Invoker暴露和引用的主功能入口，它负责Invoker的生命周期管理。Invoker是实体域，它是Dubbo的核心模型，其它模型都向它靠扰，或转换成它，它代表一个可执行体，可向它发起invoke调用，它有可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现
- Remoting
	+ 信息交换层 exchange: 封装请求响应模式，同步转异步，以 Request, Response 为中心，扩展接口为 Exchanger, ExchangeChannel, ExchangeClient, ExchangeServer
	+ 网络传输层 transport: 抽象 mina 和 netty 为统一接口，以 Message 为中心，扩展接口为 Channel, Transporter, Client, Server, Codec
	+ 数据序列化层 serialize: 可复用的一些工具，扩展接口为 Serialization, ObjectInput, ObjectOutput, ThreadPool

****

+ 在 RPC 中，Protocol 是核心层，也就是只要有 Protocol + Invoker + Exporter 就可以完成非透明的 RPC 调用，然后在 Invoker 的主过程上 Filter 拦截点。
+ 图中的 Consumer 和 Provider 是抽象概念，只是想让看图者更直观的了解哪些类分属于客户端与服务器端，不用 Client 和 Server 的原因是 Dubbo 在很多场景下都使用 Provider, Consumer, Registry, Monitor 划分逻辑拓普节点，保持统一概念。
+ 而 Cluster 是外围概念，所以 Cluster 的目的是将多个 Invoker 伪装成一个 Invoker，这样其它人只要关注 Protocol 层 Invoker 即可，加上 Cluster 或者去掉 Cluster 对其它层都不会造成影响，因为只有一个提供者时，是不需要 Cluster 的。
+ Proxy 层封装了所有接口的透明化代理，而在其它层都以 Invoker 为中心，只有到了暴露给用户使用时，才用 Proxy 将 Invoker 转成接口，或将接口实现转成 Invoker，也就是去掉 Proxy 层 RPC 是可以 Run 的，只是不那么透明，不那么看起来像调本地服务一样调远程服务。
+ 而 Remoting 实现是 Dubbo 协议的实现，如果你选择 RMI 协议，整个 Remoting 都不会用上，Remoting 内部再划为 Transport 传输层和 Exchange 信息交换层，Transport 层只负责单向消息传输，是对 Mina, Netty, Grizzly 的抽象，它也可以扩展 UDP 传输，而 Exchange 层是在传输层之上封装了 Request-Response 语义。
+ Registry 和 Monitor 实际上不算一层，而是一个独立的节点，只是为了全局概览，用层的方式画在一起。

## dubbo模块分包
![dubbo-relation](/img/in-post/2018/8/dubbo-modules.jpg)
模块说明：
+ dubbo-common 公共逻辑模块：包括 Util 类和通用模型。
+ dubbo-remoting 远程通讯模块：相当于 Dubbo 协议的实现，如果 RPC 用 RMI协议则不需要使用此包。
+ dubbo-rpc 远程调用模块：抽象各种协议，以及动态代理，只包含一对一的调用，不关心集群的管理。
+ dubbo-cluster 集群模块：将多个服务提供方伪装为一个提供方，包括：负载均衡, 容错，路由等，集群的地址列表可以是静态配置的，也可以是由注册中心下发。
+ dubbo-registry 注册中心模块：基于注册中心下发地址的集群方式，以及对各种注册中心的抽象。
+ dubbo-monitor 监控模块：统计服务调用次数，调用时间的，调用链跟踪的服务。
+ dubbo-config 配置模块：是 Dubbo 对外的 API，用户通过 Config 使用D ubbo，隐藏 Dubbo 所有细节。
+ dubbo-container 容器模块：是一个 Standlone 的容器，以简单的 Main 加载 Spring 启动，因为服务通常不需要 Tomcat/JBoss 等 Web 容器的特性，没必要用 Web 容器去加载服务。

整体上按照分层结构进行分包，与分层的不同点在于：
+ container 为服务容器，用于部署运行服务，没有在层中画出。
+ protocol 层和 proxy 层都放在 rpc 模块中，这两层是 rpc 的核心，在不需要集群也就是只有一个提供者时，可以只使用这两层完成 rpc 调用。
+ transport 层和 exchange 层都放在 remoting 模块中，为 rpc 调用的通讯基础。
+ serialize 层放在 common 模块中，以便更大程度复用。

## dubbo依赖关系
![dubbo-relation](/img/in-post/2018/8/dubbo-relation.jpg)	

### 调用链
![dubbo-extension](/img/in-post/2018/8/dubbo-extension.jpg)	
[官方文档: 框架设计](http://dubbo.apache.org/#!/docs/dev/design.md?lang=zh-cn)



# SPI

## java的SPI

> Service Provider Interface : 服务提供接口。
JavaSPI 实际上是“基于接口的编程＋策略模式＋配置文件”组合实现的动态加载机制：
1. 定义一组接口(com.test.Car)
2. 写出接口的一个或多个实现(Bus, Jeep)
3. 在`src/main/resources/`下建立`/META-INF/services`目录，新增一个以接口命名的文件`com.test.Car`
4. 内容是要应用的实现类(com.test.Bus 或 com.test.Jeep 或 2个都写)
5. 使用`ServiceLoader`来加载配置文件中指定的实现

```
例子: JDBC 数据库驱动包: 可替换的插件机制

mysql-connector-java-5.1.18.jar 中就有一个 /META-INF/services/java.sql.Driver的文件
里面内容为: com.mysql.jdbc.Driver

ojdbc8.jar 中同样有个/META-INF/services/java.sql.Driver的文件
内容为: oracle.jdbc.OracleDriver

JDBC 4.0以后DriverManager类会在静态代码块中loadInitialDrivers()，里面会通过SPI获取drivers自动加载类到JVM内存
ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
Iterator driversIterator = loadedDrivers.iterator();

如果有多个驱动时，DriverManager.getConnection中会遍历所有已经加载的驱动实例去创建连接，当一个驱动创建连接成功时就会返回这个连接，同时不再调用其他的驱动实例。
每个驱动实例在getConnetion的第一步就是按照url判断是不是符合自己的处理规则，是的话才会和db建立连接。所以并不是每个驱动实例都去实际尝试建立连接的。
```

> ServiceLoader 的实现涉及到如下概念： 1.指向对象类型的 Class&lt;S> 对象； 2.类加载器 ClassLoader； 3.服务实现类的资源抽象； 4.服务实现类的全名字符串。  结合类加载器和资源抽象获得服务实现类的全名字符串，再通过类加载器获取 Class&lt;S> 对象，最后通过 Class&lt;S> 对象来构造服务实现类 S 的实例 s 。

## dubbo的SPI
```java
public @interface SPI {
	String value() default ""; //指定默认的扩展点
}

@SPI("dubbo")
public interface Protocol{
}
```
只有在接口有@SPI注解的接口类才会去查找扩展点实现，会依次从这几个文件中读取扩展点：
+ META-INF/dubbo/internal/		// dubbo内部实现的各种扩展都放在了这个目录了
+ META-INF/dubbo/
+ META-INF/services/

例如，Dubbo-rpc模块默认protocol实现DubboProtocol，key为dubbo:  
![internal-Protocol](/img/in-post/2018/8/internal-Protocol.png)
ProtocolListenerWrapper -> ProtocolFilterWrapper (-> DubboProtocol)
![internal-Protocol2](/img/in-post/2018/8/internal-Protocol2.png)

SPI组件在dubbo称为ExtensionLoader–扩展容器: Extension 扩展点、Wrapper 包装类、Activate 激活点、Adaptive 代理。

Adaptive的决策过程，最终还是会通过容器获得当次调用的扩展点来完成调用：
- 取得扩展点的key
	1. Adaptive方法必须保证方法参数中有一个参数是URL类型或存在能返回URL对象的方法，容器通过反射探测并自动获取。 
	2. key默认是接口类型上的@SPI#value，方法上的@Adaptive#value有更高的优先级。 
	3. 如果2都为空则以interface class#simpleName为key， key生成规则:”AbcDefGh”=>”abc.def.gh” 
- 如果key值是protocol，则直接调用URL#getProtocol方法获取扩展点名称，如果不是则使用URL#getMethodParameter(key)或getParameter(key)，其返回值作为扩展点名称
- 通过ExtensionFactory获取扩展点。ExtensionFactory用于获取指定名称的扩展点，它最终还是通过容器的getExtension去获取或生成。如果用户声明的provider里有wrapper，则返回的是被包装过的实体对象


### 与java的SPI区别
1 . dubbo SPI 可以通过getExtension(String key)的方法方便的获取某一个想要的扩展实现，java的SPI机制需要加载全部的实现类。  
2 . 提供注解方式，可以方便的扩展实现。  
3 . 对于扩展实现IOC依赖注入功能，如现在实现者A1含有setB()方法，会自动注入一个接口B的实现者，此时会注入一个动态生成的接口B的实现者B$Adpative


### ExtensionLoader类

> ExtensionLoader是Dubbo中的SPI的实现方法，它是Dubbo框架的微容器，也为框架提供各种组件的扩展点

三种注解:  
1 . SPI  
2 . Adaptive  
3 . Activate

#### getExtensionLoader(Class<T> type)
每个定义的SPI的接口都会构建一个ExtensionLoader实例，存储在ConcurrentMap<Class<?>,ExtensionLoader<?>> EXTENSION_LOADERS 这个map对象中
```java
private ExtensionLoader(Class<?> type) {
	this.type = type;
	objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
}

@SPI
public interface ExtensionFactory {
    <T> T getExtension(Class<T> type, String name);
}

private T createAdaptiveExtension() {
	try {
		return injectExtension((T) getAdaptiveExtensionClass().newInstance());
	} catch (Exception e) {
		throw new IllegalStateException("Can not create adaptive extension " + type + ", cause: " + e.getMessage(), e);
	}
}
```
![internal-Protocol2](/img/in-post/2018/8/dubbo-spi-1.jpg)

#### loadExtensionClasses()
先读取SPI注解的value值，有值作为默认扩展实现的key。依次读取上述3个默认路径下的文件

#### loadDirectory(Map<String, Class<?>> extensionClasses, String dir, String type)
获取文件夹下文件的URL，遍历执行loadResource(Map<String, Class<?>> extensionClasses, ClassLoader classLoader, java.net.URL resourceURL)方法  
将文件内容每一行读取出来，遍历每一行执行loadClass方法

#### loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name)
1. 判断clazz是否与接口类型一致
2. 判断实现类上有没有@Adaptive注解
3. 如果有，将此类设为cachedAdaptiveClass作为适配类缓存起来。否则适配类通过javasisit修改字节码生成
4. 如果没有，isWrapperClass判断实现类是否存在入参为接口的构造器(就是DubbboProtocol类是否有入参为Protocol的构造器)，有的话作为包装类缓存到此ExtensionLoader的Set<Class<?>>集合cachedWrapperClasses中，这个其实是个装饰模式。类似ProtocolListenerWrapper与ProtocolFilterWrapper都有传入Protocol的构造方法。
5. 如果既不是适配对象，也不是wrapped的对象，那就是扩展点的具体实现对象。查找实现类上有没有@Activate注解，有则缓存到变量cachedActivates的map中。这里同时兼容老版本的Activate注解，判断存在也放入cachedActivates中。最后将实现类缓存到cachedClasses中，以便于使用时获取
```java
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
	if (!type.isAssignableFrom(clazz)) {
		throw new IllegalStateException("Error when load extension class(interface: " +
				type + ", class line: " + clazz.getName() + "), class "
				+ clazz.getName() + "is not subtype of interface.");
	}
	if (clazz.isAnnotationPresent(Adaptive.class)) {
		if (cachedAdaptiveClass == null) {
			cachedAdaptiveClass = clazz;
		} else if (!cachedAdaptiveClass.equals(clazz)) {
			throw new IllegalStateException("More than 1 adaptive class found: "
					+ cachedAdaptiveClass.getClass().getName()
					+ ", " + clazz.getClass().getName());
		}
	} else if (isWrapperClass(clazz)) {
		Set<Class<?>> wrappers = cachedWrapperClasses;
		if (wrappers == null) {
			cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
			wrappers = cachedWrapperClasses;
		}
		wrappers.add(clazz);
	} else {
		clazz.getConstructor();
		if (name == null || name.length() == 0) {
			name = findAnnotationName(clazz);
			if (name.length() == 0) {
				throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + resourceURL);
			}
		}
		String[] names = NAME_SEPARATOR.split(name);
		if (names != null && names.length > 0) {
			Activate activate = clazz.getAnnotation(Activate.class);
			if (activate != null) {
				cachedActivates.put(names[0], activate);
			} else {
				// support com.alibaba.dubbo.common.extension.Activate
				com.alibaba.dubbo.common.extension.Activate oldActivate = clazz.getAnnotation(com.alibaba.dubbo.common.extension.Activate.class);
				if (oldActivate != null) {
					cachedActivates.put(names[0], oldActivate);
				}
			}
			for (String n : names) {
				if (!cachedNames.containsKey(clazz)) {
					cachedNames.put(clazz, n);
				}
				Class<?> c = extensionClasses.get(n);
				if (c == null) {
					extensionClasses.put(n, clazz);
				} else if (c != clazz) {
					throw new IllegalStateException("Duplicate extension " + type.getName() + " name " + n + " on " + c.getName() + " and " + clazz.getName());
				}
			}
		}
	}
}
```

#### getAdaptiveExtensionClass()
回到getAdaptiveExtensionClass方法:  
1 . 如果 cachedAdaptiveClass 有值，说明有且仅有一个实现类有@Adaptive, 实例化这个对象返回。  
2 . 如果 cachedAdaptiveClass 为空，调用createAdaptiveExtensionClass()创建适配类字节码。

> 为什么要创建适配类，一个接口多种实现，SPI机制也是如此，这是策略模式，但是我们在代码执行过程中选择哪种具体的策略呢。Dubbo采用统一数据模式com.alibaba.dubbo.common.URL(它是dubbo定义的数据模型不是jdk的类)，它会穿插于系统的整个执行过程，URL中定义的协议类型字段protocol，会根据具体业务设置不同的协议。url.getProtocol()值可以是dubbo也是可以webservice，可以是zookeeper也可以是redis。

适配类的作用是根据url.getProtocol()的值extName，去ExtensionLoader.getExtension(extName)选取具体的扩展点实现。

所以能够利用javasist生成设配类的条件：
1. 接口方法中必须至少有一个方法有@Adaptive注解。
2. 有@Adaptive注解的方法参数必须有URL类型参数或者有参数中存在getURL()方法。 

```java
private Class<?> createAdaptiveExtensionClass() {
	// 生成Adaptive代码code
	String code = createAdaptiveExtensionClassCode();
	ClassLoader classLoader = findClassLoader();
	// 利用dubbo的spi扩展机制获取compiler的适配类
	org.apache.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
	// 编译生成的adaptive代码
	return compiler.compile(code, classLoader);
}
```
createAdaptiveExtensionClassCode()方法生成javasist用来生成Protocol适配类后的代码：
```java
package org.apache.dubbo.rpc;
import org.apache.dubbo.common.extension.ExtensionLoader;

public class Protocol$Adaptive implements org.apache.dubbo.rpc.Protocol {

    // 没有@Adaptive的方法如果被调到抛异常
    public void destroy() {
        throw new UnsupportedOperationException("method public abstract void org.apache.dubbo.rpc.Protocol.destroy() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");

    }

    // 没有@Adaptive的方法如果被调到抛异常
    public int getDefaultPort() {
        throw new UnsupportedOperationException("method public abstract int org.apache.dubbo.rpc.Protocol.getDefaultPort() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }

    // 接口中export方法上有@Adaptive注解
	public org.apache.dubbo.rpc.Exporter export(org.apache.dubbo.rpc.Invoker arg0) throws org.apache.dubbo.rpc.RpcException {
        if (arg0 == null) 
			throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
        
        // 参数类中要有URL属性
        if (arg0.getUrl() == null) 
			throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
			
		// 从入参获取统一数据模型URL
		org.apache.dubbo.common.URL url = arg0.getUrl();    
		String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
        
        // 从统一数据模型URL获取协议，协议名就是SPI扩展点实现类的key
        if(extName == null) 
			throw new IllegalStateException("Fail to get extension(org.apache.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");

        // 利用dubbo服务查找机制根据名称找到具体的扩展点实现
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);

        // 调具体扩展点的方法
        return extension.export(arg0);
    }

    // 接口中refer方法上有@Adaptive注解
    public org.apache.dubbo.rpc.Invoker refer(java.lang.Class arg0, org.apache.dubbo.common.URL arg1) 
		throws org.apache.dubbo.rpc.RpcException {

        // 统一数据模型URL不能为空
        if (arg1 == null)
            throw new IllegalArgumentException("url == null");

        org.apache.dubbo.common.URL url = arg1;
        
        // 从统一数据模型URL获取协议，协议名就是SPI扩展点实现类的key
        String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
        if (extName == null)
            throw new IllegalStateException("Fail to get extension(org.apache.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");

        // 利用dubbo服务查找机制根据名称找到具体的扩展点实现
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        
		// 调具体扩展点的方法
        return extension.refer(arg0, arg1);
    }
}
```
compiler.compile动态编译代码
```
@SPI("javassist")	// 如果没有配置，默认选用javassist编译源代码
public interface Compiler {
	Class<?> compile(String code, ClassLoader classLoader);
}
```
![internal-Protocol2](/img/in-post/2018/8/Compiler.png)


#### getExtension(String name)
按名称查找extension，先从缓冲中获取，没有则创建。
![internal-Protocol2](/img/in-post/2018/8/dubbo-spi-2.jpg)

#### createExtension(String name)
createExtension是创建扩展点的入口，先通过getExtensionClasses加载三个路径下对应的扩展类，然后调用injectExtension注入依赖。然后判断此扩展是否有包装类缓存，有的话利用包装器增强这个扩展点实现的功能。
```java
private T createExtension(String name) {
	Class<?> clazz = getExtensionClasses().get(name);
	if (clazz == null) {
		throw findException(name);
	}
	try {
		T instance = (T) EXTENSION_INSTANCES.get(clazz);
		if (instance == null) {
			EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
			instance = (T) EXTENSION_INSTANCES.get(clazz);
		}
		injectExtension(instance);
		Set<Class<?>> wrapperClasses = cachedWrapperClasses;
		if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
			for (Class<?> wrapperClass : wrapperClasses) {
				instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
			}
		}
		return instance;
	} catch (Throwable t) {
		throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
				type + ")  could not be instantiated: " + t.getMessage(), t);
	}
}
```

#### injectExtension(T instance)
实现了简单的IOC机制，对扩展实现中公有的set方法且入参个数为一个的方法依赖注入。如果此时依赖没有创建好，尝试通过对象工厂objectFactory.getExtension递归创建扩展点。
```java
private T injectExtension(T instance) {
	try {
		if (objectFactory != null) {
			for (Method method : instance.getClass().getMethods()) {
				if (method.getName().startsWith("set")
						&& method.getParameterTypes().length == 1
						&& Modifier.isPublic(method.getModifiers())) {
					Class<?> pt = method.getParameterTypes()[0];
					try {
						String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
						Object object = objectFactory.getExtension(pt, property);
						if (object != null) {
							method.invoke(instance, object);
						}
					} catch (Exception e) {
						logger.error("fail to inject via method " + method.getName()
								+ " of interface " + type.getName() + ": " + e.getMessage(), e);
					}
				}
			}
		}
	} catch (Exception e) {
		logger.error(e.getMessage(), e);
	}
	return instance;
}
```
objectFactory.getExtension，objectFactory的实现类是AdaptiveExtensionFactory，getExtension方法是一个入口，最终实现是在factories中即是SpiExtensionFactory
```java
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory{ 
	...
	
	// 持有所有ExtensionFactory对象的集合，dubbo内部默认实现的对象工厂是SpiExtensionFactory和SrpingExtensionFactory
	// 经过TreeMap排好序的查找顺序是优先先从SpiExtensionFactory获取，如果返回空在从SpringExtensionFactory获取
	public <T> T getExtension(Class<T> type, String name) {
        for (ExtensionFactory factory : factories) {
            T extension = factory.getExtension(type, name);
            if (extension != null) {
                return extension;
            }
        }
        return null;
    }
}

// SpiExtensionFactory
public <T> T getExtension(Class<T> type, String name) {
	// 传入的参数类型必须是接口类型并且接口上注释了@SPI注解
	if (type.isInterface() && type.isAnnotationPresent(SPI.class)) {
		ExtensionLoader<T> loader = ExtensionLoader.getExtensionLoader(type);
		if (!loader.getSupportedExtensions().isEmpty()) {
			// 返回的是一个适配类对象
			return loader.getAdaptiveExtension();
		}
	}
	return null;
}
```

#### 包装类增强
比如Protocol继承关系ProtocolFilterWrapper, ProtocolListenerWrapper这个两个类是装饰对象用来增强其他扩展点实现的功能。  **ProtocolFilterWrapper**功能主要是在refer 引用远程服务的中透明的设置一系列的过滤器链用来记录日志，处理超时，权限控制等等功能。  **ProtocolListenerWrapper**在provider的exporter,unporter服务和consumer的refer服务，destory调用时添加监听器，dubbo提供了扩展但是没有默认实现哪些监听器。

它们会从cachedWrapperClasses中获取后，包装器增强Protocol实现`injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance))`。
当获取DubboProtocol时，就会拿到这2个Wapper包装增强。




### 扩展点

Dubbo的扩展点加载从 JDK 标准的 SPI (Service Provider Interface) 扩展点发现机制加强而来。

**约定：**  
在扩展类的 jar 包内，放置扩展点配置文件 META-INF/dubbo/接口全限定名，内容为：配置名=扩展实现类全限定名，多个实现类用换行符分隔。这里的配置文件是放在自己的 jar 包内，不是 dubbo 本身的 jar 包内，Dubbo 会全 ClassPath 扫描所有 jar 包内同名的这个文件，然后进行合并。

```java
// META-INF/dubbo/com.alibaba.dubbo.rpc.Protocol
xxx=com.alibaba.xxx.XxxProtocol

public class XxxProtocol implemenets Protocol { 
    // 自定义扩展...
}
```
扩展点使用单一实例加载(请确保扩展实现的线程安全性)，缓存在 ExtensionLoader 中

### 扩展点特性

#### 扩展点自动包装
自动包装扩展点的 Wrapper 类。ExtensionLoader 在加载扩展点时，如果加载到的扩展点有拷贝构造函数，则判定为扩展点 Wrapper 类。
```java
public class XxxProtocolWrapper implemenets Protocol {
    Protocol impl;
    public XxxProtocol(Protocol protocol) { impl = protocol; }
 
    // 接口方法做一个操作后，再调用extension的方法
    public void refer() {
        // ... 一些操作
        impl.refer();
        // ... 一些操作
    }
    // ...
}
```
Wrapper 类同样实现了扩展点接口，但是 Wrapper 不是扩展点的真正实现。它的用途主要是用于从 ExtensionLoader 返回扩展点时，包装在真正的扩展点实现外。即从 ExtensionLoader 中返回的实际上是 Wrapper 类的实例，Wrapper 持有了实际的扩展点实现类。

扩展点的 Wrapper 类可以有多个，也可以根据需要新增。

通过 Wrapper 类可以把所有扩展点公共逻辑移至 Wrapper 中。新加的 Wrapper 在所有的扩展点上添加了逻辑，有些类似 AOP，即 Wrapper 代理了扩展点。

#### 扩展点自动装配
加载扩展点时，自动注入依赖的扩展点。加载扩展点时，扩展点实现类的成员如果为其它扩展点类型，ExtensionLoader 在会自动注入依赖的扩展点。ExtensionLoader 通过扫描扩展点实现类的所有 setter 方法来判定其成员。即 ExtensionLoader 会执行扩展点的拼装操作。类似于自动加上@Autowired。

#### 扩展点自适应
ExtensionLoader 注入的依赖扩展点是一个 Adaptive 实例，直到扩展点方法执行时才决定调用是一个扩展点实现。

**Dubbo 使用 URL 对象(包含了Key-Value)传递配置信息。**

扩展点方法调用会有URL参数(或是参数有URL成员)

这样依赖的扩展点也可以从URL拿到配置信息，所有的扩展点自己定好配置的Key后，配置信息从URL上从最外层传入。URL在配置传递上即是一条总线。
**注入的 Adaptive 实例可以提取约定 Key 来决定使用哪个实现来调用对应实现的真正方法。**Adaptive 实例的逻辑是固定，指定提取的 URL 的 Key，即可以代理真正的实现类上，可以动态生成。

在 Dubbo 的 ExtensionLoader 的扩展点类对应的 Adaptive 实现是在加载扩展点里动态生成。指定提取的 URL 的 Key 通过 @Adaptive 注解在接口方法上提供。
```java
public interface Protocol {
	@Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;
	
	@Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;
}

public interface Transporter {
    @Adaptive({"server", "transport"})
    Server bind(URL url, ChannelHandler handler) throws RemotingException;
 
    @Adaptive({"client", "transport"})
    Client connect(URL url, ChannelHandler handler) throws RemotingException;
}
```
对于 bind() 方法，Adaptive 实现先查找 server key，如果该 Key 没有值则找 transport key 值，来决定代理到哪个实际扩展点。

#### 扩展点自动激活
对于集合类扩展点，比如：Filter, InvokerListener, ExportListener, TelnetHandler, StatusChecker 等，可以同时加载多个实现，此时，可以用自动激活来简化配置，如：
```java
// 对服务方与消费方都激活；当配置了validation参数，且参数为有效值时激活，顺序10000
@Activate(group = {Constants.CONSUMER, Constants.PROVIDER}, value = Constants.VALIDATION_KEY, order = 10000)
public class ValidationFilter implements Filter {
}
```

[官网-各SPI扩展扩展点(说明/接口/配置/已有扩展/示例)](http://dubbo.apache.org/#!/docs/dev/impls/protocol.md?lang=zh-cn)