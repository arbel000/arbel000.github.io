---
layout:     post
title:      "Java基础: JVM(六) 类变量和类方法解析"
date:       2019-03-27
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



# 类变量解析

在`ClassFileParser::parseClassFile()`函数的解析步骤中，常量池解析后进行父类和接口解析，接着就是调用`parse_fields()`函数解析类变量信息：
[classFileParser.cpp](http://hg.openjdk.java.net/jdk6/jdk6/hotspot/file/c6ff3162647f/src/share/vm/classfile/classFileParser.cpp#l2527)
```
void ClassFileParser::java_lang_Class_fix_pre(objArrayHandle* methods_ptr, 
  FieldAllocationCount *fac_ptr, TRAPS) {
    // ...
    // Fields (offsets are filled in later)
    struct FieldAllocationCount fac = {0,0,0,0,0,0,0,0,0,0};
    objArrayHandle fields_annotations;
    typeArrayHandle fields = parse_fields(cp, access_flags.is_interface(), &fac, &fields_annotations, CHECK_(nullHandle));
    // ...
```
在调用parse_fields前定义了一个变量fac，其结构是：
```
struct FieldAllocationCount {
  int static_oop_count;
  int static_byte_count;
  int static_short_count;
  int static_word_count;
  int static_double_count;
  int nonstatic_oop_count;
  int nonstatic_byte_count;
  int nonstatic_short_count;
  int nonstatic_word_count;
  int nonstatic_double_count;
};
```
可以得知记录了5种静态类型和5种非静态类型，分别是**Oop引用类型、Byte字节类型、Short短整型、Word双字类型、Double浮点型**。在`parse_fields()`函数中会分别统计static和非static的这5种变量的数量，后面JVM为Java类分配内存空间时，会**根据这些变量的数量计算占用内存大小**，在JVM内部，除了引用类型，所有内置的基本类型都是由剩余的4种类型表示的，因此一个Java类的域变量要占多少内存，只要分别统计这几类型的数量就可以了

在[ClassFileParser::parse_fields()](http://hg.openjdk.java.net/jdk6/jdk6/hotspot/file/c6ff3162647f/src/share/vm/classfile/classFileParser.cpp#l667)函数中的主要逻辑为：  
1、读取Java类中的变量数量  
2、读取变量的访问标识  
3、读取变量名称索引  
4、读取变量类型索引  
5、读取变量属性  
6、判断变量类型  
7、统计各类型数量

在Java类对应的字节码文件中，有专门的一块区域保存Java类的变量信息，字节码文件中依次描述各个变量的**访问标识、名称索引、类型索引和属性**，这些都占用2个字节(u2)，因此会依次调用`cfs->get_u2_fast()`并解析，解析完后通过`fields->short_at_put(...)`保存到fields所代表的内存区域中，而**fields是在方法一开始就申请的内存**：
```
u2 length = cfs->get_u2_fast();
// Tuples of shorts [access, name index, sig index, initial value index, byte offset, generic signature index]
typeArrayOop new_fields = oopFactory::new_permanent_shortArray(length*instanceKlass::next_offset, CHECK_(nullHandle));
typeArrayHandle fields(THREAD, new_fields);
```
其最终分配的大小由`length*instanceKlass::next_offset`决定，其中length是Java类中的变量数量，而后面的是一个[枚举](http://hg.openjdk.java.net/jdk6/jdk6/hotspot/file/c6ff3162647f/src/share/vm/oops/instanceKlass.hpp#l267)，其对应值为7，因此最终申请的内存大小是`7 * length`，而由于length类型是u2，因此如果一个Java类中有5个变量，则JVM会申请`14 * 5 = 70`字节的内存空间来存放变量的属性

Java类变量在JVM内存中的映像和字节码文件对其的维度描述并不相同，字节码文件仅仅描述了标量的访问标识、名称索引、类型索引、属性，而JVM内存除此之外**还描述了每一个变量的偏移量和泛型索引**。在遍历解析各个变量属性的循环后面，JVM会取出每一个变量类型进行判断，并根据类型来统计5种类型(static和非static各5种)的总数量

## 偏移量

对于静态类型变量，会先拿到**起始偏移量**，然后分别根据**各个静态类型变量**的偏移量[计算总偏移量](http://hg.openjdk.java.net/jdk6/jdk6/hotspot/file/c6ff3162647f/src/share/vm/classfile/classFileParser.cpp#l2639)，计算顺序依次是static_oop、static_double、static_word、static_short、static_byte，每一种下游数据的偏移量都依赖其上游数据类型所占的字节宽度和数量。对于JDK 1.6而言，静态变量存储在Java类在JVM中对应的**镜像类Mirror**中，当Java代码访问静态变量时，最终**JVM会通过设置偏移量进行访问**

对于非静态类型变量，相比复杂许多，主要包含[5个步骤](http://hg.openjdk.java.net/jdk6/jdk6/hotspot/file/c6ff3162647f/src/share/vm/classfile/classFileParser.cpp#l2661)：  
1、计算非静态变量**起始偏移量**  
2、计算nonstatic_double_offset和nonstatic_oop_offset这2种非静态类型变量的起始偏移量(分配策略不同，起始偏移量不同)  
3、计算剩余3种类型变量的起始偏移量  
4、计算**对齐补白**空间  
5、计算补白后非静态变量所需的内存空间**总大小**  
对于一个给定的Java类，其主要的内存便是其**非静态类型的变量**所占的空间，另一部分是**虚拟方法分发表**

### 非静态变量起始偏移量

要计算非静态变量起始偏移量，先需要知道从哪里开始偏移。对于非静态类型变量，其偏移量是**相对于**未来即将new出来的Java对象实例在JVM内部所对应的**instanceOop对象实例首地址**的偏移位置

前面的常量池中讲过，JVM内部使用oop-klass这种一分为二的模型去描述一个对象，常量池本身也是这个模型，当然对于Java类在JVM内部也是一样的一分为二的模型描述，对应的oop类是**instanceOopDesc**，对应klass是**instanceKlass**。在oop-klass模型中，oop模型主要存储**对象实例的实际数据**，而klass模型主要存储**对象的结构信息和虚函数方法表(即描述类的结构和行为)**。当JVM**加载**一个Java类时，会首先创建对应的**instanceKlass对象**，而当**new**一个Java对象实例时，则会构建出对应的**instanceOop对象**，其主要由Java类的成员变量组成，而这些成员变量在instanceOop中的排序顺序，便是由各种变量类型的偏移量决定

在Hotspot内部，任何oop对象都包含对象头，因此实际上非静态变量的偏移量要从**对象头的末尾**开始计算。如果没有启用压缩策略，最终返回instanceOopDesc类型大小，即两个指针宽度，在64位平台下就是16字节；如果开启了压缩策略，oop._mark作为指针是不会压缩的，而_klass会仅占用4字节，因此对象头只会占用12字节。除外还会获取**父类非静态字段大小**，由于Java类是面向对象的，因此除了Object外，所有Java类都显式或隐式继承了某个父类，而**字段继承和方法继承**构成了继承的全部内涵。如果说继承是目的，那么字段在子类中的内存占用就是技术手段，子类必须将父类中所定义的非静态字段信息**全部复制一遍**，才能实现字段继承的目标。因此在计算子类非静态字段的起始偏移量的时候，必须将父类可被继承的字段的内存大小考虑在内，因此子类的非静态字段起始偏移量，在计算完oop对象头的大小后，还要为父类的可继承字段预留空间

### 内存对齐与字段重排

得出Java类字段的起始偏移量后，接下来就能基于这个起始偏移量计算出Java类中所有字段的偏移量。然而Java字段的偏移地址与内存对齐有莫大关系，JVM为了处理内存对齐也是颇费一番心思，甚至不惜将字段进行重排

内存对齐与数据在内存中的位置有关，**如果一个变量的内存起始位置正好等于其长度的整数倍，则这种内存分配就被称为自然对齐**，比如在32位平台下的int整形变量的内存地址为0x00000008，那么该变量就是自然对齐的。现代计算机中内存空间都是按照字节划分的，即内存的粒度细分到存储单元，而一个存储单元恰恰包含8个比特，正好1字节(byte)，因此理论上似乎对于任何类型的变量的访问都可以从任何地址开始。然而事实上各个硬件平台在对存储空间的处理上有很大不同，一些平台上对某些特定类型数据只能从某些特定地址开始存取，如果访问一个没有经过对齐的变量就会发生错误；而有些平台虽然支持非对齐内存位置的数据读写，但是会在存取效率上有所损失。因此由于以上两个原因，一种是程序健壮性，一种是高性能，均需要各种类型数据按照一定规则在空间上排列，因此要对数据进行对齐

无论是C/C++这样的编译型语言，还是Java/C#这类的高级语言，一般情况下都不需要开发者考虑对齐的问题，因为编译器或虚拟机会自动将数据进行对齐补白，例如：
```
#include<stdio.h>
struct A{
    int a;
    char b;
    short c;
};
struct B{
    char b;
    int a;
    short c;
};
int main() {
    int a = sizeof(struct A);
    int b = sizeof(struct B);
    printf("%d, %d", a, b);
}
```
声明了2个结构体类型A和B，它们的差别是变量的声明顺序，在32位机器上它们的长度为：char(1)、short(2)、int(4)，那么如果没有对齐它们占的内存大小应该是7，然而实际结果为A占用8，B占用12：
![padding](/img/in-post/2019/03/padding.png)

从这个例子可以看到，像C/C++这种接近底层的编程语言，开发中不需要关注内存对齐问题，因为编译器会完成一切，然而也显露出一个问题，就是编译器并不会使用除内存对齐以外的其他优化技巧，由于成员项的声明顺序不同就导致了两者在**空间利用率**上的巨大差异。如果有更多的成员变量，并且声明式无序的，那么内存利用率也会得不到保证。所以在C/C++开发时，需要关注内存对齐的问题，在定义结构体类型的成员项时，按类型大小由小到大依次声明，可以尽量减少中间的填补空间。当然也可以显示声明填补空间：`char reserved`来声明1个补白字节，`char reserved[3]`声明3个字节补白空间

Java的字段最终也是保存在堆栈中的，而且Java类中变量类型会被映射为CPU硬件平台支持的数据类型，因此Java字段在内存中的位置也**必须满足自然对齐**的要求，尤其Java作为跨平台的编程语言，更要支持各种异构平台对数据对齐的要求。JVM在设计上使内部5大类型数据都满足内存对齐的原则：

|      数据类型       | 占用(byte) | JVM内部数据类型 | 
| :------------------ | :--------- | :-------------- |
| boolean             | 1          | byte            |
| byte                | 1          | byte            |
| char                | 2          | short           |
| short               | 2          | short           |
| int                 | 4          | word            |
| float               | 4          | word            |
| long                | 8          | double          |
| double              | 8          | double          |
| 引用类型(reference) | 4/8        | oop             |

JVM必须确保这5类基本数据类型都能自然对齐，即确保**其内存地址能被其所占用的字节宽度所整除**，而解决自然对齐的手段无非就是内存补白，而JVM不但使用了**补白**，还使用了**字段重排**。JVM的字段重排策略主要为：  
1、将相同类型的字段组合在一起  
2、按照double->word->short->byte->oop的顺序依次分配  
让相同类型字段组合在一起很好理解，但是为什么JVM按这个顺序分配？这样重排效果怎么样？看一下这个例子：
```
class A{
    byte b;
    long l;
    byte b2;
    int i;
}
```
![sort](/img/in-post/2019/03/sort.png)

### 计算变量偏移量

那么知道了Hotspot对字段内存的分配策略，会将字段进行重排，不难看出只要先计算出内部5种类型字段的起始偏移量，然后根据每一类型有多少个Java类字段，就可以得出每一个具体的字段的偏移量。因此Hotspot中先拿到Java类字段的起始偏移量，然后解析Java类常量池得到各种字段的数量，根据不同的顺序策略计算oop或long的起始偏移量，这里Hotspot提供了[2种重排策略](http://hg.openjdk.java.net/jdk6/jdk6/hotspot/file/c6ff3162647f/src/share/vm/classfile/classFileParser.cpp#l2751)：
```
if( allocation_style == 0 ) {
  // Fields order: oops, longs/doubles, ints, shorts/chars, bytes
  next_nonstatic_oop_offset    = next_nonstatic_field_offset;
  next_nonstatic_double_offset = next_nonstatic_oop_offset + 
				 (nonstatic_oop_count * oopSize);
} else if( allocation_style == 1 ) {
  // Fields order: longs/doubles, ints, shorts/chars, bytes, oops
  next_nonstatic_double_offset = next_nonstatic_field_offset;
} else {
  ShouldNotReachHere();
}
```
这两种策略不同点只在于oop在最前面还是在最后面。除外在分配好longs/doubles后，就是分配ints类型，即计算ints的起始偏移量，一定是在longs内存空间的末尾，即`next_nonstatic_double_offset + (nonstatic_double_count * BytesPerLong)`，剩余类型的起始偏移量计算也是如此。由于先处理longs/doubles进行对齐，所以末尾的下一个内存地址一定是8字节的整数倍，那么也一定是4的整数倍，因此ints字段是**天然对齐**的，同理之后的类型也是一定天然对齐的。在计算完5大类型起始偏移量后，就只要对Java类中所有字段进行遍历，[分别计算出各个字段的偏移量](http://hg.openjdk.java.net/jdk6/jdk6/hotspot/file/c6ff3162647f/src/share/vm/classfile/classFileParser.cpp#l2840)就可以了，然后按5大类型的起始偏移量进行顺序排序即可

特别的，由于Java类在堆内存中，都是从oop头开始的，而这个oop对象头所占的内存空间与JVM是否开启指针压缩策略有关，在64位平台下如果开启指针压缩，对象头只占12字节，没开启会占用16字节。那么如果`nonstatic_double_count > 0`，这不符合long类型的自然对齐原则，因此对齐会造成之间4个字节的浪费，JVM会把这部分空间也利用起来，将int、short、byte等宽度小于4字节的字段往这个内存间隙中插入。首先[将int类型填充进gap，如果有剩余填充short，还有剩余填充byte，还有剩余填充oopmap](http://hg.openjdk.java.net/jdk6/jdk6/hotspot/file/c6ff3162647f/src/share/vm/classfile/classFileParser.cpp#l2772)。这就是**gap填充**，不可避免的则会破坏重排顺序策略，而如果显示继承父类，父类字段末尾也不是按8字节对齐的，那么之间的间隙同样会形成补白空隙，进行gap填充

### 字段内存分配总结

从上述可以看出，JVM对内存空间的利用率是很高的，如果去掉占用12或16字节的对象头，其利用率会超出大多数的编程语言。那么通过Hotspot努力实现的分配策略，可以看出JVM的规范要求：  
1、任何对象都是以8字节为粒度进行对齐的(类在JVM内部的堆内存映像从**整体上对齐**，为后续其他类分配方便)  
2、类中的属性按照**优先级**进行排序：长整型/双精度、整型/浮点型、字符/短整型、字节/布尔、最后引用类型，这些属性会按各自类型宽度对齐(自然对齐)  
3、不同类继承关系中的成员**不能混合排序**，会先按第2条进行父类中成员排序，再子类的成员(可能导致内存利用率变低，但是继承体系较深时内存分析时方便)  
4、当父类的最后一个属性与子类第一个属性之间间隔不足4字节时，必须扩展到4字节的基本单位，即父类字段所占空间必然是**4的整数倍**(让父类属性集合从整体上做到对齐，方便后续子类字段处理对齐时方便)  
5、如果子类第一个成员是longs/doubles，并且父类没有用完8字节(没有显示的父类，且JVM开启指针压缩，oop对象头只占12字节时)，JVM会破坏第2条，按照int、short、byte、引用类型的顺序向为填满的空间**填充**(为了补白，有更高的内存空间利用率)

对于有父类参与时，需要同时满足第4条与第5条，这时候会违反第2条
![father-sort](/img/in-post/2019/03/father-sort.png)


## 继承关系

父类的字段重排和补白在前面的总结中已经演示过了

那么父类中private的字段是否只属于父类私有？子类是无法继承得到的么？其实在执行子类构造函数时，肯定先要执行父类的构造函数，**父类的private字段其实已经在子类的内存中了**，只不过虽然子类能继承其父类私有变量，但没有权利直接使用，除非父类开放了getter方法让子类使用

使用一个例子，来验证一下Java字段的继承：父类的private字段、final字段是否被继承，父类的字段是否被覆盖
```
public abstract class Father {
    private Integer i = 1;
    protected long l = 12L;
    protected final short s = 2;
    public char c = 'A';	
}

public class Son extends Father {
    private Integer i = 3;
    private long l = 18L;
    private short s = 4;
    public char c = 'B';

    public void add(int a, int b) {
        Son son = this;
        int z = a + b;
        int x = 3;
	}
    
	public static void main(String[] args) {
        Son son = new Son();
        son.add(1, 2);
	}
}
```
1、在IDE中运行main()方法时在add方法打断点，或者使用jdb命令打断点调试`jdb -XX:+UseSerialGC -Xmx10m -XX:-UseCompressedOops`  
2、在jdb中使用`stop in Son.add`、`run Son`、`next`命令让断点到add方法第二行z的计算上  
3、使用jps查看当前Java进程的进程ID  
4、使用命令`java -classpath "%JAVA_HOME%/lib/sa-jdi.jar" sun.jvm.hotspot.HSDB`启动**HSDB工具**  
5、ALT+A输入进程ID连接(最开始有个报错缺少sawindbg.dll，可以从JDK/jre/bin下复制到jre/bin下)  
6、在HSDB工具点击windows->console打开控制台，输入`universe`可以查看当前Java进程的堆内存
![heap](/img/in-post/2019/03/heap.png)
7、使用scanoops命令查询Son类  
8、使用inspect命令查看这个地址oop的全部数据，size为64  
![inspect](/img/in-post/2019/03/inspect.png)
9、也可以从HSDB工具的tools->inspector打开窗口，输入地址查看界面化信息
![oop](/img/in-post/2019/03/oop.png)
结论：当子类中定义与父类相同名字、相同类型的字段时，无论父类中字段的访问修饰符是什么，或者是否被final修饰，**子类都不会覆盖父类字段**。这与Java类继承中的方法继承大不相同，但是子类虽然不覆盖父类字段，也不是有权直接使用父类全部字段  
10、这里看的字段顺序和定义的顺序一样，显然不是真正分配的顺序，从tools->class browser可以看到Son类中各个字段的偏移量
![son-class](/img/in-post/2019/03/son-class.png)
11、从最小的field偏移量也是40，而oop的对象头在64位机器上最多占16字节，显然前面的部分是父类的，点击super class可以查看
![father-class](/img/in-post/2019/03/father-class.png)
12、这样就很明白了，对象头占用16字节，Father字段域从**16~40**(long:8,short:2,char:1+5,reference:8)，Son字段域为**40~64**(long:8,short:2,char:1+5,reference:8)，分别做了2处补白  
13、对于定义的Integer引用类型成员变量，对于类成员变量i，其在**Son类实例**对应的instanceOop的字段域中，i的实际值被存储在**java.lang.Integer类型实例**在JVM内部对应的instanceOop的字段域中，而在Son实例的instanceOop的i仅仅是一个指针引用，同样可以通过inspect界面查看。因此得知**类的成员变量的引用(指针)和类成员变量的实例都分配在JVM的堆中，类的成员变量的引用分配在所在类的实例的instanceOop的字段域中**  
![main](/img/in-post/2019/03/main.png)
14、在main()主方法中实例化了一个Son类对象，由于son属于main方法的局部变量，因此一定在其栈帧中有这个地址的引用。由于当前Son进程的main主线程最终会调用到Java程序的主函数，因此可以通过主线程观察到Son类的**main主函数的栈帧**，栈帧从下往上看(增长方向自下而上，即高地址往低地址)  
![son-stack](/img/in-post/2019/03/son-stack.png)
![stack-frame](/img/in-post/2019/03/stack-frame.png)

Java类在堆内存中的内存空间，主要由Java类非静态字段占据，Hotspot解析Java类非静态字段和分配空间的主要逻辑总结为：  
1、解析常量池，统计Java类中**非静态字段的总数量**，按照5大类型(oops、longs/doubles、ints、shorts/chars、bytes)分别统计  
2、计算Java类字段的起始偏移量，**起始偏移位置**从父类继承的字段域的末尾开始  
3、按照分配策略，计算5大类型中**每一个类型**的起始偏移量  
4、以5大类型各个类型的起始偏移量为基准，计算每一个大类型下**各个具体字段**的偏移量  
5、计算Java类在内存中所需要的内存空间  
这样Hotspot就能确定一个Java类所需要的堆内存空间，当全部解析完Java类后，Java类的**全部字段信息以及偏移量将会保存到instanceKlass中**，这样一个Java类的字段结构信息便全部解析完成。当Java程序中使用new关键字创建类的实例对象时，Hotspot便会从**instanceKlass中读取Java所需要的堆内存大小**并分配对应的内存空间


# 类方法解析

在对Java字节码解析终于来到了最后一部分，方法解析。对于面向对象语言，一个类具有属性与行为两个基本要素，而Java又作为解释型语言由JVM执行行为 --- Java方法，在执行之前，必须要先对方法进行解释，毕竟JVM虚拟机没有真正的运算能力，最终是依靠CPU完成Java字节码的运算。Java方法的解析总体上可分为3个步骤：  
1、在Java类源代码**编译期间**，编译器将Java源代码翻译为对应字节码指令，同时还完成Java方法局部变量表计算，最大操作数栈的计算  
2、在JVM**运行期间**，**JVM加载类型**，调用classFileParser::parseClassFile()函数对Java class字节码文件进行解析，会完成Java方法的解析、字节码指令存放、父类与接口方法继承与重载等一系列逻辑  
3、在调用系统加载器System Class Loader(SCL)对应用程序的Java类进行**加载的过程**中，完成方法符号链接、验证，最重要的是完成**vtable与itable的构建**，从而支持在JVM运行期的方法动态绑定(也叫晚绑定)。除了SCL外还有其他类加载器用于加载Java类  
在完成好Java类方法的解析工作后，JVM才能在运行期通过invoke_virtual等字节码指令完成Java方法的调用和执行

同样在`ClassFileParser::parseClassFile()`函数的解析步骤中，通过调用`parse_methods()`函数完成[Java方法解析的第2步骤](http://hg.openjdk.java.net/jdk6/jdk6/hotspot/file/c6ff3162647f/src/share/vm/classfile/classFileParser.cpp#l1727)：从Java class流中读取当前Java类中定义的全部方法数量，接着**循环遍历**每一个方法，调用`parse_method()`函数对Java方法逐个解析，[parse_method方法](http://hg.openjdk.java.net/jdk6/jdk6/hotspot/file/c6ff3162647f/src/share/vm/classfile/classFileParser.cpp#l1216)主要逻辑为：  
1、读取和验证Java方法的访问标识、名称  
2、解析方法的属性  
3、创建methodOop  
4、复制字节码

## 方法签名解析与校验

Java class字节码文件中的方法属性部分解析，从方法的**flags标识的索引**(该标识在常量池的索引号)开始，flags标识占用2字节长度，并且后面连续跟了4字节，分别是方法名称索引和方法描述索引，也是各占2个字节。Java方法的签名信息主要由3部分组成：  
1、**方法的标识**：public、private、static、final、synchronized、native等  
2、**方法的名称**  
3、**方法的描述**：描述入参信息、返回类型，例如()V标识无入参的void类型方法  
这3部分信息共同组成Java方法的**签名信息**，由于每种信息都是指向常量池的索引号，因此只需要2字节，Java方法签名一共占用6字节长度。在`parse_method()`方法开始，连续3次调用`cfs->get_u2_fast()`从字节码文件中读取出签名的3部分，并逐一进行校验，通过3个verify方法：`verify_legal_method_name`、`verify_legal_method_modifiers`、`verify_legal_method_signature`分别校验有效性

例如`verify_legal_method_name`会判断方法名称的第一个字符是否是`<`，如果是则判断是否为编译器自动生成的`<init>或<clinit>`方法，如果不是则校验不通过；如果不以`<`开始且版本大于1.5，会判断Java方法名中是否有特殊标识，如果有则不通过

## 方法属性解析

在解析和校验完方法的签名信息后，接着就开始解析Java方法的属性了。在字节码文件中主要使用几大属性来描述Java方法的方方面面信息，这些属性包括有：**code、exception、line number table等**。这些属性不一定每个都在字节码文件中出现，比如没有抛出异常则不会有异常表产生。同时字节码文件描述属性的字节码区域之间不是有序的，因此字节码文件中仅存储**Java方法的属性总数量**，在`parse_method()`函数中根据总数量，[while循环](http://hg.openjdk.java.net/jdk6/jdk6/hotspot/file/c6ff3162647f/src/share/vm/classfile/classFileParser.cpp#l1306)逐个读取Java方法的当前属性，通过属性名来判断当前是哪个属性，然后根据不同属性承载信息与结构的不同，分别使用不同的策略进行解析

#### code解析

code属性主要包含属性的总长度、最大栈深度、局部变量表数量、字节码指令，除此之外code属性本身是一个复合属性，下面还包含几个子属性，例如行号表、局部变量表等。code属性起始于**属性名称的常量池索引号**，索引号后跟随**属性长度**，因此在处理逻辑中首先通过`cfs->get_u2_fast()`和`cfs->get_u4_fast()`分别获取当前属性的索引号和总长度。在总长度后面的3个属性是：**max_stack、max_locals、code_length**，由于在不同版本JVM中，它们所占的数据宽度不同，因此会根据版本调用`cfs->get_u*_fast()`获取这3个属性数据。在code_length后面的就是Java源码所对应的字节码指令了，这部分指令最终会被从字节码文件复制到内存中，具体而言是复制到Java方法在JVM内部对应的methodOop对象，这时还不会复制，仅通过`cfs->get_u1_buffer()`将字节码的第一条指令所在位置记录到code_start中，最终Hotspot会根据**code_start和code_length确认复制字节码指令的区域**
```
while (method_attributes_count--) {   
    cfs->guarantee_more(6, CHECK_(nullHandle));  // method_attribute_name_index, method_attribute_length
    u2 method_attribute_name_index = cfs->get_u2_fast();
    u4 method_attribute_length = cfs->get_u4_fast();
    // ...
	
    symbolOop method_attribute_name = cp->symbol_at(method_attribute_name_index);
    if (method_attribute_name == vmSymbols::tag_code()) {
        // Parse Code attribute
        // ...
	  
        // Stack size, locals size, and code size
        if (_major_version == 45 && _minor_version <= 2) {
            cfs->guarantee_more(4, CHECK_(nullHandle));
            max_stack = cfs->get_u1_fast();
            max_locals = cfs->get_u1_fast();
            code_length = cfs->get_u2_fast();
        } else {
            cfs->guarantee_more(8, CHECK_(nullHandle));
            max_stack = cfs->get_u2_fast();
            max_locals = cfs->get_u2_fast();
            code_length = cfs->get_u4_fast();
        }
        // ...

        // Code pointer
        code_start = cfs->get_u1_buffer();
        // ...
    }
    // ...
```

#### LVT&LVTT

在Java方法的属性中，有一种属性是**局部变量表**，LocalVariableTable，简称为LVT。LVT属性用于描述Java方法栈帧中局部变量表中的变量与Java源码定义变量之间的关系，这种关系并**非运行时必需**，因此默认情况下不会生成到class文件中，如果需要可以通过为javac添加`-g:vars`选项生成。如果class字节码文件中没有生成局部变量表，则在调试Java程序时，无法看到源码定义的参数名称，IDE会用arg0、arg1占位符代替原来的参数，这对程序运行并不会造成影响

class文件中，LocalVariableTable属性表的结构：

|         类型        |             名称            |             数量            | 
| :------------------ | :-------------------------- | :-------------------------- |
| u2                  | attribute_name_index        | 1                           |
| u4                  | attribute_length            | 1                           |
| u2                  | local_variable_table_length | 1                           |
| local_veriable_info | local_variable_table        | local_variable_table_length |

local_veriable_info是复合数据结构，用于描述**Java方法栈帧与源代码中局部变量的关联**，结构为：

|   类型   |      名称        |                                     定义                                   | 数量 | 
| :------- | :--------------- | :------------------------------------------------------------------------- | :--- |
| u2       | start_pc         | 当前局部变量的生命周期开始的字节码偏移量                                   | 1    |
| u2       | length           | 局部变量的作用范围覆盖长度，和start_pc一起表示局部变量在字节码中的作用范围 | 1    |
| u2       | name_index       | 当前局部变量的名称所对应的常量池的索引号                                   | 1    |
| u2       | descriptor_index | 当前局部变量的描述信息所对应的惨啊凌迟的索引号                             | 1    |
| u2       | index            | 当前局部变量在栈帧局部变量中slot的位置                                     | 1    |

研究局部变量表对于程序开发者没什么具体意义，而对Java程序调试的一些工程实现策略有关，因此这种策略不仅在Java中有应用，在其他编程语言中也有类似实现。在调试过程中让IDE面对编译后毫无意义的栈帧占位符时，能展现出占位符对应的源码中的原始变量名，这是所有编程语言都具备的能力

Hotspot局部变量表的分析逻辑在Java编译器中实现，编译器会对Java源码进行语法解析，分析Java方法的栈帧结构，并据此分析各个变量的作用域。分析的结果以特定的组织结构存储在class字节码文件中。当Hotspot加载某个Java类时，会按照这种特定的组织结构还原出编译器所分析的结果，从而在调试期间能据此展示出局部变量的原始名称。会在`parse_localvariable_table`函数中读取出数据，进行简单的校验，然后在`parse_method()`函数中进行[复制局部变量表](http://hg.openjdk.java.net/jdk6/jdk6/hotspot/file/c6ff3162647f/src/share/vm/classfile/classFileParser.cpp#l1614)，主要是将局部变量表从字节码文件**复制到JVM内部对应的constMethodOop**这个内部对象实例的内存区域的**末尾**(即byte code之后)

在JVM运行期，如果打断点后，JVM运行到断点处后中断暂停，此时JVM能通过栈帧上指向methodOop的指针定位到constMethodOop，由于局部变量表就保存在constMethodOop的内存末尾位置，因此JVM能基于constMethodOop进一步获取到当前Java方法所有的局部变量，从而在IDE中将入参和局部变量以原始变量名显示出来

在JDK 1.5引入泛型之后，LocalVeriableTable属性增加了一个"姐妹属性"，即**LocalVariableTypeTable**，简称为LVTT，其结构与LVT类似，仅仅把记录的字段描述符的descriptor_index替换为**字段特征签名**(signature)

## 创建methodOop

完成对Java方法的各项属性解析后，Hotspot开始在内存中创建一个与Java方法**对等的内部对象** --- methodOop。其包含Java方法的一切信息，比如方法名、返回类型、字节码指令、栈深、局部变量表、行号表等，相当于把字节码中的方法信息存到了**结构化的methodOop**中，让JVM在运行期能方便地访问Java方法的各种属性信息

在`parse_method()`函数中通过调用`oopFactory::new_method()`函数完成methodOop对象的创建：
```
methodOop oopFactory::new_method(int byte_code_size, AccessFlags access_flags,
                                 int compressed_line_number_size,
                                 int localvariable_table_length,
                                 int checked_exceptions_length, TRAPS) {
  methodKlass* mk = methodKlass::cast(Universe::methodKlassObj());
  assert(!access_flags.is_native() || byte_code_size == 0,
         "native methods should not contain byte codes");
  constMethodOop cm = new_constMethod(byte_code_size,
                                      compressed_line_number_size,
                                      localvariable_table_length,
                                      checked_exceptions_length, CHECK_NULL);
  constMethodHandle rw(THREAD, cm);
  return mk->allocate(rw, access_flags, CHECK_NULL);
}
```
这里面实际创建了2个对象，分别是methodOop和constMethodOop，这2个对象在JDK6和8中都有，**methodOop**主要用于存储Java方法的名称、签名、访问标识、解释入口等信息，而**constMethodOop**则主要用于存储方法的字节码指令、行号表、异常表等信息。在JDK8中的变化之前[博文](https://zhouj000.github.io/2019/03/18/java-base-jvm4/)说过，就是最大栈深度和局部变量表数量存储的位置发生了移动
![methodoop](/img/in-post/2019/03/methodoop.png)

创建methodOop和常量池基本一致，常量池最后调用`constantPoolKlass:allocate()`函数完成创建，而类方法通过`methodKlass::allocate()`函数创建，逻辑都是先获取methodOop的大小size，然后调用`CollectedHeap::permancent_obj_allocate()`函数在perm区(对于JDK8则在metaSpace区)为创建的对象分配内存，最后再调用一系列的`setter方法`初始化创建的对象


## Java方法属性复制

当与Java方法对应的methodOop创建完成后，Hotspot需要将Java方法的**几大属性数据复制进创建的对象中**，包含：code(字节码指令)、行号表、异常表、局部变量表等。在创建methodOop时，顺便创建了constMethodOop对象实例，Java方法的属性信息被分配在constMethodOop对象实例的内存区域末尾位置。在创建constMethodOop时已经预先为这些属性申请好了空间，所以接下来只要将这些属性从class文件中复制到申请的内存中

## clinit和init

在Java中，有两种特殊方法是非开发者定义，而是由**Java编译器自动生成**的，它们就是`<clinit>`和`<init>`。当Java类中存在使用`static`修饰的静态类型字段，或者存在`static{}`包裹的逻辑时，编译器就会自动生成`<clinit>`方法；而当Java类定义了构造方法，或者其非static类成员变量被赋予了初始值的时候，编译器就自动生成了`<init>`方法

#### init

1、无论一个Java类是否有定义构造函数，编译器都会自动生成一个默认的构造函数`<init>()`  
2、`<init>()`方法主要完成Java类**成员变量的初始化**逻辑，同时执行Java类中被`{}块`包裹的代码逻辑。如果Java中的成员变量没有被赋予初始值，那么`<init>()`方法中也不会对其进行初始化  
3、无论为Java显示定义了多少个构造函数，还是仅适用默认的无参构造函数，编译器都会**修改构造函数的字节码指令**，将Java类成员变量的初始化指令和`{}`包裹的代码逻辑嵌入到每一个构造函数中，并且嵌入的位置在各构造函数自身的逻辑之前  
4、如果Java类显式继承了父类时，编译器会让子类的各个构造函数**调用父类的默认构造函数**`<init>()`，其嵌入顺序位于子类各构造函数自身逻辑之前，从而在子类实例中完成父类成员变量的初始化逻辑

使用一个例子来验证一下：
```
public abstract class Father {
	private int a = 200;
	public int b = 201;
	public static int c = 202;
	{
		int d = 203;
		System.out.println(a);
	}
	public Father() {
	}
	public Father(int a) {
		String s = "father";
		this.a = a;
	}
}

public class Son extends Father {
	private int e = 204;
	private static int f = 205;
	{
		int g = 206;
		System.out.println(g);
	}
    public Son() {
		String s = "son";
	}
	public Son(int e) {
		this.e = e;
		String s = "son";
	}
}
```
使用`javap -verbose Son`可以看到
```
public Son();
  descriptor: ()V
  flags: ACC_PUBLIC
  Code:
    stack=2, locals=2, args_size=1
       0: aload_0
       1: invokespecial #1                  // Method Father."<init>":()V
       4: aload_0
       5: sipush        204
       8: putfield      #2                  // Field e:I
      11: sipush        206
      14: istore_1
      15: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream
      18: iload_1
      19: invokevirtual #4                  // Method java/io/PrintStream.println:(I)V
      22: ldc           #5                  // String son
      24: astore_1
      25: return
    LineNumberTable:
      line 11: 0
      line 3: 4
      line 7: 11
      line 8: 15
      line 12: 22
      line 13: 25

public Son(int);
  descriptor: (I)V
  flags: ACC_PUBLIC
  Code:
    stack=2, locals=3, args_size=2
       0: aload_0
       1: invokespecial #1                  // Method Father."<init>":()V
       4: aload_0
       5: sipush        204
       8: putfield      #2                  // Field e:I
      11: sipush        206
      14: istore_2
      15: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;
      18: iload_2
      19: invokevirtual #4                  // Method java/io/PrintStream.println:(I)V
      22: aload_0
      23: iload_1
      24: putfield      #2                  // Field e:I
      27: ldc           #5                  // String son
      29: astore_2
      30: return
    LineNumberTable:
      line 15: 0
      line 3: 4
      line 7: 11
      line 8: 15
      line 16: 22
      line 17: 27
      line 18: 30	  
```

#### clinit

当Java类中存在使用`static`修饰的静态类型字段，或者存在`static{}`包裹的逻辑时，编译器就会自动生成`<clinit>`方法。但是`<clinit>`方法并**不具有继承性**，因为`<clinit>`方法是在**类加载过程中**被调用，而父类与子类是分别加载的，当父类加载完后，父类中static成员变量初始化和被`static{}`包裹的逻辑都已经执行完了，没必要在子类中再执行一遍，所以子类只需要完成自身的static变量初始化和`static{}`包裹的逻辑就可以了

由于`<clinit>`方法在Java类第一次被JVM加载时就被调用，而`<init>`方法则需要在Java类被实例化时调用，因此`<clinit>`方法在`<init>`方法之前调用且仅被调用**一次**。`<clinit>`方法的访问标识一定是ACC_STATIC，而`<init>`则为ACC_PUBLIC

#### {}与static{}作用域

被`{}块`包裹的代码块，能访问非静态成员变量和静态成员变量，但是内部定义的变量不能被外部访问，与Java普通非static方法的作用域完全相同，而在编译阶段，`{}块`中的局部变量和普通非static方法也一样，被组织成了**局部变量表**，从上面的例子输出可以看出，第14行的`istore_1`表示编译器将变量g存储到局部变量表的第2个局部变量。既然编译器将`{}块`的局部变量处理成局部变量表，而局部变量表是与函数相关的，而编译器将`{}块`的逻辑嵌入到了`<init>`构造函数中，因此这里的局部变量表自然就是Java类的**构造函数**的了

那么如果有多个`{}块`会怎样呢？看下例子:
```
private int a = 11;
private static int b = 12;
{
	int c = 100;
}
{
	int d = 101;
}
{
	int c = a + b;
}

public Son();
  descriptor: ()V
  flags: ACC_PUBLIC
  Code:
    stack=2, locals=2, args_size=1
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: aload_0
       5: bipush        11
       7: putfield      #2                  // Field a:I
      10: bipush        100
      12: istore_1
      13: bipush        101
      15: istore_1
      16: aload_0
      17: getfield      #2                  // Field a:I
      20: getstatic     #3                  // Field b:I
      23: iadd
      24: istore_1
      25: return
```
发现它们都是使用的`istore_1`，那么很明显了，编译器将这些`{}块`并不是简单合并到构造函数中，而是**单独存在**的，多个`{}块`之间没有任何联系。既然`{}块`的局部变量不会被其他`{}块`引用，更不会被构造函数引用到，因此`{}块`在使用完局部变量表后，就可以立即被"回收"了，被"回收"的局部变量表空间其实就是被其他`{}块`直接拿来**复用**(实际上没有回收动作，只是被用来复用了)

那么既然`{}块`能被当做一个普通的非static方法处理，那么`static{}`自然可以看做一个static修饰的Java静态方法处理，唯一不同的就是其在Java类被加载时就执行了。其作用域也很显然，仅能访问静态成员变量，且定义的局部变量不能被外部访问

这里说的`<clinit>`方法和`<init>`方法，还有前面讲到的methodOop、constMethodOop都可以通过HSDB看到，在class browser或inspector都可以根据地址查看


## vtable

Java是一门面向对象的语言，面向对象一大特色便是多态，多态的主要目的就是消除类型之间的耦合关系。**实现多态的技术称为动态绑定**(dynamic binding)，是指在执行期间判断所引用对象的实际类型，根据其实际类型调用其相应的方法。在Java中，动态绑定也叫做**晚绑定**，这是因为在Java中还有一类绑定是在编译期间便能确定，所以所谓的晚绑定概念是相对于编译器绑定而言的。绑定通俗将就是让不同的对象对同一个函数进行调用，或者反过来讲就是将同一个函数与不同的对象绑定起来，所以多态性得以实现的一大前提就是编程语言必须是面向对象的，同时函数与对象相互绑定，意味着函数属于对象的一部分，这便具备了封装的特性，因为有了封装，才有对象，才能叫做面向对象编程。又同时，一个函数能绑定多个不同对象，意味着多个不同对象具有相同的行为，这边是继承的含义。因此面向对象3大特性 --- **封装、继承、多态**，其中前2个特性都是为了最后一个多态而准备的

### 多态实现与vtable

JVM实现晚绑定的机制基于**vtable**，即virtual table，虚方法表。JVM通过虚方法表在**运行期动态确定**所调用的目标类的目标方法

C++也实现了多态，当使用**virtual**修饰方法成为虚方法后，表示该方法拥有多态性，此时会根据类型指针所指向的实际对象而在运行期调用不同的方法，与Java中的多态在语义中是完全一致的。C++为了实现多态，就在C++类实例对象中嵌入虚函数表vtable，通过虚函数表来实现运行期的**方法分派**。C++中所谓虚函数表，其实就是一个普通的表，表中存储的是**方法指针**，方法指针会指向目标方法的内存地址，所以虚函数表只是一堆指针的集合而已。对于大部分C++编译器，出现虚函数表时，都会将其分配在C++对象实例的起始位置，即内存分配会先分配虚函数表，再分配类中的字段空间

Java的多态实现机制也是如此，使用了**vtable**来实现动态绑定。Java类在JVM内部对应的对象是instanceKlassOop(JDK 8中是instanceKlass)，在JVM加载Java类的过程中，JVM会动态解析Java类的方法以及其对父方法的重写，进而构建出一个vtable，并将vtalbe分配到instanceKlassOop内存区的**末尾**，从而支持运行期的方法动态绑定。JVM的vtable机制与C++的vtable机制之间最大的不同在于，**C++的vtable在编译期间便由编译器完成分析与模型构建，而JVM的vtable机制则在JVM运行期、Java类被加载时进行动态构建**，也可以认为JVM在运行期做了C++编译器在编译期间所做的事情

前面讲过，在`classFileParser.cpp::parseClassFile()`函数对class字节码文件进行解析，其中调用了`parse_methods()`函数解析Java类中的方法，在`parse_methods()`调用完后，会继续调用`klassVtable::compute_vtable_size_and_num_mirandas()`函数，[计算当前Java类的vtable大小](http://hg.openjdk.java.net/jdk6/jdk6/hotspot/file/c6ff3162647f/src/share/vm/oops/klassVtable.cpp#l46)，其主要思路分为两步：  
1、获取**父类vtable**的大小，并将当前类的vtable的大小设置为父类vtable的大小  
2、循环遍历当前Java类的每一个方法，调用[needs_new_vtable_entry()函数](http://hg.openjdk.java.net/jdk6/jdk6/hotspot/file/c6ff3162647f/src/share/vm/oops/klassVtable.cpp#l398)进行判断，如果判断为true，则将**vtable增加1**  

Java类在运行期间进行动态绑定的方法，一定被声明为**public或者protected的**，并且**没有static和final修饰**，且Java类上也**没有final修饰**，这主要是因为：  
1、如果一个Java方法被**static**修饰，那么不会参与到整个Java类的继承体系中，所以静态方法不会参与运行期的动态绑定机制。所谓动态绑定，是指Java类实例与Java方法搭配，而静态方法的调用根本就不需要通过类实例  
2、如果一个Java方法被**private**修饰，则外部根本无法调用此方法，只能被该类的内部其他方法调用，因此不会参与动态绑定  
3、如果一个Java方法被**final**修饰，则子类无法重写该方法，那么既然是Java类固有方法，也不会有多态  
4、如果一个Java类被**final**修饰，那么该Java类中所有非静态方法都隐式被final修饰，显然不会出现多态，不用动态绑定  
满足上面的条件，那么Java方法只能被public或protected修饰，那么方法有可能参与动态绑定。然而仅仅这4个条件还不能满足，还需要：  
5、父类中必须包含**名称**相同的Java方法，其**签名**也必须完全一致，显而易见，即子类重写的方法  
那么满足条件的方法，`needs_new_vtable_entry()`函数就会返回true，会对vtable_length加1

每一个Java类在JVM内部都有一个对应的instanceKlassOop，vtable就被分配在这个oop内存区域的后面。**vtable表中的每一个位置存放一个指针，指向Java方法在内存中所对应的methodOop内存首地址**。如果一个Java类继承了父类，则该Java类就会**继承**父类的vtable。若该Java类中声明了一个非private、非final、非static的方法，且是父类方法的重写，则JVM会更新父类vtable表中指向父类被重写方法的指针，**使其指向子类中该方法的内存地址**。若该方法不是父类方法的重写，则JVM会向该Java类的vtable中插入一个**新的指针元素**，使其指向该方法的内存位置

```
public class Father {
	public void print() {
			System.out.println("father");
	}
}

public class Son extends Father {
	public void print() {
		System.out.println("son");
		newFun();
		privateFun();
	}
	public void newFun() {
	}
	private void privateFun() {
	}
}
```
Hotspot在运行期加载Father时，其vtable中将会有一个指针元素指向其`print()`方法在Hotspot内部的内存首地址。当Hotspot加载Son时，首先**继承**Father的vtable，因此Son也就有了一个vtable，并且vtable里有一个指针指向Father类的`print()`方法的内存地址。Hotspot遍历Son类所有方法，发现`print()`是符合条件的，于是将Son的vtable中原本指向Father类的print()方法的内存地址的指针**修改成指向Son类**自己的`print()`方法所在的**内存地址**。而当解析到`newFun()`方法时，并非重写方法，但是**满足**vtable条件，于是Hotspot将Son类原本继承于Father的vtable的长度**增加1**，并将新增的vtable的指针元素指向`newFun()`方法在内存中的位置，而`privateFun()`是private修饰的，因此**不符合条件**，不会增长vtable

### vtable与invokevirtual指令

前面见过，JVM通过调用CallStub例程开始调用Java程序的主函数main()，在CallStub例程内部最终调用zero_locals这个例程，而在zero_locals例程中最终跳转到Java程序的main()主函数的第一条字节码指令开始执行Java程序。那么在Java程序内部，一个Java方法调用另一个Java方法时，是怎么调用的？先要了解Java的字节码指令中方法的4种调用实现：  
1、**Invokevirtual**，最常见的情况，包含virtual dispatch(虚方法分发)机制  
2、**Invokespecial**，调动private和构造方法，绕过了virtual dispatch  
3、**Invokeinterface**，实现与Invokevirtual类似  
4、**Invokestatic**，调动静态方法  
其中**Invokevirtual指令涉及多态的特性**，凡是Java类中需要在运行期动态绑定的方法调用，都通过Invokevirtual指令，该指令将实现**方法的分发**。在Hotspot执行Invokevirtual指令的过程中，最终会**读取被调用的类的vtable虚函数表，并据此决定真实的目标调用方法**。JVM内部实现virtual dispatch机制时，会**首先从receiver**(被调用方法的对象)的类的实现中查找对应的方法，如果没有找到则去**父类**查找，直到找到函数并实现调用，而不是依赖于引用的类型

使用javap分析Son的print方法
```
public void print();
  descriptor: ()V
  flags: ACC_PUBLIC
  Code:
    stack=2, locals=1, args_size=1
       0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: ldc           #3                  // String son
       5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       8: aload_0
       9: invokevirtual #5                  // Method newFun:()V
      12: aload_0
      13: invokespecial #6                  // Method privateFun:()V
      16: return
```
调用`newFun()`的是invokevirtual指令，调用`privateFun()`的是invokespecial指令。这是因为`newFun()`是public且非staic和final的，因此这个方法可以被继承，即**可被子类重写的**。而编译器在编译期间**并不知道**Son有没有子类，因此只能使用invokevirtual指令去调用`newFun()`方法，从而使`newFun90`方法支持在运行期进行动态绑定。虽然编译器在编译期间可以分析整个工程来确定Son有没有子类，但是JVM是支持在**运行期动态创建新的类型的**，例如使用cglib，因此编译器根本无法得知运行期会不会出现一个类继承了Son并重写newFun方法。而`privateFun()`则不同，它是private方法，即私有方法，是不需要动态绑定的，因此编译期间就能确定其调用者，所以对应的字节码指令为invokespecial

### miranda方法

在Hotspot早期的虚拟机里有一个bug，JVM在遍历解析Java方法时，仅仅遍历Java类以及所有父类的方法，但是不会去查找Java类所实现接口interface里的方法，这样会导致如果Java类**没有实现**接口里的方法，则**接口方法**不会被虚拟机查到。为了解决这个问题，编译器引入一个相反的方法，就是在编译器往Java类中**插入接口里定义的方法**，这些方法就是所谓的miranda方法，但是miranda方法并不是Java规范的一部分，这相当于解决问题引入的新bug

那么不是interface里的方法都需要被实现么？怎么会有这个问题呢？但是Java中的abstract抽象类允许不实现接口方法，例如：
```
public interface IA {
	void test();
}

public abstract class B implements IA {
	public B() {
		test();
	}
}
```
这样式可以通过编译的，但是在B的构造函数中调用了test方法，如果编译器没有miranda机制，则在B的构造函数中调用test()接口方法时，肯定会编译报错，因为这个方法只存在于IA中，而不存在与B与B的任何父类中

在Java类中调用miranda方法时，实现的是**动态绑定策略**，而非早绑定。因此miranda方法将会被加入到**vtable**中，在调用`klassVtable::compute_vtable_size_and_num_mirandas()`计算vtable长度时，其中包含了对miranda方法的处理，会计算出当前类的所有miranda方法的总数，并将其加入到vtable的长度变量中，miranda方法的总数即当前类实现的所有接口以及各接口父类接口类中的方法、且并未在当前类实现的方法总数。对于接口方法，Java类中还会保存一种内部结构 --- **itable**，即接口方法表。itable主要用于**接口类方法的分发**，机制与vtable类似，都会在运行期进行方法的动态绑定

### vtable机制逻辑实现

```
public class Test {
	public static void main(String[] args) {
		Father father = new Father();
		say(father);
		Father son = new Son();
		say(son);
	}
	public static void say(Father father) {
		father.print();
	}
}
```
多态通过vtable实现，这里打印的结果是`father son`。由于Father也是继承了**Object**，因此其vtable长度为6，即除了Object中的5个可被重写虚方法外，其自身包含1个虚方法，并且其vtable中的第6个指针元素指向print()方法在JVM内部所对应的method实例对象的内存地址。同理，子类Son的vtable长度为7，因为子类会**完全继承**父类的vtable，如果子类重写了父类的方法，JVM会将子类vtable中原本指向父类方法的指针成员修改为重新指向子类的方法，除外又有一个newFun()方法符合条件，创建新的指针
![vtable](/img/in-post/2019/03/vtable.png)

```
Constant pool:
   #2 = Class              #21            // Father
   #7 = Methodref          #2.#24         // Father.print:()V
  #24 = NameAndType        #27:#11        // print:()V

public static void say(Father);
  descriptor: (LFather;)V
  flags: ACC_PUBLIC, ACC_STATIC
  Code:
    stack=1, locals=1, args_size=1
       0: aload_0
       1: invokevirtual #7                  // Method Father.print:()V
       4: return
```
第一步`aload_0`，表示从第0个slot位置加载Java引用对象，由于say方法为static方法，所以没有隐式this指针，第1个局部变量就是第一个入参father引用对象。第二步执行`invokevirtual #7`指令，Java多态的秘密就藏在后面的操作数operand中，是**常量池的索引值**，可以看到常量池7代表的字符串是`Father.print:()V`，则表示invokevirtual在运行期调用的方法是`print:()V`。在运行期，JVM首先确定**被调用的方法所属的Java类实例对象**，JVM会读取被调用的方法的堆栈，并获取堆栈中局部变量表的**第0个slot位置**的数据，其**一定为指向被调用方法所属的Java类实例**。因为凡是对应字节码指令为invokevirtual的Java方法，就必定是Java类的成员方法，而非静态方法，那么成员方法第一个入参一定是**隐式this指针**，该指针就指向了Java类对象的实例。而第一个入参一定位于局部变量表的第0个位置，因此JVM可以从invokevirtual所调用的**局部变量表中读取到this指针**，从而知道被调用的Java类实例到底是谁。JVM获取到invokevirtual指令所调用的方法所属的实际类对象后，接着就能通过对象获得到**对应的vtable方法分发表**。vtable表保存了当前类的方法的指针，JVM遍历vtable每一个指针成员，并根据指针读取到对应的**method对象**，判断invokevirtual指令所调用的方法名称和签名是否与vtable表中指针指向的方法的名称和签名是否一致，如果名称和签名完全一致，那么就找到了invokevirtual所**实际调用的目标方法**，于是JVM定位到目标方法第一条字节码指令并开始执行。如此便完成了方法在运行期的动态分发和执行

### 总结

1、vtable分配在**instanceKlassOop**对象实例的**内存末尾**  
2、所谓vtable，可以看做是一个**数组**，数组的每一项成员元素都是一个**指针**，指针指向Java方法在JVM内部所对应的method实例对象的内存**首地址**  
3、**vtable**是Java实现面向对象的**多态性**的机制，如果一个Java方法可以被继承和重写，则最终通过**invokevirtual**字节码指令完成Java方法的动态绑定和分发  
4、Java子类会**继承**父类的vtable  
5、Java中所有类都继承自**java.lang.Object**，其有5个虚方法：`void finalize()、boolean equals(Object)、String toString()、int hashCode()、Object Object clone()`，因此如果一个Java类不声明任何方法，其vtable长度默认为5  
6、Java类中**不是每一个**Java方法的内存地址都会保存到vtable表中，只有子类声明的方法是public/protected，且没有final/static修饰，并且该方法并非对父类方法的重写时，JVM才会在vtable表中为该方法增加一个引用  
7、如果子类中的某个方法重写了父类方法，那么子类中vtable的原本对父类方法的指针引用会被**替换为对子类的方法引用**



参考：  
《解密Java虚拟机》  

