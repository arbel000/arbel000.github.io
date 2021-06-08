---
layout:     post
title:      "Java基础: JVM(八) 热加载"
date:       2019-05-26
author:     "ZhouJ000"
header-img: "img/in-post/2019/post-bg-2019-headbg.jpg"
catalog: true
tags:
    - java
    - jvm
--- 

[Java基础: JVM(一) JVM概述与字节码](https://zhouj000.github.io/2019/03/11/java-base-jvm1/)  
[Java基础: JVM(二) 常量池](https://zhouj000.github.io/2019/03/12/java-base-jvm2/)  
[Java基础: JVM(三) JVM执行引擎01](https://zhouj000.github.io/2019/03/14/java-base-jvm3/)  
[Java基础: JVM(四) Java栈帧](https://zhouj000.github.io/2019/03/18/java-base-jvm4/)  
[Java基础: JVM(五) JVM执行引擎02](https://zhouj000.github.io/2019/03/21/java-base-jvm5/)  
[Java基础: JVM(六) 类变量和类方法解析](https://zhouj000.github.io/2019/03/27/java-base-jvm6/)  
[Java基础: JVM(七) 类生命周期与类加载器](https://zhouj000.github.io/2019/03/31/java-base-jvm7/)  
[Java基础: JVM(八) 热加载](https://zhouj000.github.io/2019/05/26/java-base-jvm8/)  
[Java基础: JVM(九) Java内存模型](https://zhouj000.github.io/2019/07/09/java-base-jmm/)  
[Java基础: JVM(十) 编译相关](https://zhouj000.github.io/2019/07/11/java-base-compiler/)  



# 相关介绍

**热加载**：在运行时**重新加载(更新)class**，基于字节码的更改，使用类加载机制。在使用应用服务器时热加载，不释放内存，不重启(tomcat)，不重新打包，通常会在容器启动时起一条后台线程，定时的检测类文件的时间戳变化，如果类的时间戳发生变化，则将类重新载入。类加载器不能重新载入一个已经加载的类，但只要使用一个新的类加载器实例，就可以将类再次载入一个正在运行的应用程序

**热部署**：在应用服务器运行时直接**重新加载整个部署项目**，清空内存重新打包，重新解压war包，在于不重启服务器编译/部署项目。它比热加载更加干净，但是也比热加载更浪费时间。在基于Java的应用服务器实现热部署的过程中，类加载器也扮演着重要的角色，且大多数应用服务器都支持热部署

**热更新**：动态下发代码，一般用于APP，使用者不必重新下载全部安装包，只需下载更新包来获取更新或BUG修复

**预加载/预热**：预先加载资源放入缓存，需要时直接从缓存获取，比如用于前端图片预加载，或者像solr的index reader预热

**类的生命周期**：加载 -> 链接 -> 初始化 -> 使用 -> 卸载

**类的加载阶段**：将Java类字节码文件加载到机器内存中，并在内存中构建出Java类的原型 — **类模板对象**，即Java类在JVM内存中的一个**快照**，其中包含了从字节码文件中解析出来的常量池、类字段、类方法等信息。这样JVM在运行期就能通过类模板获取Java类中的任何信息，能对Java类成员变量进行遍历，也能调用Java类方法，这是给机器看的

**类加载器**：Bootstrap ClassLoader(引导类加载器) -> Extension ClassLoader(扩展类加载器) -> ApplicationClassLoader(系统加载器) -> UserClassLoader(自定义类加载器)

# java热加载

## demo

```java
public class HotClassLoader extends ClassLoader {
    @Override
    public Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            String fileName = name.substring(name.lastIndexOf(".") + 1) + ".class";
            InputStream is = this.getClass().getResourceAsStream(fileName);
            byte[] b = new byte[is.available()];
            is.read(b);
            return defineClass(name, b, 0, b.length);
        } catch (IOException e) {
            throw new ClassNotFoundException(name);
        }
    }
}


public static void test() throws Exception {
	loadHelloWorld();
	Thread.sleep(10000);
	System.out.println("============= new =============");
	loadHelloWorld();
}

public static void loadHelloWorld() throws Exception {
	HotClassLoader myLoader = new HotClassLoader();
	Class<?> clazz = myLoader.findClass("com.hot.Hello");
	Object obj = clazz.newInstance();
	Method method = clazz.getMethod("say");
	method.invoke(obj);
	System.out.println(obj.getClass());
	System.out.println(obj.getClass().getClassLoader());
}
```
在运行中修改Hello类的say方法，并重新编译，可以发现打印内容改变，实现热加载：
```
hello V1
class com.hot.Hello
com.hot.HotClassLoader@7229724f
============= new =============
hello V2
class com.hot.Hello
com.hot.HotClassLoader@4eec7777
```
如果用之前的classLoader加载class，那么就会取得之前的类
```
Class<?> oldClazz = oldLoader.loadClass("com.hot.Hello");
Object old = oldClazz.newInstance();
oldClazz.getMethod("say").invoke(old);
System.out.println(old.getClass().getClassLoader());


hello V1
com.hot.HotClassLoader@7229724f
```

再看另一个例子：
```java
for (int i = 0; i < 5; i++) {
	URL classUrl = new URL("file:E:\\workspace\\test\\target\\classes\\");
	URLClassLoader loader = new URLClassLoader(new URL[]{classUrl});
	Class<?> clazz = loader.loadClass("com.hot.Hello");
	Object obj = clazz.newInstance();
	clazz.getMethod("say").invoke(obj);
	System.out.println(obj.getClass().getClassLoader());

	loader.close();
	Thread.sleep(1000 * 5);
}
```
然而就算在运行中修改了Hello类，输出结果依旧没有改变
```
hello V2
sun.misc.Launcher$AppClassLoader@18b4aac2
hello V2
sun.misc.Launcher$AppClassLoader@18b4aac2
hello V2
sun.misc.Launcher$AppClassLoader@18b4aac2
hello V2
sun.misc.Launcher$AppClassLoader@18b4aac2
hello V2
sun.misc.Launcher$AppClassLoader@18b4aac2
```
显然这里使用了系统加载器AppClassLoader：
```
System.out.println(Test.class.getClassLoader().getClass().getName());
System.out.println(Test.class.getClassLoader().getSystemClassLoader());

ClassLoader cl = Test.class.getClassLoader();
while (cl != null) {
	System.out.print(cl.getClass().getName() + " -> ");
	cl = cl.getParent();
}
System.out.println(cl);


// 输出
sun.misc.Launcher$AppClassLoader
sun.misc.Launcher$AppClassLoader@18b4aac2
sun.misc.Launcher$AppClassLoader -> sun.misc.Launcher$ExtClassLoader -> null
```

## loadClass与findClass

findClass方法用于写类加载的逻辑，而loadClass方法的逻辑是为了**保证**双亲委派机制，因此：  
1、如果不想打破双亲委派模型，那么只需要**重写findClass方法**即可  
2、如果需要打破双亲委派模型，那么就要**重写整个loadClass方法**

因此可以知道，在上面的例子中加载Hello时会将其委托给父ClassLoader进行加载。而父加载器对加载的class做了**缓存**，如果发现该类已经被加载过，就不会再加载第二次了，这样类就无法更新了

而对于findClass方法，因为**同一个classLoader不能多次加载同一个类**，如果重复加载同一个类，就会抛出异常：
```
java.lang.LinkageError: loader (instance of  com/hot/HotClassLoader): attempted  duplicate class definition for name: "com/hot/Hello"
```
因此在做热加载替换class的时候，需要保证加载该class的classLoader是**新的**

在[系列上一篇中](https://zhouj000.github.io/2019/03/31/java-base-jvm7/)，已经详细介绍过了，下面简单看下源码：
```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
	synchronized (getClassLoadingLock(name)) {
		// First, check if the class has already been loaded
		// 先在当前加载器的缓存中查找有无目标类，有则直接返回
		Class<?> c = findLoadedClass(name);
		if (c == null) {
			long t0 = System.nanoTime();
			try {
				// 父加载器不为空，委托给其加载
				if (parent != null) {
					c = parent.loadClass(name, false);
				} else {
					// 父加载器为空，则让引导类加载器进行加载
					c = findBootstrapClassOrNull(name);
				}
			} catch (ClassNotFoundException e) {
				// ClassNotFoundException thrown if class not found
				// from the non-null parent class loader
			}

			if (c == null) {
				// If still not found, then invoke findClass in order
				// to find the class.
				long t1 = System.nanoTime();
				// 都没找到，使用findClass方法
				c = findClass(name);

				// this is the defining class loader; record the stats
				sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
				sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
				sun.misc.PerfCounter.getFindClasses().increment();
			}
		}
		if (resolve) {
			resolveClass(c);
		}
		return c;
	}
}
```
而如果重写loadClass方法，抹去双亲委派机制，也不能加载核心类库，因为在最终执行到defineClass方法时，会执行preDefineClass方法对核心类库包名`java.`进行**校验保护**


总结下：  
**1、重写loaderClass方法：**  
如果要想在JVM的不同类加载器中**保留具有相同全限定名的类**，那就要通过重写loadClass来实现，此时首先是通过用户自定义的类加载器来判断该类是否可加载，如果可以加载就由自定义的类加载器进行加载，如果不能够加载才交给父类加载器去加载。这种情况下，就有可能有**大量相同的类，被不同的自定义类加载器加载**到JVM中，并且这种实现方式是不符合双亲委派模型，但是比如容器插件应用场景就适合这种方式。因为我们无法保证不同的插件中不能够有相同全限定名的类存在，其他情况应当重写findClass方法  
**2、重写findClass方法：**  
是符合双亲委派模式的，它保证了相同全限定名的类是不会被重复加载到JVM中，它首先会通过父类加载器进行加载，如果所有父类加载器都无法加载，再通过用户自定义的findClass方法进行加载。需要注意的是，如果**没有实现**自定义的findClass方法，那执行到了findClass方法是会**抛出异常**的

### defineclass与resolveClass

defineClass：可以从byte[]还原出一个Class对象，将字节码转化为Class对象

resolveClass：链接指定的Java类，手动调用这个使得被加到JVM的类被链接


## AppClassLoader与ExtClassLoader

它们都定义在Launcher类中，都继承自URLClassLoader，但是它们的实现有所区别。Launcher$ExtClassLoader的并没有重写loadClass与findClass方法，而**Launcher$AppClassLoader重写的是loadClass方法，没有遵循双亲委派模型**
```java
public Class<?> loadClass(String var1, boolean var2) throws ClassNotFoundException {
	int var3 = var1.lastIndexOf(46);
	if (var3 != -1) {
		SecurityManager var4 = System.getSecurityManager();
		if (var4 != null) {
			var4.checkPackageAccess(var1.substring(0, var3));
		}
	}

	if (this.ucp.knownToNotExist(var1)) {
		Class var5 = this.findLoadedClass(var1);
		if (var5 != null) {
			if (var2) {
				this.resolveClass(var5);
			}

			return var5;
		} else {
			throw new ClassNotFoundException(var1);
		}
	} else {
		return super.loadClass(var1, var2);
	}
}
```

## classLoader卸载

JVM中没有提供class及classLoader的unload方法，那热加载、热部署、OSGI等是通过什么机制来实现的呢？其实现思路主要是通过更换classLoader进行重新加载，而之前的classLoader及加载的class类在没有实例引用的情况下，在PermGen space区域(永久区，JDK8之后为metaspace元空间)**gc**(full gc)的情况下会被回收掉。具体PermGen space区域在达到回收条件后，会对class进行引用计算，对于没有引用的class进行回收

引用下class被gc, 需满足的三个条件：  
1、该类**所有的实例**都已经被GC，即JVM中不存在改Class的任何实例  
2、该类的java.lang.Class对象没有在任何地方**被引用**，比如不能在任何地方通过反射访问该类的方法  
3、加载该类的**ClassLoader**已经被GC  

![class-unload](/img/in-post/2019/05/class-unload.png)

**存在**：  
1、每一个Class对象都引用了加载它的ClassLoader对象  
2、每一个ClassLoader对象都持有着所有由它加载的类对象  

**场景**：  
1、应用加载器打开的线程未关闭  
2、应用加载器关联的ThreadLocale未释放  
3、其他加载器引用了应用加载器的实例

**产生内存问题**：  
如果有实例类有对classLoader的引用，PermGen的class将无法卸载，导致PermGen内存一直增加，进而导致**PermGen space error**

    
参考：  
[classloader内存泄露的总结](http://www.tinygroup.org/blog/detail/5e78ace84a594e97905c2b2c44ee97f6)

对于JDK 8以后的metaspace，由于永久代难于调优，而且在启动时就已固定，因此**方法区被移到metaspace**，**字符串常量在Java堆**，类的元数据信息(metadata)就存储在metaspace的本地内存中。其中Metaspace有2块组成：  
1、**Klass Metaspace**：就是用来存klass的，这是class在JVM里的运行时数据结构，和perm一样，这块内存大小可通过`-XX:CompressedClassSpaceSize`参数来控制，默认是1G。但是这块空间也**可以没有**，假如没有开启压缩指针就不会有这块内存，这种情况下**klass都会存在NoKlass Metaspace里**，另外如果我们把`-Xmx`设置大于32G的话，其实也是没有这块内存的，因为会这么大内存会关闭压缩指针开关。还有就是这块内存最多只会存在一块  
2、**NoKlass Metaspace**：专门来存klass相关的**其他的**内容，比如method、constantPool等，这块内存是由**多块内存**组合起来的，所以可以认为是不连续的内存块组成的。这块内存是**必须的**，虽然叫做NoKlass Metaspace，但是也其实可以存klass的内容，比如上面已经提到了对应场景  
Klass Metaspace和NoKlass Mestaspace都是**所有classloader共享的**，所以类加载器们要分配内存，但是**每个类加载器都有一个SpaceManager**，来管理属于这个类加载的内存小块。如果Klass Metaspace用完了，那就会OOM了，不过一般情况下不会，NoKlass Mestaspace是由一块块内存慢慢组合起来的，在没有达到限制条件的情况下，会不断加长这条链，让它可以持续工作

metaspace特点：  
1、充分利用了Java语言规范中的好处：类及相关的元数据的生命周期与类加载器的一致  
2、**每个加载器有专门的存储空间**  
3、只进行线性分配  
4、不会单独回收某个类  
5、省掉了GC扫描及压缩的时间  
6、元空间里的对象的位置是固定的  
7、如果GC发现某个类加载器不再存活了，会把相关的空间整个回收掉  

> 类指针压缩空间：只有是64位平台上启用了类指针压缩才会存在这个区域。对于64位平台，为了压缩JVM对象中的_klass指针的大小，引入了类指针压缩空间（Compressed Class Pointer Space），其在64位平台上默认打开

元空间和类指针压缩空间的区别：  
1、类指针压缩空间只包含**类的元数据**，比如InstanceKlass, ArrayKlass。仅当打开了`UseCompressedClassPointers`选项才生效。为了提高性能，Java中的**虚方法表**也存放到这里  
2、元空间包含类的**其它比较大的元数据**，比如方法，字节码，常量池等

元空间的内存管理由**元空间虚拟机**来完成。先前，对于类的元数据我们需要不同的垃圾回收器进行处理，现在只需要执行元空间虚拟机的C++代码即可完成。在元空间中，类和其元数据的生命周期和其对应的类加载器是相同的。话句话说，**只要类加载器存活，其加载的类的元数据也是存活的**，因而不会被回收掉。准确的来说，**每一个类加载器的存储区域都称作一个元空间，所有的元空间合在一起就是我们一直说的元空间**。当一个类加载器被垃圾回收器标记为不再存活，其对应的元空间会被回收。在元空间的回收过程中没有重定位和压缩等操作。但是元空间内的元数据会进行扫描来确定Java引用

元空间虚拟机负责元空间的分配，其采用的形式为组块分配。组块的大小因类加载器的类型而异。在元空间虚拟机中存在一个**全局的空闲组块列表**。当一个类加载器需要组块时，它就会从这个全局的组块列表中获取并维持一个自己的组块列表。当一个类加载器不再存活，那么其持有的组块将会被释放，并返回给全局组块列表。类加载器持有的组块又会被分成多个块，**每一个块存储一个单元的元信息**。组块中的块是线性分配(指针碰撞分配形式)。组块分配自内存映射区域。这些全局的虚拟内存映射区域以链表形式连接，一旦某个虚拟内存映射区域清空，这部分内存就会返回给操作系统

参考：  
[Metaspace 之一：Metaspace整体介绍（永久代被替换原因、元空间特点、元空间内存查看分析方法）](https://www.cnblogs.com/duanxz/p/3520829.html)


## 补充

流程：  
在ClassLoader执行到native的defineClass0或defineClass1后，会执行本地方法，流程为[ClassLoader.c对应的defineClass方法](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/687fd7c7986d/src/share/native/java/lang/ClassLoader.c#l91)，跳转到[jvm.cpp](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/prims/jvm.cpp#l924)到[其jvm_define_class_common方法](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/prims/jvm.cpp#l862)，再跳转到[systemDictionary.cpp](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/classfile/systemDictionary.cpp#l1083)，最后跳转到[classFileParser.cpp的parseClassFile](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/classfile/classFileParser.cpp#l3653)，这个方法在前面系列中大致讲了其从魔数、常量池等的解析，然后[创建klass](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/classfile/classFileParser.cpp#l3964)...大致这个流程

load流程图：
![loadclass](/img/in-post/2019/05/loadclass.png)
find流程图(即找缓存):
![findclass](/img/in-post/2019/05/findclass.png)

扩展：  
[JVM理解（上）：classloader加载class文件的原理和机制](https://www.jianshu.com/p/52c38cf2e3d4)  



# groovy热加载

## demo

先看看groovy能做些什么，我们先执行一句脚本。然后在另一个例子中用字符串直接加载一个类：
```
public static void shell() throws Exception {
	String script = "println 'hello'; 'name = ' + name;";

	// 传入参数
	Binding binding = new Binding();
	binding.setVariable("name", "groovy");

	// 执行脚本代码
	GroovyShell shell = new GroovyShell(binding);
	Object res = shell.evaluate(script);
	System.out.println(res);
}


public static void groovyLoader() throws Exception {
	String code = "package com.testGroovy;" +
			"import com.testGroovy.HelloService;" +
			"public class TestService implements HelloService {" +
			"public String hello() {" +
			"return 'hello groovy';" +
			"}" +
			"}" +
			";";
	GroovyClassLoader groovyClassLoader = new GroovyClassLoader();
	Class<?> clazz = groovyClassLoader.parseClass(code);
	HelloService service = (HelloService) clazz.newInstance();
	System.out.println(service.hello());
}
```
java的classLoader使用loadClass加载一个class文件，而groovyClassLoader可以直接加载一个类的字符串

## classLoader

先看一下groovyClassLoader的父关系：
```
public static void classloaders() {
	def cl = MyGroovy.class.classLoader
	while (cl) {
		println cl
		cl = cl.parent
	}
}
```
groovy MyGroovy.groovy输出为
```
groovy.lang.GroovyClassLoader$InnerLoader@1bb266b3
groovy.lang.GroovyClassLoader@661972b0
org.codehaus.groovy.tools.RootLoader@60e53b93
sun.misc.Launcher$AppClassLoader@4e25154f
sun.misc.Launcher$ExtClassLoader@1ba9117e
```

我们从头开始看起，即首先在使用**groovy命令**时，其跳转到`startGroovy`命令，然后执行`GroovyStarter`，并加载传入的**GroovyMain**执行
```
// 1.
"%DIRNAME%\startGroovy.bat" "%DIRNAME%" groovy.ui.GroovyMain %*
或 
startGroovy groovy.ui.GroovyMain "$@"

// 2.
"%JAVA_EXE%" %GROOVY_OPTS% %JAVA_OPTS% -classpath "%STARTER_CLASSPATH%" %STARTER_MAIN_CLASS% --main %CLASS% --conf "%STARTER_CONF%" --classpath "%CP%" %REPLACE_PREVIEW% %CMD_LINE_ARGS%
或 
CLASS="$1"
    shift
    # Start the Profiler or the JVM
    if [ "$useprofiler" = true ] ; then
        runProfiler
    else
        # shellcheck disable=SC2086
        exec "$JAVACMD" $JAVA_OPTS \
            -classpath "$STARTER_CLASSPATH" \
            -Dscript.name="$SCRIPT_PATH" \
            -Dprogram.name="$PROGNAME" \
            -Dgroovy.starter.conf="$GROOVY_CONF" \
            -Dgroovy.home="$GROOVY_HOME" \
            -Dtools.jar="$TOOLS_JAR" \
            "$STARTER_MAIN_CLASS" \
            --main "$CLASS" \
            --conf "$GROOVY_CONF" \
            --classpath "$CP" $REPLACE_PREVIEW \
            "$@"
    fi
}

STARTER_MAIN_CLASS=org.codehaus.groovy.tools.GroovyStarter
```
进入GroovyStarter类执行main方法，其执行rootLoader方法
```java
public static void rootLoader(String args[]) {
	String conf = System.getProperty("groovy.starter.conf",null);
	final LoaderConfiguration lc = new LoaderConfiguration();
	// ...
	while (args.length-argsOffset>0 && !(hadMain && hadConf && hadCP)) {
		// ...
		} else if (args[argsOffset].equals("--main")) {
			if (hadMain) break;
			if (args.length==argsOffset+1) {
				exit("main parameter needs argument");
			}
			lc.setMainClass(args[argsOffset+1]);
			argsOffset+=2;
			hadMain=true;
		} else if (args[argsOffset].equals("--conf")) {
			if (hadConf) break;
			if (args.length==argsOffset+1) {
				exit("conf parameter needs argument");
			}
			conf=args[argsOffset+1];
			argsOffset+=2;
			hadConf=true;
		} // ...
	}
	// this allows to override the commandline conf
	String confOverride = System.getProperty("groovy.starter.conf.override",null);
	if (confOverride!=null) conf = confOverride;
	// ...
	// load configuration file
	// 解析$GROOVY_HOME/conf/groovy-starter.conf文件，里面加载了一些jar包
	lc.configure(new FileInputStream(conf));
	// ...
	// create loader and execute main class，创建的就是RootLoader
	ClassLoader loader = AccessController.doPrivileged(new PrivilegedAction<RootLoader>() {
		public RootLoader run() {
			// 负责加载Groovy及其依赖的第三方库中的类，以及管理classPathUrls(对Java原有的ClassLoader是不可见)
			return new RootLoader(lc);
		}
	});
	// ...
	Class c = loader.loadClass(lc.getMainClass());
    m = c.getMethod("main", new Class[]{String[].class});
	m.invoke(null, new Object[]{newArgs});
	// ...
}
```
然后看GroovyMain的mian方法，其跳转到processArgs方法，然后执行process方法(我们这里就是processOnce)，最终到达run方法
```java
// 省略异常处理
private boolean run() {
	if (processSockets) {
		processSockets();
	} else if (processFiles) {
		processFiles();
	} else {
		processOnce();
	}
	return true;
}
```
然而无论哪个都是使用GroovyShell来执行脚本文件的，而GroovyShell的构造方法中
```java
public GroovyShell(ClassLoader parent, Binding binding, final CompilerConfiguration config) {
	// ...
	final ClassLoader parentLoader = (parent!=null)?parent:GroovyShell.class.getClassLoader();
	this.loader = AccessController.doPrivileged(new PrivilegedAction<GroovyClassLoader>() {
		public GroovyClassLoader run() {
			return new GroovyClassLoader(parentLoader,config);
		}
	});
	// ...
}
```
由此可见，GroovyShell使用了**GroovyClassLoader**来加载类，而该GroovyClassLoader的parent即为GroovyShell的ClassLoader，也就是GroovyMain的ClassLoader，也就是RootLoader

> 这里还会设置线程上下文类加载器（Context ClassLoader），如果没有通过setContextClassLoader方法进行设置的话，线程将继承其父线程的上下文加载器，java应用运行时的初始线程的上下文类加载器是系统类加载器（这里是由Launcher类设置的）。在线程中运行的代码可以通过该类加载器来加载类和资源。比如在SPI中，父加载器可以使用当前线程上下文加载器指定的classLoader加载的类，这改变了父加载器不能使用子加载器加载的类的情况，即破坏了双亲委托模型

### RootLoader

**由于没有遵循双亲委派模型**，因此GroovyStarter并不是由AppClassLoader加载的，而是RootLoader：
```java
protected synchronized Class loadClass(final String name, boolean resolve) throws ClassNotFoundException {
	Class c = this.findLoadedClass(name);
	if (c != null) return c;
	// customClasses定义了一些必须由Java原有ClassLoader载入的类  
	c = customClasses.get(name);
	if (c != null) return c;

	try {
		// 先尝试加载这个类  
		c = oldFindClass(name);
	} catch (ClassNotFoundException cnfe) {
		// IGNORE
	}
	// 加载不到则回到原有的双亲委派模型  
	if (c == null) c = super.loadClass(name, resolve);

	if (resolve) resolveClass(c);

	return c;
}
```
所以即使parent已经载入了GroovyStarter，RootLoader还会**再加载一次**。这是因为Java的classpath中只包含了Groovy的jar包，而不包含Groovy依赖的第三方jar包，而Groovy的**classpath**则包含了Groovy以及其**依赖的**所有第三方jar包。如果RootLoader使用双亲委派模型，那么Groovy的jar包中的类就会由System ClassLoader加载，当解析Groovy的类时，需要加载第三方的jar包，这时System ClassLoader并不知道从哪里加载，导致找不到类。因此RootLoader并没有使用双亲委派模型

> 那为什么不把这些jar包都加入Java的classpath中？这样不就不会有这个问题了吗？确实如此，但是Groovy可以通过多种方式更灵活的往自己的classpath中添加路径（你甚至可以通过代码往RootLoader的classpath中添加路径），而Java的classpath只能通过命令行添加(或使用Instrumentation)，因此就有了RootLoader这样的设计

### GroovyClassLoader 

GroovyClassLoader主要负责在**运行时编译groovy源代码为Class**的工作，从而使Groovy实现了将groovy源代码动态加载为Class的功能

从groovyClassLoader.parseClass方法一直下去，会进入doParseClass方法，其编译groovy代码的工作重要集中这里：
```java
private Class doParseClass(GroovyCodeSource codeSource) {
	// 简单校验一些参数是否为null
	validate(codeSource);
	Class answer;  // Was neither already loaded nor compiling, so compile and add to cache.
	// 创建CompilationUnit，构造方法中会初始化1个加载类的加载器(this)和1个AST transformations的加载器(null)
	CompilationUnit unit = createCompilationUnit(config, codeSource.getCodeSource());
	if (recompile!=null && recompile || recompile==null && config.getRecompileGroovySource()) {
		unit.addFirstPhaseOperation(TimestampAdder.INSTANCE, CompilePhase.CLASS_GENERATION.getPhaseNumber());
	}
	SourceUnit su = null;
	File file = codeSource.getFile();
	if (file != null) {
		su = unit.addSource(file);
	} else {
		URL url = codeSource.getURL();
		if (url != null) {
			su = unit.addSource(url);
		} else {
			su = unit.addSource(codeSource.getName(), codeSource.getScriptText());
		}
	}
	
	// 这里创建了InnerLoader 
	ClassCollector collector = createCollector(unit, su);
	unit.setClassgenCallback(collector);
	int goalPhase = Phases.CLASS_GENERATION;
	if (config != null && config.getTargetDirectory() != null) goalPhase = Phases.OUTPUT;
	// 编译groovy源代码  
	unit.compile(goalPhase);

	answer = collector.generatedClass;
	// 查找源文件中的Main Class 
	String mainClass = su.getAST().getMainClassName();
	for (Object o : collector.getLoadedClasses()) {
		Class clazz = (Class) o;
		String clazzName = clazz.getName();
		definePackageInternal(clazzName);
		setClassCacheEntry(clazz);
		if (clazzName.equals(mainClass)) answer = clazz;
	}
	return answer;
}
```

GroovyClassLoader也重写了loadClass方法，但是**符合双亲委派机制的**：
```java
public Class loadClass(final String name, boolean lookupScriptFiles, boolean preferClassOverScript, boolean resolve)
		throws ClassNotFoundException, CompilationFailedException {
	// look into cache
	Class cls = getClassCacheEntry(name);

	// enable recompilation?
	boolean recompile = isRecompilable(cls);
	if (!recompile) return cls;

	// try parent loader
	ClassNotFoundException last = null;
	try {
		// 委托父类加载
		Class parentClassLoaderClass = super.loadClass(name, resolve);
		// always return if the parent loader was successful
		if (cls != parentClassLoaderClass) return parentClassLoaderClass;
	} catch (ClassNotFoundException cnfe) {
		last = cnfe;
	} catch (NoClassDefFoundError ncdfe) {
		if (ncdfe.getMessage().indexOf("wrong name") > 0) {
			last = new ClassNotFoundException(name);
		} else {
			throw ncdfe;
		}
	}

	// check security manager
	SecurityManager sm = System.getSecurityManager();
	if (sm != null) {
		String className = name.replace('/', '.');
		int i = className.lastIndexOf('.');
		// no checks on the sun.reflect classes for reflection speed-up
		// in particular ConstructorAccessorImpl, MethodAccessorImpl, FieldAccessorImpl and SerializationConstructorAccessorImpl
		// which are generated at runtime by the JDK
		if (i != -1 && !className.startsWith("sun.reflect.")) {
			sm.checkPackageAccess(className.substring(0, i));
		}
	}

	// prefer class if no recompilation
	if (cls != null && preferClassOverScript) return cls;

	// at this point the loading from a parent loader failed
	// and we want to recompile if needed.
	if (lookupScriptFiles) {
		// try groovy file
		try {
			// check if recompilation already happened.
			final Class classCacheEntry = getClassCacheEntry(name);
			if (classCacheEntry != cls) return classCacheEntry;
			URL source = resourceLoader.loadGroovySource(name);
			// if recompilation fails, we want cls==null
			Class oldClass = cls;
			cls = null;
			cls = recompile(source, name, oldClass);
		} catch (IOException ioe) {
			last = new ClassNotFoundException("IOException while opening groovy source: " + name, ioe);
		} finally {
			if (cls == null) {
				removeClassCacheEntry(name);
			} else {
				setClassCacheEntry(cls);
			}
		}
	}

	if (cls == null) {
		// no class found, there should have been an exception before now
		if (last == null) throw new AssertionError(true);
		throw last;
	}
	return cls;
}
```


### GroovyClassLoader.InnerLoader 

```java
protected ClassCollector createCollector(CompilationUnit unit, SourceUnit su) {
	InnerLoader loader = AccessController.doPrivileged(new PrivilegedAction<InnerLoader>() {
		public InnerLoader run() {
			// 每次编译groovy源代码都会创建一个新的InnerLoader
			return new InnerLoader(GroovyClassLoader.this);
		}
	});
	return new ClassCollector(loader, unit, su);
}


// 在编译的过程中，将编译出来的字节码，通过InnerLoader进行加载
public static class ClassCollector extends CompilationUnit.ClassgenCallback {
	// ...
	protected Class createClass(byte[] code, ClassNode classNode) {
		BytecodeProcessor bytecodePostprocessor = unit.getConfiguration().getBytecodePostprocessor();
		byte[] fcode = code;
		if (bytecodePostprocessor!=null) {
			fcode = bytecodePostprocessor.processBytecode(classNode.getName(), fcode);
		}
		GroovyClassLoader cl = getDefiningClassLoader();
		Class theClass = cl.defineClass(classNode.getName(), fcode, 0, fcode.length, unit.getAST().getCodeSource());
		this.loadedClasses.add(theClass);

		if (generatedClass == null) {
			ModuleNode mn = classNode.getModule();
			SourceUnit msu = null;
			if (mn != null) msu = mn.getContext();
			ClassNode main = null;
			if (mn != null) main = (ClassNode) mn.getClasses().get(0);
			if (msu == su && main == classNode) generatedClass = theClass;
		}

		return theClass;
	}
	// ...
}
```

有了GroovyClassLoader，还需要InnerLoader主要有两个原因：  
1、由于一个ClassLoader对于**同一个名字**的类只能加载一次，如果都由GroovyClassLoader加载，那么当一个脚本里定义了C这个类之后，另外一个脚本再定义一个C类的话，GroovyClassLoader就无法加载了  
2、由于当一个类的ClassLoader被GC之后，这个类才能被GC，如果由GroovyClassLoader加载所有的类，那么只有当GroovyClassLoader被GC了，所有这些类才能被GC，而如果用InnerLoader的话，由于**编译完源代码之后，已经没有对它的外部引用，除了它加载的类**，所以只要**它加载的类没有被引用**之后，它以及它加载的类就都可以被GC了


### compile

简单看一下：
```java
// CompilationUnit
// throughPhase：Phases.CLASS_GENERATION 7，即从步骤1做到步骤7
// 		INITIALIZATION        = 1;   // Opening of files and such
// 		PARSING               = 2;   // Lexing, parsing, and AST building
// 		CONVERSION            = 3;   // CST to AST conversion
// 		SEMANTIC_ANALYSIS     = 4;   // AST semantic analysis and elucidation
// 		CANONICALIZATION      = 5;   // AST completion
// 		INSTRUCTION_SELECTION = 6;   // Class generation, phase 1
// 		CLASS_GENERATION      = 7;   // Class generation, phase 2
// 		OUTPUT                = 8;   // Output of class to disk
// 		FINALIZATION          = 9;   // Cleanup
//		ALL                   = 9;   // Synonym for full compilation
public void compile(int throughPhase) throws CompilationFailedException {
	// 保证不处理旧代码
	gotoPhase(Phases.INITIALIZATION);
	throughPhase = Math.min(throughPhase, Phases.ALL);

    while (throughPhase >= phase && phase <= Phases.ALL) {
		// 步骤4
		if (phase == Phases.SEMANTIC_ANALYSIS) {
			doPhaseOperation(resolve);
			if (dequeued()) continue;
		}
		// ---> 最后的步骤7有2个操作：CompilationUnit和ASTTransformationVisitor
		processPhaseOperations(phase);
		// Grab processing may have brought in new AST transforms into various phases, process them as well
		processNewPhaseOperations(phase);
		// ...
	}
	// ...
}
	
	
// body：CompilationUnit
// 在AST抽象语法树上循环操作所有主要的ClassNodes
public void applyToPrimaryClassNodes(PrimaryClassNodeOperation body) throws CompilationFailedException {
	for (ClassNode classNode : getPrimaryClassNodes(body.needSortedInput())) {
		SourceUnit context = null;
		context = classNode.getModule().getContext();
		if (context == null || context.phase < phase || (context.phase == phase && !context.phaseComplete)) {
			int offset = 1;
			for (Iterator<InnerClassNode> iterator = classNode.getInnerClasses(); iterator.hasNext(); ) {
				iterator.next();
				offset++;
			}
			// 
			body.call(context, new GeneratorContext(this.ast, offset), classNode);
		}
	}
}

private PrimaryClassNodeOperation classgen = new PrimaryClassNodeOperation() {
	// ...
	public void call(SourceUnit source, GeneratorContext context, ClassNode classNode) throws CompilationFailedException {
		// ...
		// Run the Verifier on the outer class
		verifier.visitClass(classNode);
		// ...
		// Prep the generator machinery
		// 返回一个groovyjarjarasm.asm.ClassWriter
		ClassVisitor visitor = createClassVisitor();
		// ...
		AsmClassGenerator generator = new AsmClassGenerator(source, context, visitor, sourceName);
		// Run the generation and create the class (if required)
		// 
		generator.visitClass(classNode);
		byte[] bytes = ((ClassWriter) visitor).toByteArray();
		generatedClasses.add(new GroovyClass(classNode.getName(), bytes));
		// Handle any callback that's been set
		if (CompilationUnit.this.classgenCallback != null) {
			classgenCallback.call(visitor, classNode);
		}
		// Recurse for inner classes
		LinkedList innerClasses = generator.getInnerClasses();
		while (!innerClasses.isEmpty()) {
			classgen.call(source, context, (ClassNode) innerClasses.removeFirst());
		}
	}
}		
```



# asm

ASM框架是一个致力于**字节码操作和分析**的框架，它可以用来**修改**一个已存在的类或者**动态产生**一个新的类。ASM提供了一些通用的字节码转换和分析算法，通过这些算法可以定制更复杂的工具。ASM提供了其它字节码工具相同的功能，但是它更关注执行效率，它被设计的更小更快，它被用于以下项目：  
1、openjdk，实现lambda表达式调用，Nashorn编译器  
2、Groovy和Kotlin编译器  
3、Cobertura 和Jacoco，测量代码范围  
4、CGLIB动态代理类  
与BCEL和SERL不同，ASM提供了更为现代的编程模型。对于ASM来说，**Java class被描述为一棵树，使用"Visitor"模式遍历整个二进制结构**，其事件驱动的处理方式使得用户只需要关注于对其编程有意义的部分，而不必了解Java类文件格式的所有细节。ASM框架提供了默认的"response taker"处理这一切


## 其他方式比较

最直接的改造Java类的方法莫过于**直接改写class文件**。Java规范详细说明了class文件的格式，直接编辑字节码确实可以改变Java类的行为，然而需要使用者对Java class文件的格式了熟于心：小心地推算出想改造的函数相对文件首部的偏移量，同时重新计算class文件的校验码以通过Java虚拟机的安全机制

Java 5开始提供的Instrument包也可以提供类似的功能：启动时往Java虚拟机中挂上一个用户定义的hook程序，可以在装入特定类的时候改变特定类的字节码，从而改变该类的行为。但是其缺点也是明显的：  
1、Instrument包是在整个虚拟机上挂了一个钩子程序，**每次装入一个新类的时候，都必须执行一遍这段程序**，即使这个类不需要改变  
2、直接改变字节码事实上类似于直接改写class文件，无论是调用ClassFileTransformer. transform方法还是Instrument.redefineClasses方法，都必须提供新Java类的字节码。也就是说同直接改写class文件一样  
因此尽管Instrument可以改造类，但事实上Instrument**更适用于监控和控制虚拟机的行为**，其Proxy编程是面向接口的，新的类是接口另一个实现，而且通过反射实现，在效率上付出了代价  
* JDK 6以后可以在JVM运行后使用Java Tool API中的attach方式，动态设置代理类，达到instrumentation的目的

ASM能够通过改造既有类，直接生成需要的代码。**增强的代码是硬编码在新生成的类文件内部的**，没有反射带来性能上的付出。同时ASM与Proxy编程不同，不需要为增强代码而新定义一个接口，生成的代码可以覆盖原来的类，或者是原始类的子类。它是一个普通的 Java 类而不是proxy类，甚至可以在应用程序的类框架中拥有自己的位置，派生自己的子类

## demo

ASM通过**树**这种数据结构来表示复杂的字节码结构，并利用**Push模型**来对树进行遍历，在遍历过程中对字节码进行修改。Push模型类似于**Visitor设计模式**，Visitor相当于用户派出的代表深入到算法内部，由算法安排访问行程，Visitor可以更换，对算法流程无法干预，与Iterator模式主动调遣算法有所不同

**ClassReader**：该类用来**解析**编译过的class字节码文件  
**ClassWriter**:：类用来**重新构建**编译后的类，比如说修改类名、属性以及方法，甚至可以生成新的类的字节码文件  
**ClassAdapter**：该类**实现了ClassVisitor接口**，它将对它的方法调用委托给另一个ClassVisitor对象
```java
public class ASMGettingStarted {
    /**
     * 动态创建一个类，有一个无参数的构造函数
     */
    static ClassWriter createClassWriter(String className) {
        ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_MAXS);
        //声明一个类，使用JDK1.8版本，public的类，父类是java.lang.Object，没有实现任何接口
        cw.visit(Opcodes.V1_8, Opcodes.ACC_PUBLIC, className, null, "java/lang/Object", null);

        //初始化一个无参的构造函数
        MethodVisitor constructor = cw.visitMethod(Opcodes.ACC_PUBLIC, "<init>", "()V", null, null);
        constructor.visitVarInsn(Opcodes.ALOAD, 0);
        //执行父类的init初始化
        constructor.visitMethodInsn(Opcodes.INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false);
        //从当前方法返回void
        constructor.visitInsn(Opcodes.RETURN);
        constructor.visitMaxs(1, 1);
        constructor.visitEnd();
        return cw;
    }

    /**
     * 创建一个run方法，里面只有一个输出
     * public void run()
     * {
     * System.out.println(message);
     * }
     *
     * @return
     * @throws Exception
     */
    static byte[] createVoidMethod(String className, String message) throws Exception {
        //注意，这里需要把classname里面的.改成/，如com.asm.Test改成com/asm/Test
        ClassWriter cw = createClassWriter(className.replace('.', '/'));

        //创建run方法
        //()V表示函数，无参数，无返回值
        MethodVisitor runMethod = cw.visitMethod(Opcodes.ACC_PUBLIC, "run", "()V", null, null);
        //先获取一个java.io.PrintStream对象
        runMethod.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
        //将int, float或String型常量值从常量池中推送至栈顶  (此处将message字符串从常量池中推送至栈顶[输出的内容])
        runMethod.visitLdcInsn(message);
        //执行println方法（执行的是参数为字符串，无返回值的println函数）
        runMethod.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
        runMethod.visitInsn(Opcodes.RETURN);
        runMethod.visitMaxs(1, 1);
        runMethod.visitEnd();

        return cw.toByteArray();
    }

    public static void main(String[] args) throws Exception {
        String className = "com.testAsm.MyAsmTest";
        byte[] classData = createVoidMethod(className, "hello ASM");
        Class<?> clazz = new GeneratorClassLoader().defineClassForName(className, classData);
//        clazz.getMethods()[0].invoke(clazz.newInstance());
        Object obj = clazz.newInstance();
        clazz.getMethod("run").invoke(obj);
        System.out.println(obj.getClass().getName());
    }

    private static class GeneratorClassLoader extends ClassLoader {

        public Class defineClassForName(String className, byte[] classFile) throws ClassFormatError {
            return defineClass(className, classFile, 0, classFile.length);
        }
    }
}
```
在ASM中，提供了一个ClassReader类，其可以**通过字节数组或class文件间接的获得字节码数据**，然后分析字节码，**构建出抽象的语法树**在内存中表示字节码。它调用accept方法，接受一个实现了**ClassVisitor**接口的实例，来依次调用ClassVisitor接口的各个方法。字节码**空间上的偏移**被转换成visit事件时间上调用的先后，visit事件就是对不同**visit函数**的调用，由ClassReader决定如何调用，我们只需要提供不同的Visitor来对字节码树进行不同的修改。ClassVisitor会产生一些子过程，比如visitMethod会返回一个实现MethordVisitor接口的实例，子过程完成或返回父过程，然后继续访问下一个节点，当然我们也可以不通过ClassReader类，只不过要严格遵守一定的访问顺序即可

各个ClassVisitor通过**职责链模式**，可以非常简单的封装对字节码的各种修改，而无须关注字节码的字节偏移。**ClassAdaptor类**实现了 ClassVisitor接口所定义的所有函数，当新建一个ClassAdaptor对象时，传入一个实现ClassVisitor接口的对象，作为职责链中的下一个访问者(Visitor)，当需要对字节码进行调整时，只需从ClassAdaptor类派生出一个子类，覆写需要修改的方法，完成相应功能后再把调用传递下去

ASM的最终的目的是生成可以被正常装载的class文件，因此其框架结构为我们提供了一个**生成字节码的工具类 —— ClassWriter**。它实现了**ClassVisitor接口**，而且含有一个toByteArray()函数，返回**生成的字节码的字节流**，将字节流写回文件即可生产调整后的class文件，一般它都作为职责链的终点


创建一个接口：
```java
public static void main(String[] args) throws IOException {
	ClassWriter cw = new ClassWriter(0);

	//通过visit方法确定类的头部信息
	cw.visit(Opcodes.V1_5, Opcodes.ACC_PUBLIC + Opcodes.ACC_ABSTRACT + Opcodes.ACC_INTERFACE,
			"com/testAsm/MyAsmInterface", null, "java/lang/Object", new String[]{"com/testAsm/a/HelloAsm"});

	//定义类的属性
	cw.visitField(Opcodes.ACC_PUBLIC + Opcodes.ACC_FINAL + Opcodes.ACC_STATIC,
			"LESS", "I", null, new Integer(-1)).visitEnd();
	cw.visitField(Opcodes.ACC_PUBLIC + Opcodes.ACC_FINAL + Opcodes.ACC_STATIC,
			"EQUAL", "I", null, new Integer(0)).visitEnd();
	cw.visitField(Opcodes.ACC_PUBLIC + Opcodes.ACC_FINAL + Opcodes.ACC_STATIC,
			"GREATER", "I", null, new Integer(1)).visitEnd();

	//定义类的方法
	cw.visitMethod(Opcodes.ACC_PUBLIC + Opcodes.ACC_ABSTRACT, "myMethod",
			"(Ljava/lang/Object;)I", null, null).visitEnd();
	cw.visitEnd(); //使cw类已经完成

	//将cw转换成字节数组写到文件里面去
	byte[] data = cw.toByteArray();
	File file = new File("E:\\workspace\\test\\target\\classes\\com\\testAsm\\a\\MyAsmInterface.class");
	FileOutputStream fout = new FileOutputStream(file);
	fout.write(data);
	fout.close();
}
```
使用**反编译工具**查看：
```java 
package com.testAsm;

import com.testAsm.a.HelloAsm;

public abstract interface MyAsmInterface
  extends HelloAsm
{
  public static final int LESS = -1;
  public static final int EQUAL = 0;
  public static final int GREATER = 1;
  
  public abstract int myMethod(Object paramObject);
}
```
使用`javap -verbose`查看：
```
public interface com.testAsm.MyAsmInterface extends com.testAsm.a.HelloAsm
  minor version: 0
  major version: 49
  flags: ACC_PUBLIC, ACC_INTERFACE, ACC_ABSTRACT
Constant pool:
   #1 = Utf8               com/testAsm/MyAsmInterface
   #2 = Class              #1             // com/testAsm/MyAsmInterface
   #3 = Utf8               java/lang/Object
   #4 = Class              #3             // java/lang/Object
   #5 = Utf8               com/testAsm/a/HelloAsm
   #6 = Class              #5             // com/testAsm/a/HelloAsm
   #7 = Utf8               LESS
   #8 = Utf8               I
   #9 = Integer            -1
  #10 = Utf8               EQUAL
  #11 = Integer            0
  #12 = Utf8               GREATER
  #13 = Integer            1
  #14 = Utf8               myMethod
  #15 = Utf8               (Ljava/lang/Object;)I
  #16 = Utf8               ConstantValue
{
  public static final int LESS;
    descriptor: I
    flags: ACC_PUBLIC, ACC_STATIC, ACC_FINAL
    ConstantValue: int -1

  public static final int EQUAL;
    descriptor: I
    flags: ACC_PUBLIC, ACC_STATIC, ACC_FINAL
    ConstantValue: int 0

  public static final int GREATER;
    descriptor: I
    flags: ACC_PUBLIC, ACC_STATIC, ACC_FINAL
    ConstantValue: int 1

  public abstract int myMethod(java.lang.Object);
    descriptor: (Ljava/lang/Object;)I
    flags: ACC_PUBLIC, ACC_ABSTRACT
```



# java Instrumentation

对于操作字节码，ASM、CGlib、Java Proxy、Javassist都可以，不过要等到需要被操作的类被加载了才行，而Java提供了一个可行的机制，用来在ClassLoader加载字节码之前完成对操作字节码的目的。`java.lang.instrument.Instrumentation`类为提供直接操作Java字节码的一个途径，虽然本是用来检测Java代码的，但本质上实现对代码检测的途径就是直接修改字节码，有两种方法可以达到目的：  
1、当JVM以指示一个代理类的方式**启动时**，将传递给代理类的`premain方法`一个Instrumentation实例  
2、当JVM提供某种机制在JVM**启动之后**某一时刻启动代理时，将传递给代理代码的`agentmain方法`一个Instrumentation实例  
Instrumentation使得可以构建一个独立于应用程序的**代理程序**(Agent)，用来监测和协助运行在JVM上的程序，甚至能够替换和修改某些类的定义，这个特性实际上是一种**虚拟机级别支持的AOP实现方式**

利用`java.lang.instrument`做静态Instrumentation是**Java 5**的新特性，它把Java的instrument功能从本地代码中解放出来，使之可以用Java代码的方式解决问题。而在**Java 6**里面，instrumentation包被赋予了更强大的功能：启动后的instrument、本地代码(native code)instrument，以及动态改变classpath等等

参考：  
[基于Java Instrument的Agent实现](https://www.jianshu.com/p/b72f66da679f)  
[Java Instrumentation](https://www.cnblogs.com/yelao/p/9841810.html)

扩展：  
[CGlib Enhancer 主流程源码解析](https://www.ffutop.com/2018-07-10-CGlib-Enhancer/)  
[Java Proxy 源码解析](https://www.ffutop.com/2018-07-20-Java-Proxy/)

