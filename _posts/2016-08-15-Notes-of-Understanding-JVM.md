---
layout: post
title: 《深入理解Java 虚拟机》读书笔记
category: 技术
tags: Java
catalog:  true
description: 读书笔记
---



## 引言

为什么 作为Java/Android 开发工程师的我们 需要了解 JVM 的原理?

因为虚拟机接管了对硬件平台的兼容和对内存等资源的管理工作，但为了达到给所有硬件提供一致的虚拟平台的目的，牺牲了一些与硬件相关的性能特性。如果开发者不了解虚拟机一些技术特性的运行原理，就无法写出最适合虚拟机运行和自优化的代码。



## 学习总结

### GC 的过程：

从 GC Roots 根节点找引用链开始进行可达性分析，

- 依靠 OopMap 可直接获知对象引用的位置，完成 GC Roots枚举
- 为保证 GC 时引用关系不变所以停顿所有线程（即 Stop The World），
- 使用 Safepoint 中断运行中的线程，使用 SafeRegion 来保证未运行的线程也能配合 GC。

知道可回收的对象后，进行垃圾收集

- 新生代使用复制算法，老年代使用“标记-整理”算法
- 大对象、存活很多轮 GC 的对象进入老年代
- 新生代容纳不下的对象进入老年代
- 达到一定比例的某个年龄段对象群进入老年代




## 第二部分 自动内存管理机制

## CH2 Java 内存区域与内存溢出异常

### 运行时数据区域

![JVM运行时数据区域](/images/jvm/jvm-runtime-data-partition.png)

#### 程序计数器

是一块较小的内存空间，是当前线程所执行的字节码的行号指示器。通过改变该计数器的值来选取下一条需要执行的字节码指令。

执行到Java 方法时，值为正在执行的虚拟机字节码指令的地址。执行到Native 方法时，值为Undefined。

唯一一个无OutOfMemoryError的区域。

线程私有。

#### Java 虚拟机栈

虚拟机栈描述的是Java 方法执行的内存模型：
每个方法在执行的同时都会创建一个栈帧（Stack Frame）用于存储局部变量表、操作数栈、动态链接、方法出口等信息。每一个方法从调用直到执行完成的过程，对应着一个栈帧在虚拟机栈中从入栈到出栈的过程。

可抛出 StackOverFlowError 异常和 OutOfMemoryError异常。

线程私有。

#### 本地方法栈

描述本地方法的执行过程。

可抛出 StackOverFlowError 异常和 OutOfMemoryError异常。

线程私有。

#### Java 堆

存储所有的对象实例和数组。

所有线程共享。可能划分出多个线程私有的分配缓冲区（Thread Local Allocation Buffer，TLAB）。

大小通过 -Xmx 和 -Xms控制。

可抛出 OutOfMemoryError异常。

对于采用分代收集算法的收集器，可细分为新生代和老年代;再细分为 Eden 空间、From Survivor 空间、To Survivor 空间。

#### 方法区

存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码。

回收目标：主要是常量池的回收和类型的卸载。

所有线程共享。

可抛出 OutOfMemoryError异常。

##### 运行时常量池

是方法区的一部分。
存放编译器生成的各种字面量、符号引用、翻译出来的直接引用。
运行期间可将新的常量放入池中，如 String 类的 intern() 方法。

#### 直接内存

**不是虚拟机运行时数据区的一部分**。
(为了避免在Java 堆和Native 堆中来回复制数据，提高性能,)NIO类可以使用 Native 函数库直接分配堆外内存，然后通过一个存储在Java堆中的 DirectByteBuffer 对象作为这块内存的引用进行操作。
可抛出 OutOfMemoryError异常

### 虚拟机中的对象

#### 如何创建

1. 虚拟机遇到一条new指令时，首先将去检查这个指令的参数是否能在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已被加载、解析和初始化过。如果没有，那必须先执行相应的**类加载**过程。
2. 在类加载检查通过后，虚拟机将为新生对象分配内存。对象所需内存的大小在类加载完成后便可完全确定，为对象分配空间的任务等同于把一块确定大小的内存从Java堆中划分出来。

- 如何划分可用空间（分配方式）

假设Java堆中内存是绝对规整的，所有用过的内存都放在一边，空闲的内存放在另一边，中间放着一个指针作为分界点的指示器，那所分配内存就仅仅是把那个指针向空闲空间那边挪动与对象大小相等的距离，这种分配方式称为“**指针碰撞**”（Bump the Pointer）。
如果Java堆中的内存并不是规整的，已使用的内存和空闲内存相互交错，那就没有办法简单地进行指针碰撞了，虚拟机就必须维护一个列表，记录上哪些内存块是可用的，在分配的时候从列表中找到一块足够大的空间划分给对象实例，并更新列表上的记录，这种分配方式称为“**空闲列表**”（Free List）。
使用Serial、ParNew等带Compact过程的收集器时，采用的分配方式是指针碰撞;
使用CMS这种基于Mark-Sweep算法的收集器时，通常采用空闲列表。

- 并发情况下的线程安全

一种是对分配空间的动作进行同步处理——实际上虚拟机采用CAS配上失败重试的方式保证更新操作的原子性;
另一种是把内存分配的动作按照线程划分在不同的空间之中进行，即每个线程在Java堆中预先分配一小块内存，称为本地线程分配缓冲（Thread Local Allocation Buffer，TLAB）。那个线程要分配内存，就在哪个线程的TLAB上分配，只有TLAB用完并分配新的TLAB时，才需要同步锁定。

3. 内存分配完成后，虚拟机需要将分配到的内存空间都初始化为零值（不包括对象头），如果使用TLAB，这一工作过程也可以提前至TLAB分配时进行。
4. 对对象进行设置，例如这个对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码、对象的GC分代年龄等信息。这些信息存放在对象的对象头（Object Header）之中。
5. 执行<init>方法。

#### 如何布局

对象在内存中的布局可以分为3块区域：**对象头**（Header）、**实例数据**（Instance Data）和 **对齐填充**（Padding）。

HotSpot虚拟机的**对象头**包括两部分：
存储对象自身的运行时数据（Mark Word），
类型指针。

如果对象是一个Java 数组，那在对象头中还必须有一块用于记录数组长度的数据。

**实例数据**是对象真正存储的有效信息，也是在程序代码中所定义的各种类型的字段内容。从父类继承下来的和子类定义的，都需要记录下来。这部分的存储顺序会受到虚拟机分配策略（FieldsAllocationStyle）和字段在Java源码中定义顺序的影响。HotSpot虚拟机默认的分配策略为longs/doubles、ints、shorts/chars、bytes/booleans、oops（Ordinary Object Pointers），从分配策略中可以看出，相同宽度的字段总是被分配到一起。在这一前提下，在父类中定义的变量会出现在子类之前。如果CompactFields参数值为true，那么子类中较窄的变量也可能会插入到父类变量的空隙。

**对齐填充**并不是必然存在的，仅仅是作为占位符。由于HotSpot VM要求对象起始地址必须是8字节的整数倍，即对象的大小必须是8字节的整数倍，而对象头部分正好是8字节的倍数（1倍或者2倍），因此，当对象实例数据部分没有对齐时，就需要通过对齐填充来补全。

#### 如何访问定位

- 通过句柄访问对象

堆中将会划分出一块内存来作为句柄池，reference中存储的是对象的句柄地址，而句柄中包含了对象实例数据和类型数据各自的具体地址信息。
使用句柄的好处是：reference中存储的句柄地址很稳定，在对象被移动时只会改变句柄中的实例数据指针，而reference本身不需要更新。

- 通过直接指针访问对象(HotSpot VM)

reference中存储的是对象地址，对象实例数据中放置访问类型数据的相关信息。

![](/images/jvm/access-object-by-handler.jpg)


通过句柄访问对象

![](/images/jvm/access-object-by-direct-pointer.jpg)


通过直接指针访问对象



#### 准确式内存管理

Exact Memory Management,也可以叫 Non-Conservative/Accurate Memory Management)而得名，即虚拟机可以知道内存中某个位置的数据具体是什么类型。譬如内存中有一个32位的整数123456，它到底是一个reference 类型指向123456的内存地址还是一个数值为123456的整数，虚拟机将有能力分辨出来，这样才能在 GC 时准确判断堆上的数据是否还可能被使用。HotSpot VM 就是采用的准确式内存管理。

## CH3 垃圾收集器与内存分配策略

### 垃圾回收器

### 哪些需要GC回收

程序计数器、虚拟机栈、本地方法栈的内存分配和回收都具备确定性，在方法结束或者线程结束时，内存自然就跟着回收了。
**Java堆和方法区**的内存分配和回收是动态的。一个接口中的多个实现类需要的内存可能不一样，一个方法中的多个分支需要的内存也可能不一样，我们只有在运行期才能知道会创建哪些对象。
垃圾收集器关注的是Java堆和方法区。

#### 什么时候回收

##### Java堆的回收时机

- 引用计数算法（Reference Counting）

给对象中添加一个引用计数器，每当有一个地方引用它时，计数器值就加1;当引用失效时，计数值就减1;任何时候计数器值为0的对象就是不可能再被使用的。
对象之间相互循环引用会出问题。

- 可达性分析算法（Reachability Analysis）

通过一系列的称为“GC Roots”的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链（Reference Chain），当一个对象到GC Roots没有任何引用链相连，即从GC Roots到这个对象不可达时，则证明此对象是不可用的、可回收的。

可作为GC Roots的对象有：

    - 虚拟机栈（栈帧中的本地变量表）中引用的对象
    - 方法区中类静态属性引用的对象
    - 方法区中常量引用的对象
    - 本地方法栈中JNI（即Native方法）引用的对象

##### 方法区的回收时机

方法区的主要回收内容为：废弃常量和无用的类

同时满足下面3个条件的类才是“无用的类”：

- 该类的所有实例都已经被回收，Java堆中不存在该类的任何实例
- 加载该类的ClassLoader已经被回收
- 该类对应的java.lang.Class对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法



#### 怎么回收（垃圾收集算法）

##### 标记-清除（Mark-Sweep）算法

非后台回收算法。该算法从GC Roots对象出发，遍历每一个引用去寻找所有需要回收的对象，对每个需要回收的对象都进行标记。标记结束之后，开始清理工作，被标记的对象都会被释放掉。会产生大量内存碎片。

##### 停止-复制（stop-and-copy）算法

非后台回收算法。将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将所有存活的对象从当前堆复制到另外一个堆，没被复制的死对象则全部是垃圾，存活对象被复制到新堆之后全部紧密排列，就可以直接分配新空间了。适用于存活对象少。
新生代的对象98%是朝生夕灭，所以不需要严格按照1：1的比例来划分内存空间，而是将内存分为一块较大的 Eden 空间和两块较小的 Survior 空间，每次使用 Eden 和其中一块 Survivor。当回收时，将 Eden 和 Survivor 中还存活的对象一次性地复制到另外一块 Survivor 空间上，最后清理掉 Eden 和刚才用过的 Survivor 空间。HotSpot 虚拟机默认的 Eden 和 Survivor 的大小比例为 8：1。当 Survivor 空间不够用时，需依赖老年代进行分配担保（Handle Promotion）。 

缺陷：

1. 得有两个堆，然后得在这两个分离的堆之间来回倒腾，从而得维护比实际需要多一倍的空间。
2. 程序进入稳定状态之后，可能只会产生少量垃圾，甚至没有垃圾。尽管如此，复制式回收器仍然会将所有内存自一处复制到另一处，这很浪费。

##### 标记-整理（Mark-Compact）算法

标记过程同标记-清除，后续是将所有存活对象向内存区域的一端移动，然后清理掉端边界以外的内存。适用于存活对象多。

##### 分代收集（Generational Collection）算法

根据对象存活周期的不同将内存分为新生代和老年代，这样就可以根据各个年代的特点采用最适当的收集算法。在**新生代**中，每次垃圾收集时都会发现有大量对象死去，只有少量存活，因此选用**停止复制**算法;而老年代中因为对象存活率高、没有额外空间对它进行分配担保，就必须使用“标记-清除”算法或“标记-整理”算法来进行回收。

### 内存分配与回收策略

对象的内存分配，就是在堆上分配（但也可能经过 JIT 编译后被拆散为标量类型并间接地栈上分配），对象主要分配在新生代的 Eden 区，如果启动了本地线程分配缓冲，将按线程优先在 TLAB 上分配。少数情况下也可能会直接分配在老年代中。具体的分配规则是不固定的，普遍的规则如下：

- 对象优先在 Eden 区分配。
  当 Eden 区没有足够空间进行分配时，VM将发起一次 Minor GC（指发生在新生代的GC） 。
  这样做的目的是，Java对象大多数朝生夕灭，而 Minor GC 非常频繁，回收速度快。
- 大对象直接进入老年代。
  大对象是指需要大量连续内存空间的 Java 对象，典型的有很长的字符串和数组。
  这样做的目的是，防止大对象放在新生代，使得在 Eden 区和两个 Survivor 区之间发生大量的内存复制，影响回收速度。
- 长期存活的对象将进入老年代
  虚拟机给每个对象定义一个对象年龄计数器，每活过一次 Mirror GC 则年龄加1,达到一定阈值就从 Survivor 区晋升到老年代。
  这样做的目的是，按照分代收集算法的要求，存活率较高的对象应进入老年代。
- 动态对象年龄判定
  如果在Survivor 空间中相同年龄所有对象大小的总和大于 Survivor 空间的一半，年龄大于等于该年龄的对象直接进入老年代。
- 空间分配担保
  只要老年代的连续空间大于新生代对象总大小或者历次晋升的平均大小，就会进行 Mirror GC,否则将进行 Full GC。（JDK 6 Update 24 之后的规则）
  这样做的目的是，防止老年代的空间不足以完成分配担保。

### 实践

使用Visual VM来获取虚拟机运行时内部的信息。

[Visual VM下载地址][1]

[安装指南][2]



## 第三部分 虚拟机执行子系统

## CH6 类文件结构 



## CH7 虚拟机类加载机制



### 类加载机制

> 虚拟机把描述类的数据从Class文件加载到内存，并对数据进行校验，解析和初始化，最终形成可以被虚拟机直接使用的 Java 类型。



### 类的生命周期

 

类从被加载到虚拟机内存中开始，到卸载出内存为止，它的整个生命周期包括：加载（Loading）、验证（Verification）、准备（Preparation）、解析（Resolution）、初始化（Initialization）、使用（Using）、卸载（Unloading）7个阶段。其中验证、准备、解析3部分统称为连接（Linking）。

![类的生命周期](/images/jvm/class-lifetime.jpg)



### 类加载的过程



包括了加载、验证、准备、解析、初始化五个阶段：

#### 加载



1. 通过一个类的全限定名获取类的二进制字节流。字节流来源可以是Zip包（Jar）、网络（Applet）、运行时生成（动态代理）、文件（JSP文件）、数据库等。
2. 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。
3. 在 内存中生成一个代表这个类的 java.lang.Class 对象，作为方法区中这个类的各种数据的访问入口。



#### 验证



为了确保Class文件中的字节流包含的信息符合当前虚拟机的要求，不会危害虚拟机自身的安全。完成以下四个阶段的验证：文件格式的验证、元数据的验证、字节码验证和符号引用验证。



#### 准备



准备阶段是正式为类变量分配内存并**设置类变量初始值**的阶段，这些内存都将在方法区中分配。



```java
public static int value=123;//在准备阶段value初始值为0 ,在初始化阶段才会变为123 
public static final int value=123;//在准备阶段value赋值为123 
```



思考：

为什么类变量可以不设初始值，而局部变量不设初始值则编译时报错？

准备阶段会为类变量设默认零值，而如果局部变量必须由我们设初始值，因为局部变量不存在准备阶段。



![基本数据类型的初始值](/images/jvm/primitive-type-default-value.jpg)



#### 解析



解析阶段是虚拟机将常量池中的符号引用替换为直接引用的过程



#### 初始化



是根据程序员通过程序指定的主观计划去**初始化类变量和其他资源**，也就是**执行类构造器`<clinit>`方法**的过程。

`<clinit>`方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块中的语句合并产生的。

`<clinit>`方法与类的构造函数（即实例构造器<init>方法）不是同一个概念。

`<clinit>`方法在多线程环境下也只会执行一次，在同一个类加载器下，一个类型只会初始化一次。



### 类加载器



实现“通过类的权限定名获取该类的二进制字节流”这个动作的代码模块叫做类加载器。



```java
//比较两个类是否相等，包括Class对象的 equal()方法、isAssignableFrom()方法、isInstance()方法、instanceOf 关键字的返回结果
if(同一个Class文件 && 被同一个虚拟机加载 && 被同一个类加载器加载)｛
	相等；
｝else｛
  	不相等；
｝	
```



### 类加载器的分类



主要有四种类加载器:

- 启动类加载器(Bootstrap ClassLoader):用来加载java核心类库（如rt.jar），无法被java程序直接引用。通常使用 C/C++ 语言实现，是虚拟机自身的一部分。负责加载指定路径中且文件名能被虚拟机识别的类库到虚拟机内存中，指定路径指 <JAVA_HOME>\lib 目录或 -Xbootclasspath 参数指定的路径。
- 其他的类加载器：由 Java 语言实现，独立于虚拟机外部，并且都继承自抽象类 java.lang.ClassLoader，Java 开发者可以直接使用这些类加载器。可分为以下3种：

  1. 扩展类加载器（Extensions ClassLoader）:用来加载 Java 的扩展库。它的实现负责加载指定路径中的所有类库。指定路径指 <JAVA_HOME>\lib\ext 目录或 java.ext.dirs 系统变量指定的路径中。
  2. 应用程序类加载器（Application ClassLoader）：用来加载 用户类路径（ClassPath）上所指定的类库，如果没有自定义的类加载器，则这个加载器就是默认的类加载器，一般来说，Java 应用的类都是由它来完成加载的。可以通过 ClassLoader.getSystemClassLoader()来获取它。也称为系统类加载器（System ClassLoader）。
  3. 自定义类加载器（User ClassLoader）：通过继承 java.lang.ClassLoader类的方式实现




### 双亲委派模型（Parents Delegation Model）



> 当一个类加载器收到类加载请求时，首先不会自己去尝试加载这个类，而是将其委派给父类加载器去完成，每一层的类加载器都是如此，因此所有的加载请求最终都应该传送到顶层的启动类加载器中。当父类加载器不能加载时，反馈给子类加载器，由子类加载器去尝试加载。



 这里类加载器之间的父子关系一般不是以继承（Inheritance）的关系来实现，而是以组合（Composition）的关系来复用父加载器的代码。



 ![parents-delegation-model](/images/jvm/parents-delegation-model.jpg)



## CH8 虚拟机字节码执行引擎



执行引擎在执行Java 代码时可能有解释执行（通过解释器执行）和编译执行（通过即时编译器产生本地代码执行）两种方式。



### 运行时栈帧结构



栈帧（Stack Frame）是用于支持虚拟机进行方法调用和方法执行的数据结构，它是虚拟机运行时数据区中的虚拟机栈的栈元素。栈帧存储了方法的局部变量表、操作数栈、动态连接和方法返回地址等信息。每一个方法从调用开始到执行完成的过程，都对应着一个栈帧在虚拟机栈里从入栈到出栈的过程。

栈帧的概念结构如下图。



![stack-frame-struct](/images/jvm/stack-frame.jpg)



#### 局部变量表 ####



局部变量表(Local Variable Table)是一组变量值存储空间，用于存放方法参数和方法内部定义的局部变量。在 Java 程序编译为 Class 文件时，就已在方法的 Code 属性的max_locals 数据项中确定了所需的局部变量表的最大容量。

局部变量表的容量以变量槽(Variable Slot)为最小单位，一个Slot 可以存放一个32位以内的数据类型。虚拟机按照Slot的索引值来访问对应的变量。局部变量表中第0位索引的Slot默认是用于传递方法所属对象实例的引用，即 this 引用。

不使用的对象应手动赋值为 null。原因在于，某些极端情形下，不再使用的变量占用的Slot未被其他变量复用，未对局部变量表有新的读写操作，作为GC Roots一部分的局部变量表仍保持着对该变量的关联。如果对象占用内存大、此方法的栈帧长时间不能被回收、方法调用次数未达到JIT的编译条件，则可以使用本技巧，如果经过JIT编译，则无需如此。



#### 操作数栈#### 

是一个LIFO栈，编译时写到方法的 Code 属性的max_stacks 数据项中。



### 方法调用



> 不等同于方法执行，方法调用的的唯一任务就是确定被调用方法的版本。



所有方法调用中的目标方法在 Class 文件例都是一个常量池里的符号引用，而不是直接引用（方法在实际运行时内存布局中的入口地址）。



#### 解析调用

指的是方法在编译期就有一个确定的调用版本，并且在运行期也是不可改变的。包括：静态方法、私有方法、final方法，因为它们都不可能通过继承被重写为其他版本。



#### 分派调用



## 第五部分 高效并发



## CH12 Java 内存模型与线程



### java内存模型

java内存模型(JMM)是线程间通信的控制机制.JMM定义了主内存和线程之间抽象关系。线程之间的共享变量存储在主内存（main memory）中，每个线程都有一个私有的本地内存（local memory），本地内存中存储了该线程以读/写共享变量的副本。本地内存是JMM的一个抽象概念，并不真实存在。它涵盖了缓存，写缓冲区，寄存器以及其他的硬件和编译器优化。Java内存模型的抽象示意图如下：



![java memory model](/images/jvm/java-memory-model.png)





从上图来看，线程A与线程B之间如要通信的话，必须要经历下面2个步骤：

1. 首先，线程A把本地内存A中更新过的共享变量刷新到主内存中去。

2. 然后，线程B到主内存中去读取线程A之前已更新过的共享变量。

   ​


写的很好的一篇:[http://www.infoq.com/cn/articles/java-memory-model-1](http://www.infoq.com/cn/articles/java-memory-model-1)




## CH13 线程安全与锁优化

### 相关概念

#### 新生代 GC（Minor GC）

指发生在新生代的垃圾收集动作，因为 Java 对象大多都具备朝生夕灭的特性，所以 Minor GC 非常频繁，一般回收速度也比较快。

#### 老年代 GC（Major GC / Full GC）

指发生在老年代的GC,出现了 Major GC，经常会伴随至少一次的 Minor GC（但非绝对的，在 Parallel Scavenge 收集器的收集策略里就有直接进行 Major GC 的策略选择过程）。Major GC 的速度一般会比 Minor GC 慢10倍以上。




#### Object大小的计算

一个Object的大小计算方法:
一个引用4byte+空Object本身8byte+其它数据类型占据自身大小byte(例如char占用2byte);
然而由于系统分配以8byte为单位，所以每个Object占据的大小必须为8的倍数，比如一个空的Object应该占据4+8=12，也就是说需要占据16byte

|    基本数据类型    | 大小(byte) |
| :----------: | :------: |
| byte/boolean |    1     |
|  short/char  |    2     |
|  int/float   |    4     |
| long/double  |    8     |



[1]: http://visualvm.java.net/download.html
[2]: https://blog.idrsolutions.com/2013/05/setting-up-visualvm-in-under-5-minutes/