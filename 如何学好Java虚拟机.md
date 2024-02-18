# 如何诊断后台服务变慢

## 怎样找到最耗CPU的Java线程

1/根据top命令找到进程pid

2/根据进程pid查找线程 top -Hp pid

3/转换16进制  printf "%x" pid

4/ jstack  根据pid获取线程栈，对比相应的线程id即可

[VJTools](https://gitee.com/vipshop/VJTools/tree/master/vjtop)

## 查看上下文切换数量

​	vmstat  1 10  时间间隔为1，收集10次

​	如果每秒上下文(cs，context switch)切换很高，并且比系统中断高很多(in，system interrupt)，就表明很有可能是因为不合理的多线程调度所导致

## GC日志输出

-verbose:gc 打印 GC 日志

PrintGCDetails 打印详细 GC 日志

PrintGCDateStamps 系统时间，更加可读，PrintGCTimeStamps 是 JVM 启动时间

PrintGCApplicationStoppedTime 打印 STW 时间

PrintTenuringDistribution 打印对象年龄分布

loggc 将以上 GC 内容输出到文件中；注意：将此文件放在单独的磁盘，防止与业务数据产生IO资源竞争

HeapDumpOnOutOfMemoryError  OOM 时 Dump 信息

HeapDumpPath Dump 文件保存路径

ErrorFile 错误日志存放路径

## GC日志可视化

gceasy

GCViewer

## JDK 常用命令

jps：用来显示 Java 进程

jstat -gcutil $pid 1000#每隔1秒输出垃圾回收情况（eden、surivor0、Surivor1、Old、Metaspace占空间，年轻代的回收次数和回收耗时，Full GC 的次数和耗时）

jmap：用来 dump 堆信息

jstack：用来获取调用栈信息

jhsdb：用来查看执行中的内存信息

## 保留现场信息

系统当前网络连接

ss -antp > $DUMP_DIR/ss.dump 2>&1

网络状态统计

netstat -s > $DUMP_DIR/netstat-s.dump 2>&1

网络流量

sar -n DEV 1 2 > $DUMP_DIR/sar-traffic.dump 2>&1

进程资源

lsof -p $PID > $DUMP_DIR/lsof-$PID.dump

CPU 资源

mpstat > $DUMP_DIR/mpstat.dump 2>&1

vmstat 1 3 > $DUMP_DIR/vmstat.dump 2>&1

sar -p ALL  > $DUMP_DIR/sar-cpu.dump  2>&1

uptime > $DUMP_DIR/uptime.dump 2>&1

I/O 资源

iostat -x > $DUMP_DIR/iostat.dump 2>&1

内存问题

free -h > $DUMP_DIR/free.dump 2>&1

其他全局

ps -ef > $DUMP_DIR/ps.dump 2>&1

dmesg > $DUMP_DIR/dmesg.dump 2>&1

sysctl -a > $DUMP_DIR/sysctl.dump 2>&1

dump 堆信息

jmap $PID > $DUMP_DIR/jmap.dump 2>&1

jmap -heap $PID > $DUMP_DIR/jmap-heap.dump 2>&1

jmap -histo $PID > $DUMP_DIR/jmap-histo.dump 2>&1

jmap -dump:format=b,file=$DUMP_DIR/heap.bin $PID > /dev/null  2>&1

JVM 执行栈

jstack $PID > $DUMP_DIR/jstack.dump 2>&1

kill -3 $PID

# 即时编译器优化

将热点代码以方法为单位转换成机器码，直接运行在底层硬件 之上

## 实现

方法计数器

​	甄别热点方法

回边计数器

​	甄别热点循环代码

## 分层编译五个层次

解释执行

执行不带 profiling 的 C1 代码

执行仅带方法调用次数以及循环回边执行次数 profiling 的 C1 代码

执行带所有 profiling 的 C1 代码；

执行 C2 代码

## 相关参数

打印编译发生的细节-XX:+PrintCompilation

调整热点代码门限值-XX:CompileThreshold=N

关闭计数器衰减-XX:-UseCounterDecay

调整 Code Cache 大小-XX:ReservedCodeCacheSize=<SIZE>(注：JIT 就无法继续编译，编译执行会变成解释执行，性能会降低一个数量级。同时，JIT 编译器会一直尝试去优化代码，从而造成了 CPU 占用上升。

指定的编译线程数-XX:CICompilerCount=N

关闭偏斜锁-XX:-UseBiasedLocking

诊断安全点的影响：-XX:+PrintSafepointStatistics -XX:+PrintGCApplicationStoppedTime
在 JDK 9 之后，PrintGCApplicationStoppedTime 已经被移除了，你需要使用“- Xlog:safepoint”之类方式来指定。

## JITWatch

观察 JIT 执行过程的图形化工具(https://github.com/AdoptOpenJDK/jitwatch)

## 逃逸分析

同步锁消除

栈上分配

分离对象或者标量替换

## 方法内联

把方法内代码拷贝、粘贴到调用者的位置

## intrinsic

被@HotSpotIntrinsicCandidate标注的方法，在HotSpot中都有一套高效的实现，该高效实现基于CPU指令，运行时，HotSpot维护的高效实现会替代的源码实现，从而获得更高的效率。

# 运行时数据区域

## 线程私有

程序计数器

当前线程执行的java字节码指令的行号指示器；如果执行java方法，返回虚拟机正在执行的字节码指令的地址；如果指定的是Native方法，计数器的值为null。

java虚拟机栈

方法执行的时候都会创建一个栈帧，存储局部变量、动态链接、操作栈、返回出口等信息。若果线程请求的栈深度大于虚拟机所允许的深度，将抛出StackOverFlow异常。

本地方法栈

为虚拟机使用到的Native方法服务

## 线程共享

### java堆内存

存放生成的对象

### 方法区

存储已经被 JVM 加载的类型信息、常量、静态变量、代码缓存等数据

演进

​	JDK1.7永久代,在堆内

​	JDK1.8元空间（堆外），大小受本机内存大小的限制

### 直接内存

特点

​	常见于 NIO 操作时，用于数据缓冲区

​	分配回收成本较高，但读写性能高

​	不受 JVM 内存回收管理

 分配和回收原理

使用了 Unsafe 对象完成直接内存的分配回收，并且回收需要主动调用 freeMemory 方法

ByteBuffer 的实现类内部，使用了 Cleaner来监测 ByteBuffer 对象

ByteBuffer 对象被垃圾回收，那么就会由 ReferenceHandler 线程通过 Cleaner 的 clean 方法调用 freeMemory 来释放直接内存

# 类的加载

## 双亲委派模式

除了顶层的启动类加载器以外，其余的类加载器，在加载之前，都会委派给它的父加载器进行加载。这样一层层向上传递，直到祖先们都无法胜任，它才会真正的加载

## 默认三种类加载器

Bootstrap Classloader

顶级类加器，任何类的加载行为，都要经它过问。它的作用是加载核心类库，也就是rt.jar、resources.jar、charsets.jar等。-Xbootclasspath 参数从指定的路径加载

Extension ClassLoader

扩展类加载器，主要用于加载 lib/ext 目录下的 jar 包和 .class 文件。同样的，通过系统变量 java.ext.dirs 可以指定这个目录

Application ClassLoader

应用类加载器，有时候也叫作SystemClassLoader。一般用来加载classpath下的其他所有jar包和.class文件

## 类的加载机制

加载

将外部的 .class 文件，加载到 Java 的方法区内

验证

肯定不能任何.class文件都能加载，那样太不安全了，容易受到恶意代码的攻击。验证阶段在虚拟机整个类加载过程中占了很大一部分，不符合规范的将抛出java.lang.VerifyError错误

准备

为一些类变量分配内存，并将其初始化为默认值

注：static int a = 1 ; 准备阶段过后是0； final static int a = 1; 准备阶段后是1；

解析

将符号引用替换为直接引用的过程

初始化

为类变量赋值；执行静态代码块

注：static 语句块，只能访问到定义在 static 语句块之前的变量

JVM 会保证在子类的初始化方法执行之前，父类的初始化方法已经执行完毕

<cinit> 方法和 <init> 方法有什么区别

其中static字段和static代码块，是属于类的，在类的加载的初始化阶段就已经被执行。类信息会被存放在方法区，在同一个类加载器下，这些信息有一份就够了，它对应的是 <cinit> 方法；

 new 一个新对象的时候，都会调用它的构造方法，就是 <init>，用来初始化对象的属性。每次新建对象的时候，都会执行

如何替换 JDK 的类

双亲委派机制，是无法直接在应用中替换JDK的原生类的。但是，有时候又不得不进行一下增强、替换，比如你想要调试一段代码，或者比Java团队早发现了一个BugJava 提供了 endorsed 技术，用于替换这些类。这个目录下的 jar 包，会比 rt.jar 中的文件，优先级更高，可以被最先加载到

# 对象在内存的布局

## 普通对象

markword（默认8字节）

对象的hashcode

GC年龄

是否偏向锁

锁的标志位

class pointer（默认压缩占4字节）

instance data实例数据

padding（对齐填充，保证能被8整除）

## 数组

markword

class pointer

instance data

length(长度)

padding

# 引用类型

强引用

只有在和 GC Roots 断绝关系时，才会被回收

软引用

用于维护一些可有可无的对象。在内存足够的时候，软引用对象不会被回收，只有在内存不足时，系统则会回收软引用对象

弱引用

当 JVM 进行垃圾回收时，无论内存是否充足，都会回收被弱引用关联的对象

虚引用

虚引用必须和引用队列（ReferenceQueue）联合使用

主要用来跟踪对象被垃圾回收的活动

# 垃圾收集器

## ParNew

由多条 GC 线程并行地进行垃圾清理。清理过程依然要停止用户线程

## Parallel Scavenge 

另一个多线程版本的垃圾回收器，追求 CPU 吞吐量，能够在较短时间内完成指定任务，适合没有交互的后台计算

## Serial Old

与年轻代的 Serial 垃圾收集器对应，都是单线程版本。老年代的 Old Serial，使用标记-整理算法。

## Parallel Old

是 Parallel Scavenge 的老年代版本，追求 CPU 吞吐量。

## CMS

以获取最短 GC 停顿时间为目标的收集器，垃圾收集时用户线程和GC线程能够并发执行

工作过程

初始标记

只标记直接关联 GC root 的对象

并发标记

用于标记所有可达的对象，持续时间较长，而且和用户线程并行

并发预清理

目的是为了让重新标记阶段的 STW 尽可能短

并发可取消的预清理

最终标记

暂停用户线程，完成老年代中所有存活对象的标记

并发清除

删掉不可达的对象，并回收它们的空间

优点

并发收集

低停顿

缺点

产生内存碎片

占用CPU资源

出现Concurrent Mode Failure

无法处理浮动垃圾

解决内存碎片

UseCMSCompactAtFullCollection（默认开启）

在要进行 Full GC 的时候，进行内存碎片整理。内存整理的过程是无法并发的，所以停顿时间会变长

CMSFullGCsBeforeCompaction

每隔多少次不压缩的 Full GC 后，执行一次带压缩的 Full GC。默认值为 0，表示每次进入 Full GC 时都进行碎片整理

## G1垃圾收集器

也分为Eden区、Survivor、Old，只不过它们在内存上不是连续的

把内存划分为大小固定的Region,它的数值是在1M到32M字节之间的2 的幂值数，每个region的角色并不是一成不变的

Humongous Region,存放大对象如果对象的大小超过Region 50%，就在此region分配

引入RSet概念，Region初始化时，会初始化一个RSet，该集合用来记录其它Region指向该Region中对象的引用，每个Region默认按照512Kb划分成多个Card，所以RSet需要记录的东西应该是xxRegion的xxCard

不会出现内存碎片

回收过程

young gc

在Eden耗尽时会触发年轻代的垃圾回收

将Eden存活的对象复制到幸存区，幸存区存活的对象如果GC年龄超过阈值，复制到老年代，否则复制其他的幸存区

mixed gc

当使用内存超过一定阈值就会触发

以region为单位进行收集，年轻代和老年代都会收集

回收过程类似CMS

full gc

serial old gc 

stop the world

三色标记-CMS和G1的并发标记算法

白色：未被标记的对象

灰色：自身被标记，成员变量未被标记

黑色：自身和成员变量均已标记完成

漏标

在remark过程中，黑色指向了白色，如果不对黑色重新扫描，就会出现漏标

在并发标记过程中，Mutator删除了从灰色对象到白色对象的引用，会产生漏标

解决漏标的方法

incremental update 增量更新

关注引用的增加，把黑色的重新标记成灰色的，重新扫描一次（CMS）

SATB snaphot at the beginning

关注引用的删除，当一个对象（灰色）对另一个对象(白色)的引用消失时，要把这个引用推到GC的堆栈，保证消失的对象还能被扫描到

G1使用此解决方案，搭配RSet，效率高不需要扫描整个堆查找指向白色的引用

## ZGC垃圾收集器

停顿时间不会超过 10ms，停顿时间不会随着堆的增大而增大

可支持几百M，甚至几T的堆大小

没有新生代和老生代的概念

采用的着色指针技术，利用指针中多余的信息位来实现着色标记

使用了读屏障来解决 GC 线程和应用线程可能存在的并发（修改对象状态的）问题

# 分代

## 原因

大部分对象的生命周期都很短,把那些朝生暮死的对象集中分配到一起，把不容易消亡的对象分配到一起，对于不容易死亡的对象我们就可以设置较短的垃圾收集频率，这样就能消耗更少的资源来实现更理想的功能

## 年轻代

伊甸园空间（Eden)和两个幸存者空间（Survivor from、to)，用于存储刚刚创建的对象

参数 -XX:SurvivorRatio配置eden、from、to空间占比，默认8:1:1

垃圾回收之后存活的对象少，采用复制算法，效率高，但有空间的浪费

## 老年代

长时间存活的对象占用的区域

一般使用“标记-清除”、“标记-整理”算法。老年代存活的对象比较长，占用的空间比较大，不适合复制

## 提升

每当发生一次 Minor GC，存活对象年龄都会加 1。直到达到一定的阈值，就会把这些对象给提升到老年代；通过参数 ‐XX:+MaxTenuringThreshold 进行配置，最大值是 15。

分配担保，年轻代发生垃圾回收之后，存活的对象在幸存区不足的情况下，对象直接在老年代分配

大对象直接在老年代分配，超出某个大小的对象将直接在老年代分配。通过参数 -XX:PretenureSizeThreshold 进行配置。默认为 0，表明全对象优先在Eden区进行分配。

动态对象年龄判定，如果幸存区中相同年龄对象大小的和，大于幸存区的一半，大于或等于 age 的对象将会直接进入老年代

# 垃圾收集算法

标记-清除

根据 GC Roots 遍历所有的可达对象，这个过程，就叫作标记，把未被标记的对象回收掉；容易产生内存碎片

标记-复制

提供一个对等的内存空间，将存活的对象复制过去，然后清除原内存空间；解决了碎片问题，但是浪费了一半的内存空间

标记-整理

移动所有存活的对象，且按照内存地址顺序依次排列，然后将末端内存地址以后的内存全部回收；解决了内存碎片的问题。

# 怎么进行对象分配

是否可以栈上分配（逃逸分析、标量替换）

是不是大对象？大对象直接老年代分配

是否TLAB分配，在Eden区分配

标记-清除时，空闲列表分配

标记-复制时，指针碰撞分配

一个Object对象占多少字节

java -XX:+PrintCommandLineFlags -version

默认启用压缩，markword占用8字节， class pointer占用4字节，对齐填充4字节，总共16字节；不启用压缩或者压缩失效，占用16字节

# 如何判断对象已死

引用计数法

当一个对象被引用时加1，当引用失效时减1，引用计数为0时，此对象可回收

缺点：无法回收互相引用的对象

可达性分析算法

当一个对象到GC ROOTS没有任何引用链相连，此对象可回收

# 何为GC ROOTS

1、Java虚拟机栈中的引用的对象 

2、方法区中的类静态属性引用的对象

3、本地方法栈中 JNI的引用的对象

# 对象访问

句柄

稳定，对象被移动只要修改句柄中的地址

直接指针

访问速度快，节省一次指针定位的开销

# 评价 GC算法的标准

1、分配的效率：主要考察在创建对象时，申请空闲内存的效率；
2、回收的效率：它是指回收垃圾时的效率；
3、是否产生内存碎片：在讲解 malloc 的时候，我们讲过内存碎片。碎片是指活跃对象之
间存在空闲内存，但这一部分内存又不能被有效利用。比如内存里有两块不连续的 16
字节空闲空间，此时分配器要申请一块 32 字节的空间，虽然总的空闲空间也是 32 字
节，但由于它们不连续，不能满足分配器的这次申请。这就是碎片空间；
4、空间利用率：这里主要是衡量堆空间是否能被有效利用。比如基于复制的算法无论何时
都会保持一部分内存是空闲的，那么它的空间利用率就无法达到 100%，这是由算法本
身决定的；
5、是否停顿：Collector 在整理内存的时候会存在搬移对象的情况，因为修改指针是一种非
常敏感的操作，有时候它会要求 Mutator（业务线程，可能改变对象的存活和死亡状态） 停止工作。是否需要 Mutator 停顿，以及停
顿时长是多少，是否会影响业务的正常响应等。停顿时长在某些情况下是一个关键性指
标；
6、实现的复杂度：有些算法虽然看上去很美妙，但因为其实现起来太复杂，代码难以维
护，所以无法真正地商用落地。这也会影响到 GC 算法的选择。

# 引用计数算法

## 概念

如果它被引用的次数大于 0，那它就是一个活跃对象；如果它被引用的次数为 0，那它就是一个垃圾对象。

为了记录一个对象有没有被其他对象引用，可以在每个对象的头上添加一个叫“计数器”的东西，用来记录有多少其他对象引用了它

虚拟机在执行引用赋值语句时，插入write barrier，修改计数信息，新引用指向的对象的计数+1，原先引用的对象计数-1

## 优点

1、可以立即回收；每个对象在被引用次数为0的时候，是立即可以知道的

2、没有暂停时间；对象的回收不需要另外线程

## 缺点

1、每次赋值都有额外的计算

2、会有链式回收的情况

3、循环引用

在使用引用计数算法进行内存管理的语言中，Python 在引用计数之外，另外引入了三色标记算法，保证了在出现循环引用的情况下，垃圾对象也能被正常回收

Java 对象都是在 Java 堆中创建的，它们之间的相互引用关系可以使用图结构来表示。找
出堆中活跃对象的过程就是在堆中对 Java 对象进行遍历的过程。遍历算法的起点是根集
合

所有不在堆中，而指向堆里的引用都是根引用，根引用的集合就是根集合。根集合是很多
基于可达性分析的 GC 算法遍历的起点

# 可达性分析

要想识别一个对象是不是垃圾，首先需要找到“根”引用集合。所谓根引用指的是不在堆中，但指向堆中的引用。根引用包括了栈上的引用、全局变量等

基本思想是把对象之间的引用关系构成一张图，这样我们就可以从根出发，开始对图进行遍历。能够遍历到的对象，是存在被引用的情况的对象，就是活跃对象；不能被访问到的，就是垃圾对象

对图进行遍历有两种算法，分别是深度优先遍历（Depth First Search，DFS）和广度优先遍历（Breadth First Search，BFS）

使用深度优先搜索算法对活跃对象进行遍历，在遍历的同时就把活跃对象复制到 To 空间中去了。活跃对象有可能被重复访问，可以使用forwarding 指针来解决这个问题

深度优先搜索的递归写法实现简单，但效率差，非递归写法需要额外的辅助数据结构，
但它能使业务线程运行时有更好的空间局部性，有利于提高缓存命中率

广度优先搜索的实现可以借助 To 空间做为辅助队列，节约空间。但不利于业务线程的
缓存命中率

# 基于复制的垃圾回收算法

基本思想

把某个空间里的活跃对象复制到其他空间，把原来的空间全部清空，这就相当于是把活跃的对象从一个空间搬到新的空间。把原空间称为 From 空间，把新的目标空间称为 To 空间

最基础的 copy 算法

就是把程序运行的堆分成大小相同的两半，一半称为 From 空间，一半称为 To 空间。当创建新的对象时，都是在 From 空间里进行内存的分配。等 From空间满了以后，垃圾回收器就会把活跃对象复制到 To 空间，把原来的 From 空间全部清空。然后再把这两个空间交换，也就是说 To 空间变成下一轮的 From 空间，现在的 From空间变成 To 空间

比较适合管理短生命周期对象

通过碰撞指针的方式分配内存

## Scavenge 算法

每次回收中能存活下来的对象占总体的比例都比较小。根据这个特点，把 To 空间设置得小一点，来提升空间的利用率

Hotspot 在实现 copy 算法时做了一些改进。它将 From 空间称为 Eden 空间，To 空间在算法实现中则被分成 S0 和 S1 两部分，这样 To 空间的浪费就可以减少了

Scavenge 算法是简单 copy 算法的一种改进。配置 Survivor 空间的大小是 JVM GC 中的重要参数，例如：-XX:SurvivorRatio=8，代表 Eden:S0:S1=8:1:1

在复制的过程中，对象的地址会发生变化，为了保证引用的正确性，引入了forwarding 指针。也就是说每个对象的头部引入一个新的域，叫做forwarding。正常状态下，它的值是 NULL，如果一个对象被拷贝到新的空间里以后，就把它的新地址设到旧对象forwarding 指针里

## 特点

对象之间紧密排列，中间没有空隙，也就是没有内存碎片；
分配内存的效率非常高。因为每次分配对象都是把指针简单后移即可，操作非常少，所以效率高；
回收的效率取决于存活对象的多少，如果存活的对象少，则回收效率高。


内存利用率并不高。

copy 算法需要搬移对象，所以需要业务线程暂停

# Mark-Sweep 算法

## 基本思想

Mark-Sweep 算法由 Mark 和 Sweep 两个阶段组成。在 Mark 阶段，垃圾回收器遍历活跃对象，将它们标记为存活。在 Sweep 阶段，回收器遍历整个堆，然后将未被标记的区域回收

## 分配对象

用一个链表将所有的空闲空间维护起来，这个链表就是空闲链表（freelist）。当内存管理器需要申请内存空间时，便向这个链表查询，如果找到了合适大小的空闲块，就把空闲块分配出去，同时将它从空闲链表中移除

当一个空闲块分配给一个新对象后，如果剩余空间大于 24 字节，便将剩余的空间加入到空
闲链表，当剩余空间不足 24 字节的话，就做为碎片空间填充一些无效值

## Mark 阶段

从根引用出发，根据引用关系进行遍历，所有可以遍历到的对象就是活跃对象，一边遍历，一边将哪些对象是活跃的记录下来。遍历的方法采用深度优先和广度优先两种策略

## Sweep 阶段

要做的事情就是把非活跃对象，也就是垃圾对象的空间回收起来，重新将它们放回空闲链表中。具体做法就是按照空间顺序，从头至尾扫描整个空间里的所有对象

Mark-Sweep 算法回收的是垃圾对象，如果垃圾对象比较少，回收阶段所做的事情就比较少。所以它适合于存活对象多，垃圾对象少的情况

# 提前晋升

提前晋升是指分代垃圾回收算法中，年轻代对象的年龄还没有达到晋升的阈值就因为年轻代空间的不足而不得不提前搬进老年代的现象。

提前晋升不仅说明年轻代的压力大，而且会导致老年代空间的快速消耗

可以使用jstat工具判断提前晋升。

如果每一次年轻代GC后，幸存者空间是满的，而老年代空闲空间明显下降，说明一些年轻代中的对象在年龄没有达到阈值就升入了老年代

解决办法

主要是扩大年轻代的空间。一种是扩大堆的大小，如果堆的大小不能再扩大，也可以考虑老年代空间配置得小一些。最好的办法还是尽量减少对象的创建，优化业务逻辑

# 什么是分代垃圾回收

基于对象的生命周期引入了分代垃圾回收算法，它将堆空间划分为年轻代和老年代

对于存活时间比较短的对象，年轻代可以用 Scavenge 算法回收。Java 中的函数创建的对象也会在堆里进行分配，这就导致 Java 中的对象的生命周期都不长

对于存活时间比较长的对象，老年代可以使用 Mark-Sweep 算法

通过在对象的头部记录一个名为 age 的变量，可以区分哪些是存活时间长的对象。

Scavenge GC 每做一次，就把存活的对象往Survivor 空间中复制一次，就相应把这个对象的 age加1。当对象的age值到达一个上限以后，就可以将它搬移到老年代

# 如何解决跨代引用

## 1、记录集

为了避免对年轻代回收时，对老年代对象进行遍历，可以把跨代引用记录下来，记录跨代引用的集合就是记录集。记录集中存放引用年轻代对象的老年代对象

对记录集的维护发生在对象的引用发生变化时，同样通过写屏障实现

1、被引用的对象是否在年轻代

2、发出引用的对象，也就是引用者，是否在老年代；

以上两点都满足，就说明产生了跨代引用

3、检查记录集中是否已经包含了引用者。

如果以上三点都满足，将引用者加入记录集中

还有一种情况可能产生跨代引用，那就是晋升

随着对象的增多，记录集会变得很大，而且每次对老年代做 GC，正确地维护记录集也是一件复杂的事情

## 2、Card table

Card table借鉴了位图的思路， Hotspot 的分代垃圾回收将老年代空间的 512bytes 映射为一个 byte，当该 byte 置位，则代表这 512bytes 的堆空间中包含指向年轻代的引用；如果未置位，就代表不包含

Card table的置位也是通过写屏障实现的

Cardtable不仅可以实现跨代引用的功能，还实现了标记灰色节点的功能

# 什么是CMS收集器

CMS 的全称是 Mostly Concurrent Mark and Sweep Garbage Collector（主要并发­标记­清除­垃圾收集器），它在年轻代使用复制算法，而对老年代使用标记-清除算法

老年代并发的， 垃圾回收和应用程序同时运行，降低STW的时间

CMS既然是MarkSweep，就一定会有碎片化的问题，碎片到达一定程度，CMS的老年代分配对象分配不下的时候，使用SerialOld 进行老年代回收

CMS用一个链表将所有的空闲空间维护起来，这个链表就是空闲链表（freelist）。当内存管理器需要申请内存空间时，便向这个链表查询

为了减少 GC 停顿，我们可以在做 GC Mark 的时候，让业务线程不要停下来。这意味着GC Mark 和业务线程在同时工作，这就是并发（Concurrent）的 GC 算法

并发 GC 是指 GC 线程和业务线程同时工作，并行是指多个 GC 线程同时工作

三色出现之前

第一种情况：GC线程回收时A对象不可达，标记为非活跃，业务线程在执行时将B对象的属性引用又指向了A对象，而GC线程是无感知的，造成误删除，影响了程序的正常运行

第二种情况：GC线程回收时A对象可达标记为活跃对象，业务线程在执行时将对A对象的引用删除了，变为了非活跃对象，而GC线程是无感知的，变为了浮动垃圾

## 三色标记

白色：还未搜索的对象；
灰色：已经搜索，但尚未扩展的对象；
黑色：已经搜索，也完成扩展的对象

并发标记中最严重的问题就是漏标。如果一个对象是活跃对象，但它没有被标记，这就是漏标。这就会出现活跃对象被回收的情况

黑色对象引用了白色对象，而白色对象又没有其他机会再被访问到，所以白色对象就被漏标了

解决漏标问题，还是要从写屏障入手

1、往前一步，将白色对象直接标灰

​	把白色对象直接标记，然后放入队列中待扩展。但是如果被标记之后，其他对像对它的引用消失了，被标记的对象实际上是被多标了，变成了浮动垃圾。如果对象引用修改频繁，会出现大量的浮动垃圾

2、可以通过后退一步，把黑色对象重新扩展一次，也就是说黑色结点变成灰色

## CMS执行过程

### 初始标记阶段

只标记直接关联 GC root 的对象，一般都采用 Stop The World 的做法，一般不遍历年轻代对象，也就是不关注从年轻代指向老年代的引用

### 并发标记阶段

如果在并发标记的过程中，业务线程修改了对象之间的引用关系。CMS 采用的办法是：在 write barrier 中，只要一个对象 A 引用了另外一个对象 B，不管 A 和 B 是什么颜色的，都将 A 所对应的 card 置位。

当一轮标记完成以后，如果还有置位的 card，那么垃圾回收器就会开启新一轮并发标记。新的一轮标记，会从上一轮置位的 card 所对应的对象开始进行遍历，遍历完成后再把card 全部清零，所以这样的一轮并发标记也被称为预清理

如果每一轮都有 card 置位，应该怎么办呢？CMS 也会在预清理达到一定次数以后停止，进入重标记阶段

### 重标记阶段

通常 CMS 会尝试在年轻代尽可能空的情况下运行 Final Remark 阶段，以免接连多次发生 STW 事件

重标记的作用是遍历新生代和 card 置位的对象，对老年代对象做一次重新标记

对年轻代对象进行一次遍历，找出年轻代对老年代的引用，并且为并发标记阶段扫尾

在重标记之前，进行一次年轻代 GC，这样可以减少年轻代中的对象数量，减少重标记的停顿时间。这个功能可以使用参数 -XX:+CMSScavengeBeforeReMark 来打开

### 并发清除阶段

会把垃圾对象归还给 freelist，只要注意好 freelist 的并发访问，实现垃圾回收线程和业务线程并发执行是简单的

### 最终清理阶段

清理垃圾回收所使用的资源。为下一次 GC 执行做准备

# 什么是G1算法

## 简介

G1也是一个分区的垃圾回收算法

G1的老年代和年轻代不再是一块连续的空间，整个堆被划分成若干个大小相同的Region

Region的类型有Eden、Survivor、Old、Humongous 四种

Humongous 是用来存放大对象的，如果一个对象的大小大于一个Region的 50%，就认为这个对象是一个大对象。为了防止大对象的频繁拷贝，我们可以将大对象直接放到 Humongous 中

发生gc时，垃圾最多的小堆区，会被优先收集。这就是 G1 名字的由来

## 如何解决漏标

当灰色对象对白色对象的引用关系消失以后，再将白色对象标记为灰色，即便将来黑色对象对白色对象的引用消失了，也会在当前 GC 周期内被视为活跃对象。也就是说，被标记为灰色的白色对象可能变成浮动垃圾。把这种在删除引用的时候进行维护的屏障叫做 deletion barrier

通过使用deletion barrier，在并发标记阶段，即便对象的全部引用被删除，也会被当做活跃对象来处理。就好像在 GC 开始的瞬间，内存管理器为所有活跃对象做了一个快照一样，即开始时快照（Snapshot At The Beginning，SATB）

在CMS中写屏障的逻辑是由业务线程执行的

在G1中，将对象置灰这个操作往后推迟了，业务线程只需把对象加入一个本地队列中就可以了。每个业务线程都有一个这样的线程本地队列，名字是 SATB 队列。

当本地队列满了之后，就把它交给SATB 队列集合，然后再领取一个空队列当做线程的本地 SATB 队列。GC 线程则会将SATB 队列集合中的对象标记为灰色

## 垃圾回收模式

young GC：只回收年轻代的 Region

并发标记

mixed GC：回收全部的年轻代 Region，并回收部分老年代的 Region

mixed 回收的老年代 Region 是需要进行决策的（Humongous 在回收时也是当做老年代的Region 处理的）。把 mixed GC 中选取的老年代对象 Region 的集合称之为回收集合（CollectionSet，CSet）。

CSet 的选取要素有以下两点：
1、该 Region 的垃圾占比。垃圾占比越高的 Region，被放入 CSet 的优先级就越高，这就是垃圾优先策略（Garbage First）
2、建议的暂停时间。建议的暂停时间由 -XX:MaxGCPauseMillis 指定，G1 会根据这个值来选择合适数量的老年代 Region。

MaxGCPauseMillis 默认是 200ms，不过需要注意的是，MaxGCPauseMillis 设置的越小，选取的老年代 Region就会越少，如果 GC 压力居高不下，就会触发 G1 的 Full GC

参数InitiatingHeapOccupancyPercent（IHOP），它的作用是内存空间达到一定百分比之后，启动并发标记。当然，这更进一步是为了触发mixed GC，以此来回收老年代。如果一个应用老年代对象产生速度较快，可以尝试适当调小 IHOP

## 维护跨区引用

分区回收算法为每个 Region 都引入了记录集（Remembered Set，RSet），每个 Region 都有自己的专属 RSet

和Card table 不同的是，RSet 记录谁引用了我，这种记录集被人们称为 point-in 型的，而 Card table 则记录我引用了谁，这种记录集被称为 point-out 型

对于年轻代的 Region，它的 RSet 只保存了来自老年代的引用，这是因为年轻代的回收是针对所有年轻代 Region 的，没必要画蛇添足。所以说年轻代 Region 的 RSet 有可能是空的。

而对于老年代的 Region 来说，它的 RSet 也只会保存老年代对它的引用

RSet的记录也是通过写屏障实现的，RSet中记录的也是card（dirty card）。业务线程也不是直接将dirty card 放到 RSet 中的。而是在业务线程中引入一个叫做 dirty card queu（DCQ）的队列，在写屏障中，业务线程只需要将 dirty card 放入 DCQ 中

G1 GC 中的 Refine 线程会从DCQ中找到dirtycard经过精细的检查，才会放到RSet中

RSet的三种不同结构：

1、稀疏表是一个哈希表，当 Region被外部引用很少时，就可以将相关的 card放到稀疏表
2、细粒度表则是一个真正的 card table，当 Region 之间的引用比较多时，就可以直接使
用位图来代替哈希表，因为这能加快查找的速度
3、粗粒度表则是一个区的位图，当Region的外部引用很多时，就不用再使用 card table 来进行管理了，在回收Region时，直接将Region的全部对象都遍历一次就可以了

## 活跃对象转移过程

 Evacation（转移） 发生的时机是不确定的，在并发标记阶段也可能发生。所以并发标记要使用一个 BitMap 来记录活跃对象，而 Evacation 也需要使用一个 BitMap 来将活跃的对象进行搬移

G1 维护了两个 BitMap，一个名为 nextBitMap，一个名为prevBitMap。其中，prevBitMap 是用于搬移活跃对象，而 nextBitMap 则用并发标记记录活跃对象

TAMS 指针，是 Top At Mark Start 的缩写。初始时，prevTAMS，nextTAMS 和 top 指针都指向一个分区的开始位置

随着业务线程的执行，top指针不断向后移动。并发标记开始时，nextTAMS记录下当前的 top 指针，并且针对nextTAMS之前的对象进行活跃性扫描，扫描的结果就存放在nextBitMap中

当并发标记结束以后，nextTAMS的值就记录在prevTAMS中，并且nextBitMap 也赋值给 prevBitMap。如果此时发生了 Evacation，则 prevBitMap 已经可用了。如果没有发生 Evacation，那么 nextBitMap 就会清空。在任意时刻开启 Evacation 的话，prevBitMap 总是可用的

在并发标记开始以后，再创建的对象，其实就是 nextTAMS 指针到 top 指针之间的对象，这些对象全部认为是活跃的

# ZGC算法

## 特点

1、停顿时间不会随着堆大小的增加而线性增加。最大停顿时间不超过 10ms 
2、ZGC 和 G1 有很多相似的地方，也是采用复制活跃对象的方式来回收内存，也同样将内存分成若干个区域，回收时也会选择性地先回收部分区域。

3、ZGC 与 G1 的区别在于：它可以做到并发转移（拷贝）对象，并发转移指的是在对象拷贝的过程中，应用线程和 GC 线程可以同时进行

4、为了实现并发转移，ZGC 使用了 readbarrier

5、ZGC 采用了用空间换时间的做法，也就是染色指针（coloredpointer）技术。通过这个技术，ZGC 不仅非常高效地完成了 read barrier 需要完成的工作，而且可以更高效的利用内存

在 64 位系统下， ZGC 就借用了地址的第 42 ~ 45 位作为标记位，第 0 ~41 位共 4T 的地址空间留做堆使用

第 42-45 这4位是标记位，它将地址划分为 Marked0、Marked1、Remapped、Finalizable 四个地址视图

地址视图的巧妙之处就在于，一个在物理内存上存放的对象，被映射在了三个虚拟地址上

有了地址视图之后，我们就可以在一个对象转移之后，修改它的地址视图了，同时还可以维护一张映射表（ forwarding table）。在这个映射表中，key 是旧地址，value 是新地址。当对象再次被访问时，通过插入的 read barrier 来判断对象是否被搬移过。如果forwarding table 中有这个对象，说明当前访问的对象已经转移，read barrier 这时就会将对这个对象的引用直接更改为新地址

## 回收原理

### Mark

ZGC 也不是完全没有 STW 的。在进行初始标记时，它也需要进行短暂的 STW，ZGC 只会扫描 root，之后的标记工作是并发的，所以整个初始标记阶段停顿时间很短

然后根据 root 集合进行并发标记了

在 GC 开始之前，地址视图是 Remapped。那么在 Mark 阶段需要做的事情是，将遍历到的对象地址视图变成 Marked0。

应用线程在并发标记的过程中也会产生新的对象，新分配的对象都认为是活的，它们地址视图也都标记为 Marked0。至此，所有标记为Marked0 的对象都认为是活跃对象，活跃对象会被记录在一张活跃表中

而视图仍旧是 Remapped 的对象，就认为是垃圾

### Relocate

此阶段的主要任务是搬移对象，在经过 Mark 阶段之后，活跃对象的视图为Marked0

搬移工作要做两件事情：
1、选择一块区域，将其中的活跃对象搬移到另一个区域
2、将搬移的对象放到 forwarding table， 它是一张维护对象搬移前和搬移后地址的映射表，key 是对象的旧地址，value 是对象的新地址

在 Relocate 阶段，应用线程新创建的对象地址视图标记为 Remapped

如果应用线程访问到一个地址视图是 Marked0 的对象，说明这个对象还没有被转移，那么就需要将这个对象进行转移，转移之后再加入到 forwarding table，然后再对这个对象的引用直接指向新地址，完成自愈。这些动作都是发生在 read barrier 中的，是由应用线程完成的。

当GC线程遍历到一个对象，如果对象地址视图是Marked0，就将其转移，同时将地址视图置为Remapped，并加入到forwarding table

如果访问到一个对象地址视图已经是Remapped，就说明已经被转移了，也就不做处理了

### Remap

此阶段主要是对地址视图和对象之间的引用关系做修正，因为在 Relocate 阶段，GC 线程会将活跃对象快速搬移到新的区域，但是却不会同时修复对象之间的引用





什么情况下会发生栈溢出

栈的大小可以通过 -Xss 参数进行设置，当递归层次太深的时候，则会发生栈溢出



类加载有几个过程？
加载、验证、准备、解析、初始化



MinorGC、MajorGC、FullGC 都什么时候发生？
MinorGC 在年轻代空间不足的时候发生，

MajorGC 指的是老年代的 GC，出现 MajorGC 一般经常伴有 MinorGC。

FullGC 有三种情况：第一，当老年代无法再分配内存的时候；第二，元空间不足的时候；第三，显示调用 System.gc 的时候。另外，像 CMS 一类的垃圾回收器，在 MinorGC 出现 promotion failure 的时候也会发生 FullGC



safepoint 是什么？
当发生 GC 时，用户线程必须全部停下来，才可以进行垃圾回收，这个状态我们可以认为 JVM 是安全的（safe），整个堆的状态是稳定的。

如果在 GC 前，有线程迟迟进入不了 safepoint，那么整个 JVM 都在等待这个阻塞的线程，造成了整体 GC 的时间变长



JVM 有哪些内存区域？（JVM 的内存布局是什么？）
JVM 包含堆、元空间、Java 虚拟机栈、本地方法栈、程序计数器等内存区域，其中，堆是占用内存最大的一块，

Java 的内存模型是什么？（JMM 是什么？）
JVM 试图定义一种统一的内存模型，能将各种底层硬件以及操作系统的内存访问差异进行封装，使 Java 程序在不同硬件以及操作系统上都能达到相同的并发效果。它分为工作内存和主内存，线程无法对主存储器直接进行操作，如果一个线程要和另外一个线程通信，那么只能通过主存进行交换



JVM 垃圾回收时如何确定垃圾？什么是 GC Roots？
JVM 采用的是可达性分析算法。JVM 是通过 GC Roots 来判定对象存活的，从 GC Roots 向下追溯、搜索，会产生一个叫做 Reference Chain 的链条。当一个对象不能和任何一个 GC Root 产生关系时，就判定为垃圾

GC Roots 大体包括：

 活动线程相关的各种引用，比如虚拟机栈中 栈帧里的引用；
 类的静态变量引用；
 JNI 引用等。
能够找到 Reference Chain 的对象，就一定会存活么？
不一定，还要看 Reference 类型，弱引用在 GC 时会被回收，软引用在内存不足的时候会被回收，但如果没有 Reference Chain 对象时，就一定会被回收。

强引用、软引用、弱引用、虚引用是什么？
普通的对象引用关系就是强引用。

软引用用于维护一些可有可无的对象。只有在内存不足时，系统则会回收软引用对象，如果回收了软引用对象之后仍然没有足够的内存，才会抛出内存溢出异常。

弱引用对象相比软引用来说，要更加无用一些，它拥有更短的生命周期，当 JVM 进行垃圾回收时，无论内存是否充足，都会回收被弱引用关联的对象。

虚引用是一种形同虚设的引用，在现实场景中用的不是很多，它主要用来跟踪对象被垃圾回收的活动。

你都有哪些手段用来排查内存溢出？

内存溢出包含很多种情况，我在平常工作中遇到最多的就是堆溢出。有一次线上遇到故障，重新启动后，使用 jstat 命令，发现 Old 区一直在增长。我使用 jmap 命令，导出了一份线上堆栈，然后使用 MAT 进行分析，通过对 GC Roots 的分析，发现了一个非常大的 HashMap 对象，这个原本是其他同事做缓存用的，但是一个无界缓存，造成了堆内存占用一直上升，后来，将这个缓存改成 guava 的 Cache，并设置了弱引用，故障就消失了。



我们线上使用较多的是 G1，也有年轻代和老年代的概念，不过它是一个整堆回收器，它的回收对象是小堆区 。
生产上如何配置垃圾收集器？


首先是内存大小问题，基本上每一个内存区域我都会设置一个上限，来避免溢出问题，比如元空间。通常，堆空间我会设置成操作系统的 2/3，超过 8GB 的堆，优先选用 G1。

然后我会对 JVM 进行初步优化，比如根据老年代的对象提升速度，来调整年轻代和老年代之间的比例。

接下来是专项优化，判断的主要依据是系统容量、访问延迟、吞吐量等，我们的服务是高并发的，所以对 STW 的时间非常敏感。

我会通过记录详细的 GC 日志，来找到这个瓶颈点，借用 GCeasy 这样的日志分析工具，很容易定位到问题。

假如生产环境 CPU 占用过高，请谈谈你的分析思路和定位。


首先，使用 top -H 命令获取占用 CPU 最高的线程，并将它转化为十六进制。

然后，使用 jstack 命令获取应用的栈信息，搜索这个十六进制，这样就能够方便地找到引起 CPU 占用过高的具体原因。






GC 日志的 real、user、sys 是什么意思？

 real 指的是从开始到结束所花费的时间，比如进程在等待 I/O 完成，这个阻塞时间也会被计算在内。
 user 指的是进程在用户态（User Mode）所花费的时间，只统计本进程所使用的时间，是指多核。
 sys  指的是进程在核心态（Kernel Mode）所花费的 CPU 时间量，即内核中的系统调用所花费的时间，只统计本进程所使用的时间。

什么情况会造成元空间溢出？
元空间默认是没有上限的，不加限制比较危险。当应用中的 Java 类过多时，比如 Spring 等一些使用动态代理的框架生成了很多类，如果占用空间超出了我们的设定值，就会发生元空间溢出。

什么时候会造成堆外内存溢出？
使用了 Unsafe 类申请内存，或者使用了 JNI 对内存进行操作，这部分内存是不受 JVM 控制的，不加限制使用的话，会很容易发生内存溢出。

SWAP 会影响性能么？
当操作系统内存不足时，会将部分数据写入到 SWAP ，但是 SWAP 的性能是比较低的。如果应用的访问量较大，需要频繁申请和销毁内存，那么很容易发生卡顿。一般在高并发场景下，会禁用 SWAP。

有什么堆外内存的排查思路？
进程占用的内存，可以使用 top 命令，看 RES 段占用的值，如果这个值大大超出我们设定的最大堆内存，则证明堆外内存占用了很大的区域。

使用 gdb 命令可以将物理内存 dump 下来，通常能看到里面的内容。更加复杂的分析可以使用 Perf 工具，或者谷歌开源的 GPerftools。那些申请内存最多的 native 函数，就很容易找到。
HashMap 中的 key，可以是普通对象么？有什么需要注意的地方？
Map 的 key 和 value 可以是任何类型，但要注意的是，一定要重写它的 equals 和 hashCode 方法，否则容易发生内存泄漏。



大型项目如何进行性能瓶颈调优

当一个系统出现问题的时候，研发一般不会想要立刻优化 JVM，或者优化操作系统，会尝试从最高层次上进行问题的解决：解决最主要的瓶颈点

数据库优化： 数据库是最容易成为瓶颈的组件，从 SQL 优化或者数据库本身去提高它的性能。如果瓶颈依然存在，则会考虑分库分表将数据打散，如果这样也没能解决问题，则可能会选择缓存组件进行优化。。

集群最优：存储节点的问题解决后，计算节点也有可能发生问题。一个集群系统如果获得了水平扩容的能力，就会给下层的优化提供非常大的时间空间。

硬件升级：水平扩容不总是有效的，原因在于单节点的计算量比较集中，或者 JVM 对内存的使用超出了宿主机的承载范围。在动手进行代码优化之前，我们会对节点的硬件配置进行升级。升级容易，降级难，降级需要依赖代码和调优层面的优化。

代码优化：代码优化是提高性能最有效的方式，但需要收集一些数据，这个过程可能是服务治理，也有可能是代码流程优化。使用 JavaAgent 技术，会无侵入的收集一些 profile 信息，供我们进行决策。

并行优化：并行优化的对象是这样一种接口，它占用的资源不多，计算量也不大，就是速度太慢。所以我们通常使用 ContDownLatch 对需要获取的数据进行并行处理，

JVM 优化：虽然对 JVM 进行优化，有时候会获得巨大的性能提升，但在 JVM 不发生问题时，我们一般不会想到它。原因就在于，相较于上面 5 层所达到的效果来说，它的优化效果有限。

操作系统优化：操作系统优化是解决问题的杀手锏，比如像 HugePage、Luma、“CPU 亲和性”这种比较底层的优化。但就计算节点来说，对操作系统进行优化并不是很常见。运维在背后会做一些诸如文件句柄的调整、网络参数的修改，这对于我们来说就已经够用了。

优化层次
当一个系统出现问题的时候，研发一般不会想要立刻优化 JVM，或者优化操作系统，会尝试从最高层次上进行问题的解决：解决最主要的瓶颈点。

数据库优化： 数据库是最容易成为瓶颈的组件，研发会从 SQL 优化或者数据库本身去提高它的性能。如果瓶颈依然存在，则会考虑分库分表将数据打散，如果这样也没能解决问题，则可能会选择缓存组件进行优化。这个过程与本课时相关的知识点，可以使用 jstack 获取阻塞的执行栈，进行辅助分析。

集群最优：存储节点的问题解决后，计算节点也有可能发生问题。一个集群系统如果获得了水平扩容的能力，就会给下层的优化提供非常大的时间空间，这也是弹性扩容的魅力所在。我接触过一个服务，由最初的 3 个节点，扩容到最后的 200 多个节点，但由于人力问题，服务又没有什么新的需求，下层的优化就一直被搁置着。

硬件升级：水平扩容不总是有效的，原因在于单节点的计算量比较集中，或者 JVM 对内存的使用超出了宿主机的承载范围。在动手进行代码优化之前，我们会对节点的硬件配置进行升级。升级容易，降级难，降级需要依赖代码和调优层面的优化。

代码优化：出于成本的考虑，上面的这些问题，研发团队并不总是坐视不管。代码优化是提高性能最有效的方式，但需要收集一些数据，这个过程可能是服务治理，也有可能是代码流程优化。我在第 21 课时介绍的 JavaAgent 技术，会无侵入的收集一些 profile 信息，供我们进行决策。像 Sonar 这种质量监控工具，也可以在此过程中帮助到我们。

并行优化：并行优化的对象是这样一种接口，它占用的资源不多，计算量也不大，就是速度太慢。所以我们通常使用 ContDownLatch 对需要获取的数据进行并行处理，效果非常不错，比如在 200ms 内返回对 50 个耗时 100ms 的下层接口的调用。

JVM 优化：虽然对 JVM 进行优化，有时候会获得巨大的性能提升，但在 JVM 不发生问题时，我们一般不会想到它。原因就在于，相较于上面 5 层所达到的效果来说，它的优化效果有限。但在代码优化、并行优化、JVM 优化的过程中，JVM 的知识却起到了关键性的作用，是一些根本性的影响因素。

操作系统优化：操作系统优化是解决问题的杀手锏，比如像 HugePage、Luma、“CPU 亲和性”这种比较底层的优化。但就计算节点来说，对操作系统进行优化并不是很常见。运维在背后会做一些诸如文件句柄的调整、网络参数的修改，这对于我们来说就已经够用了。

内存区域大小

 -XX:+UseG1GC：用于指定 JVM 使用的垃圾回收器为 G1，尽量不要靠默认值去保证，要显式的指定一个。
 -Xmx：设置堆的最大值，一般为操作系统的 2/3 大小。
 -Xms：设置堆的初始值，一般设置成和 Xmx 一样的大小来避免动态扩容。
 -Xmn：表示年轻代的大小，默认新生代占堆大小的 1/3。高并发、对象快消亡场景可适当加大这个区域，对半，或者更多，都是可以的。但是在 G1 下，就不用再设置这个值了，它会自动调整。
 -XX:MaxMetaspaceSize：用于限制元空间的大小，一般 256M 足够了，这一般和初始大小 -XX:MetaspaceSize 设置成一样的。
 -XX:MaxDirectMemorySize：用于设置直接内存的最大值，限制通过 DirectByteBuffer 申请的内存。
 -XX:ReservedCodeCacheSize：用于设置 JIT 编译后的代码存放区大小，如果观察到这个值有限制，可以适当调大，一般够用即可。
 -Xss：用于设置栈的大小，默认为 1M，已经足够用了。

内存调优

 -XX:+AlwaysPreTouch：表示在启动时就把参数里指定的内存全部初始化，启动时间会慢一些，但运行速度会增加。
 -XX:SurvivorRatio：默认值为 8，表示伊甸区和幸存区的比例。
 -XX:MaxTenuringThreshold：在 CMS 下默认为 6，G1 下默认为 15，这个值和对象提升有关，改动效果会比较明显。对象的年龄分布可以使用 -XX:+PrintTenuringDistribution 打印，如果后面几代的大小总是差不多，证明过了某个年龄后的对象总能晋升到老生代，就可以把晋升阈值设小。
 PretenureSizeThreshold：表示超过一定大小的对象，将直接在老年代分配，不过这个参数用的不是很多。



常用虚拟机参数

-verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps 
-XX:+PrintGCApplicationStoppedTime -XX:+PrintTenuringDistribution 
-Xloggc:/tmp/logs/gc_%p.log -XX:+HeapDumpOnOutOfMemoryError 
-XX:HeapDumpPath=/tmp/logs -XX:ErrorFile=/tmp/logs/hs_error_pid%p.log 
-XX:-OmitStackTraceInFastThrow

一般在项目中输出详细的 GC 日志，并加上可读性强的 GC 日志的时间戳。特别情况下我还会追加一些反映对象晋升情况和堆详细信息的日志，用来排查问题。另外，OOM 时自动 Dump 堆栈，我一般也会进行配置

jstat -gcutil $pid 1000

只需要提供一个 Java 进程的 ID，然后指定间隔时间（毫秒）就 OK 了。
S0 S1 E O M CCS YGC YGCT FGC FGCT GCT
0.00 0.00 72.03 0.35 54.12 55.72 11122 16.019 0 0.000 16.019
E 其实是 Eden 的缩写，S0 对应的是 Surivor0，S1 对应的是 Surivor1，O 代表的是 Old，而 M 代表的是 Metaspace

YGC 代表的是年轻代的回收次数，YGC T对应的是年轻代的回收耗时。那么 FGC 肯定代表的是 Full GC 的次数

在启动命令行中加上参数 -t，可以输出从程序启动到现在的时间。如果 FGC 和启动时间的比值太大，就证明系统的吞吐量比较小，GC 花费的时间太多了。另外，如果老年代在 Full GC 之后，没有明显的下降，那可能内存已经达到了瓶颈，或者有内存泄漏问题



加了 GC 时间的增量和 GC 时间比率两列。
jstat -gcutil -t 90542 1000 | awk 'BEGIN{pre=0}{if(NR>1) {print $0 "\t" ($12-pre) "\t" $12*100/$1 ; pre=$12 } else { print $0 "\tGCT_INC\tRate"} }'



将GC写入的日志文件，单独放在一块HDD硬盘上，防止多个应用对磁盘资源的争抢

OOM 时自动 Dump 堆栈，写入的文件比较大，短时间内会与其他的磁盘IO争抢资源



FullGC有三种情况。、

第一，当老年代无法再分配内存的时候；第二，元空间不足的时候；第三，显示调用System.gc的时候。另外，像CMS一类的垃圾回收器，在MinorGC出现promotion failure的时候也会发生FullGC

Full GC 都是在老年代空间不足的时候执行。但不要忘了，我们还有一个区域叫作 Metaspace，它的容量是没有上限的，但是每当它扩容时，就会发生 Full GC

可以看到它的默认值：
java -XX:+PrintFlagsFinal 2>&1 | grep Meta

按照经验，一般调整成 256MB 就足够了。同时，为了避免无限制使用造成操作系统内存溢出，我们同时设置它的上限。配置参数如下：
-XX:+UseConcMarkSweepGC -Xmx5460M -Xms5460M -Xmn3460M -XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M

吞吐量大不代表响应能力高，吞吐量一般这么描述：在一个时间段内完成了多少个事务操作；在一个小时之内完成了多少批量操作。

响应能力是以最大的延迟时间来判断的，比如：一个桌面按钮对一个触发事件响应有多快；需要多长时间返回一个网页；查询一行 SQL 需要多长时间，等等。

这两个目标，在有限的资源下，通常不能够同时达到，我们需要做一些权衡

平常的 Web 服务器，都是对响应性要求非常高的。选择 CMS、G1、ZGC 上

一般优化的思路有一个重要的顺序：

 程序优化，效果通常非常大；
 扩容，如果金钱的成本比较小，不要和自己过不去；
 参数调优，在成本、吞吐量、延迟之间找一个平衡点。



大部分的业务场景是高并发的。对象诞生的快，死亡的也快，对年轻代的利用直接影响了整个堆的垃圾收集。

 分配足够大的年轻代，会增加系统的吞吐，但不会增加 GC 的负担。
 容量足够的 Survivor 区，能够让对象尽可能的留在年轻代，减少对象的晋升，进而减少 Major GC



java进程崩溃消失了，什么原因？

dmesg  -T 查看奔溃的信息

大多数崩溃于 Linux 的内存管理有关。由于 Linux 系统采用的是虚拟内存分配方式，JVM 的代码、库、堆和栈的使用都会消耗内存，但是申请出来的内存，只要没真正 access过，是不算的，因为没有真正为之分配物理页面。

随着使用内存越用越多。会启用 SWAP；当 SWAP 也用的差不多了，会尝试释放 cache；当这两者资源都耗尽，杀手就出现了。oom-killer 会在系统内存耗尽的情况下跳出来，选择性的干掉一些进程以求释放一点内存。

所以这时候我们的 Java 进程，是操作系统“主动”终结的



kill -9 PID 强制停掉进程，不给进程使用回调函数的机会，也不会等进程处理完手上的工作，对于已经进入生产环境的系统来说，这是非常危险的。
kill -15 PID 在停掉进程之前调用提前写好的回调函数，或者等待进程处理完正在处理的任务之后，再停掉进程。





MM 是一个抽象的概念，它描述了一系列的规则或者规范，用来解决多线程的共享变量问题，比如 volatile、synchronized 等关键字就是围绕 JMM 的语法

JVM 试图定义一种统一的内存模型，能将各种底层硬件，以及操作系统的内存访问差异进行封装，使 Java 程序在不同硬件及操作系统上都能达到相同的并发效果



如何缩减对内存的占用？

缩减查询的字段，减少常驻内存的数据；
去掉不必要的、命中率低的堆内缓存，改为分布式缓存

 限制单次请求对内存的无限制使用



# 如何搭建虚拟机监控

Jolokia 就是一个将 JMX 转换成 HTTP 的适配器，方便了 JMX 的使用

Jokokia 可以通过 jar 包的形式，集成到 Spring Boot 中

<dependency>
        <groupId>org.jolokia</groupId>
        <artifactId>jolokia-core</artifactId>
</dependency>

配置 application.yml 

management:
  endpoints:
    web:
      exposure:
        include: jolokia

telegraf 组件作为一个通用的监控 agent，和 JVM 进程部署在同一台机器上，通过访问转化后的 HTTP 接口，以固定的频率拉取监控信息；

然后把这些信息存放到 influxdb 时序数据库中；

最后，通过Grafana 展示组件，设计 JVM 监控图表

# 如何排查堆外内存

## 1、堆外内存有哪些

元空间MetaSpace：主要是方法区和常量池的存储之地，使用数“MaxMetaspaceSize”可以限制它的大小

直接内存：通过 DirectByteBuffer 申请的内存,可以使用参数“MaxDirectMemorySize”来限制它的大小

java.nio.Bits

```java
private static volatile long maxMemory = VM.maxDirectMemory();
```

java.nio.DirectByteBuffer#DirectByteBuffer(int)

```java
DirectByteBuffer(int cap) {                   // package-private

    super(-1, 0, cap, cap);
    boolean pa = VM.isDirectMemoryPageAligned();
    int ps = Bits.pageSize();
    long size = Math.max(1L, (long)cap + (pa ? ps : 0)); 
    Bits.reserveMemory(size, cap); //判断可用内存是否小于MaxDirectMemorySize，否则进行gc，gc之后内存还不够用，抛出OOM

    long base = 0;
    try {
        base = unsafe.allocateMemory(size); //用unsafe来分配内存
    } catch (OutOfMemoryError x) {
        Bits.unreserveMemory(size, cap);
        throw x;
    }
    unsafe.setMemory(base, size, (byte) 0);
    if (pa && (base % ps != 0)) {
        // Round up to page boundary
        address = base + ps - (base & (ps - 1));
    } else {
        address = base;
    }
 		 //用来回收堆外内存，继承PhantomReference，虚引用
    cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
    att = null;
}
```

其他堆外内存：主要是指使用了 Unsafe 或者其他 JNI 手段直接直接申请的内存,没有参数限制大小

## 2、堆外内存问题排查

通过top命令，查看进程的VIRT（进程虚拟内存空间大小） 、RES（进程虚拟内存空间中已经映射到物理内存空间的那部分的大小），如果VIRT、RES明显高于实际分配的内存大小，说明肯定使用了堆外内存

 AlwaysPreTouch 参数：虽然参数指定了 JVM 大小，但只有在 JVM 真正使用的时候，才会分配给它。这个参数，在 JVM 启动的时候，就把它所有的内存在操作系统分配了。在堆比较大的时候，会加大启动时间，但在这个场景中，我们为了减少内存动态分配的影响，把这个值设置为 True

NativeMemoryTracking参数：用来追踪 Native 内存的使用情况。通过在启动参数上加入 -XX:NativeMemoryTracking=detail 就可以启用

使用jcmd查看内存分配

jcmd $pid  VM.native_memory summary

### Perf

除了能够进行一些性能分析，它还能帮助我们找到相应的 native 调用

perf record -g -p PID监控栈函数调用

perf 运行一段时间后 Ctrl+C 结束，会生成一个文件 perf.data

perf report -i perf.data 查看报告

### gperftools

google 还有一个类似的、非常好用的工具，叫做 gperftools，我们主要用到它的 Heap Profiler，功能更加强大

输出两个环境变量

mkdir -p /opt/test

export LD_PRELOAD=/usr/lib64/libtcmalloc.so 

export HEAPPROFILE=/opt/test/heap #存放内存申请的动作

pprof -text *heap | head -n 200 #分析文件，追踪到申请内存最多的函数



获取内存快照

通过配置一些参数，可以在发生 OOM 的时候，被动 dump 一份堆栈信息，这是一种；

另一种，就是通过 jmap 主动去获取内存的快照

jmap -dump:format=b,file=heap.bin 37340
jhsdb jmap  --binaryheap --pid  37340（java9）

这些命令本身会占用操作系统的资源，在某些情况下会造成服务响应缓慢，所以不要频繁执行



内存溢出

一般内存溢出，表现形式就是 Old 区的占用持续上升，即使经过了多轮 GC 也没有明显改善。内存泄漏的根本就是，有些对象并没有切断和 GC Roots 的关系

在 java9 版本里被干掉了，jmap被 jhsdb取代，可以像下面的命令一样使用：
jhsdb jmap  --heap --pid  37340 看到大体的内存布局，以及每一个年代中的内存使用情况
jhsdb jmap  --pid  37288
jhsdb jmap  --histo --pid  37340 能够大概的看到系统中每一种类型占用的空间大小，用于初步判断问题。比如某个对象 instances 数量很小，但占用的空间很大，这就说明存在大对象
jhsdb jmap  --binaryheap --pid  3734

jhsdb jmap  --binaryheap --pid  37340



问题排查工具MAT

下载地址https://eclipse.dev/mat/downloads.php

浅堆代表了对象本身的内存占用，包括对象自身的内存占用，以及“为了引用”其他对象所占用的内存。

深堆是一个统计结果，会循环计算引用的具体对象所占用的内存。但是深堆和“对象大小”有一点不同，深堆指的是一个对象被垃圾回收后，能够释放的内存大小，这些被释放的对象集合，叫做保留集（Retained Set）





如何观察 GC 的频次？

jstat -gc PID 1000 3

通过垃圾回收频率和消耗时间初步判断 GC 是否存在可疑问题



GC 调优思路

对于 GC 调优来说，首先就需要清楚调优的目标是什么？

从性能的角度看，通常关注三个方面，内存占用（footprint）、延时（latency）和吞吐量（throughput），大多数情况下调优会侧重于其中一个或者两个方面的目标，很少有情况可以兼顾三个不同的角度

一般是降低GC暂停的时间，保证一定标准的吞吐量

定位具体的问题，确定真的有 GC 调优的必要

查看内存占用情况

ps -ef|grep 应用程序名称 查看进程ID

 jcmd PID GC.heap_info 查看new、old区的使用情况

 jcmd  PID  GC.class_histogram |head -n 20 查看该进程占用内存多大的对象

查看GC情况

jstat -gccause PID 显示最近一次gc的原因

过追踪 GC 日志，查找是不是 GC 在特定时间发生了长时间的暂停，进而导致了应用响应不及时

Minor GC 发生的频次、暂停的时间，old gc发生的频次、暂停的时间，full gc发生的频次、暂停的时间

发生gc问题，一般是由程序问题或者参数问题，程序问题比如，从缓存或者数据库加载了大量的数据放在内存中，期间又发生了远程调用，这些数据一致占用着内存，迟迟得不到释放。参数问题，比如年轻代配置的太小，程序中又产生了大量对象，发生的频次会过高，如果存活的对象过多而幸存区空间又不充足，会提前晋升到老年代，又会导致old gc频次过高





可视化的 JVM 监控工具

jvisual 能做的事情很多，监控内存泄漏、跟踪垃圾回收、执行时内存分析、CPU 线程分析等，而且通过图形化的界面指引就可以完成

nohup java -Djava.rmi.server.hostname=实际ip

 -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=1099 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -jar demo-0.0.1-SNAPSHOT.jar &



直接一键查看指定进程里导致 CPU 飙升的 java 方法，非常简单方便
 show-busy-java-threads -p <指定的Java进程Id>

https://github.com/oldratlee/useful-scripts





压缩指针

在默认打印的对象内存布局信息中，Klass Pointer被压缩成4字节，如果我们不希望开启压缩指针功能，则可以增加一个JVM参数-XX:-UseCompressedOops。





1.1 确认堆内存是使用否太大

Java进程pid(使用jps -v列出java进程列表)

获取运行环境的物理内存和剩余内存（free -m）

Java进程的堆内存用量（jcmd -p PID GC.heap_info）

Java进程GC情况（jstat-gcutil）



获取进程分配的内存映射信息 ，

pmap -x PID |sort -gr -k2 |less （-g,按十进制数值进行排序;-r,进行逆序排序;-k2,对第二个字段排序）

export MALLOC_ARENA_MAX=1

开启Native Memory Tracking

-XX:NativeMemoryTracking=detail

jcmd PID  VM.native_memory detail

开启NativeMemoryTracking会造成5%的性能下降，用完记得修改JVM参数并重启永久关闭，或者可以通过以下命令临时关闭：jcmd vm.native_memory stop  PID

堆外内存是否用量太多

堆外内存，尤其是网络通信非常频繁的应用，往往大量使用Java NIO，而NIO为了提高效率，往往会申请很多的堆外内存。确认这个区域用量是否过大，最直接的方法是先查看是否是DirectByteBuffer或者MappedByteBuffer使用了较多的堆外内存

如果你的服务器已经开启了远程JMX，可以通过jdk自带的工具（比如jconsole、jvisualvm）查询



尽可能地调优新生代先

​	尽可能让对象呆在survivor中，使之在新生代中回收，减少晋升到老年代的对象

 	同时避免长时间存活的对象在survivor间不必要的拷贝

​	合理配置eden和survivor之间的内存比例

对CMS在不紧要的时间段手动进行fullgc

​	压缩，清除内存碎片

大小的平衡 单次gc的时间和gc的频率

硬件优化

​	加CPU

程序优化，避免无用对象占用old空间 

​	不掉不必要的缓存



GC 调优策略

 jmap -heap ID 查询出 JVM 的配置信息

GC 性能衡量指标：吞吐量（系统总运行时间 = 应用程序耗时 +GC 耗时，如果系统运行了 100 分钟，GC 耗时 1 分钟，则系统吞吐量为 99%。GC 的吞吐量一般不能低于95%。）、停顿时间、垃圾回收频率

查看 & 分析 GC 日志

	  -XX:+PrintGC 输出 GC 日志
	  -XX:+PrintGCDetails 输出 GC 的详细日志
	  -XX:+PrintGCTimeStamps 输出 GC 的时间戳（以基准时间的形式）
	  -XX:+PrintGCDateStamps 输出 GC 的时间戳（以日期的形式，如 2013-05-04T21:53:59.234+0800）
	  -XX:+PrintHeapAtGC 在进行 GC 的前后打印出堆的信息
	  -Xloggc:../logs/gc.log 日志文件的输出路径
GCView工具、GCeasy，打开日志文件，图形化界面查看整体的 GC 性能

1、降低 Minor GC 频率

​	可以通过增大新生代空间来降低 Minor GC 的频率，但是也会增加单次Minor GC 的时间

​	单次 Minor GC 时间是由两部分组成：T1（扫描新生代）和 T2（复制存活对象）。在虚拟机中，复制对象的成本要远高于扫描成本

​	如果在堆内存中存在较多的长期存活的对象，此时增加年轻代空间，反而会增加 Minor GC的时间

​	如果堆中的短期对象很多，那么扩容新生代，单次 Minor GC 时间不会显著增加

​	因此，单次 Minor GC 时间更多取决于 GC 后存活对象的数量，而非 Eden 区的大小

2、降低 Full GC 的频率

​	减少创建大对象、增大堆内存空间



 G1 垃圾回收器调优的小建议

## **1.** **最大** **GC** **停顿时间**

-XX:MaxGCPauseMillis=200

## **2.** **避免设置** **Young** **代大小**

## **3.** **清除旧的参数**

## **4.** **消除字符串重复项**

-XX:+UseStringDeduplication

## **5.** **了解默认设置**

-XX:MaxGCPauseMillis=200

XX:G1HeapRegionSize=n	设置 G1 区域大小。值必须为 2 的 N 次幂，如：256、512、1024…范围是 1MB 至 32MB。

-XX:G1ReservePercent=10	设置需保留的内存百分比。默认为 10%。G1 垃圾回收器会始终尝试保留 10% 的堆内存空间空闲

-XX:G1OldCSetRegionThresholdPercent=10	设置混合垃圾回收周期中要收集的 Old 区域数量上限。默认为 Java 堆内存的 10%

-XX:G1MaxNewSizePercent=60	设置用作 Young 代空间大小的最高堆内存百分比。默认值为 Java 堆内存的 60%。

-XX:G1NewSizePercent=5	设置用作 Young 代空间大小的最低堆内存百分比。默认值为 Java 堆内存的 5%。

-XX:InitiatingHeapOccupancyPercent=45	当堆内存使用率超过此百分比时会触发 GC 标记周期。默认值为 45%

-XX:ConcGCThreads=n	设置并行标记线程的数量。将 n 值设为并行垃圾回收线程（ParallelGCThreads）数的大约 1/4

-XX:ParallelGCThreads=n	设置 Stop-the-world 工作线程的数量。如果逻辑处理器的数量小于或等于 8 个，则将 n 值设置为逻辑处理器的数量。如果您的服务器有 5 个逻辑处理器，则将 n 设置为 5。如果有 8 个以上的逻辑处理器，请将该值设置为逻辑处理器数量的大约 5/8。这种设置在大多数情况下都有效，除了较大规模的 SPARC 系统——其中 n 值可以大约是逻辑处理器数的 5/16

-XX:GCTimeRatio=12	设置应用于 GC 的总目标时间与处理客户事务的总时间。确定目标 GC 时间的实际公式为 [1 / (1 + GCTimeRatio)]。默认值 12 表示目标 GC 时间为 [1 / (1 + 12)]，即 7.69%。这意味着 JVM 可将 7.69％ 的时间用于 GC 活动，其余 92.3％ 用于处理客户活动 

## **6.** **研究** **GC** **原因**

1. 在应用程序中启用 GC 日志

Java 8 及之前版本:

​	-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:{file-path}

Java 9 及之后版本：

​	-Xlog:gc*:file={file-path}

### **6.1.Full GC – Allocation Failure**

Full GC – Allocation Failure 的起因有两个：

- 应用程序创建了过多对象且回收对象的速度跟不上。
- 堆内存碎片化时，即使有大量可用空间，Old 代中的直接分配也可能失败

可能解决方案:

1、通过设置‘-XX:ConcGCThreads’值提高并发标记线程数量。增加并发标记线程数量可让垃圾回收运行速度更快

2、强制 G1 提前开始标记阶段。可通过降低‘-XX:InitiatingHeapOccupancyPercen’值实现。默认值为 45。其意义是 **G1 GC** 标记阶段只会在堆内存使用率达到 45% 时开始。降低该值会让 G1 GC 标记阶段提前触发，这样可避免 **Full GC**

3、即使堆内存中有足够的空间，但由于缺少连续空间，Full GC 也可能会发生。导致该问题的原因可能是内存中存在很多占用空间巨大的对象。解决此问题的一个潜在方案是通过选项“-XX:G1HeapRegionSize”来增加堆内存大小，从而减少大型对象所浪费的内存

### **6.2.G1** **疏散停顿或疏散失败**

1、增大‘-XX:G1ReservePercent’参数的值。默认值为 10%。这意味着 **G1** **垃圾收集器**将尝试始终保持 10％ 的可用内存。增大该值后，GC 将提前触发，防止疏散停顿的出现

2、通过降低‘-XX:InitiatingHeapOccupancyPercent’来提前开始标记周期。默认值为 45。降低该值会让标记周期更早开始。当堆内存使用率超过 45% 时会触发 GC 标记周期。另一方面，如果标记周期提前开始但没有进行回收，请将‘-XX:InitiatingHeapOccupancyPercent’阈值增加到默认值以上

3、还可增大‘-XX:ConcGCThreads’参数的值，从而提高并行标记线程的数量。增加并发标记线程数量可让垃圾回收运行速度更快

4、如果问题仍然存在，您可以考虑增加 JVM 堆内存大小（即：-Xmx）

### **6.3.G1** **大型对象分配**

任何超过一半区域大小的对象都可被认为是“大型对象”。如果区域中包含大型对象，则区域中最后一个大型对象与区域结尾之间的空间将不会被使用。如果多个这样的大型对象，那么未使用的空间就会导致堆内存碎片化。[堆碎片](https://blog.gceasy.io/2020/05/31/what-is-java-heap-fragmentation/)的出现不利于应用程序性能。如果您看到多次大型对象分配，请提高‘-XX:G1HeapRegionSize’。其值应为 2 的幂次，范围是 1MB 至 32 MB。

### **6.4.System.gc()**

a.搜索并替换

​	在程序代码库中搜索“System.gc()”与“Runtime.getRuntime().gc()”。如果出现匹配，就将相应的内容移除

b. -XX:+DisableExplicitGC

​	强制禁用 System.gc() 调用

c. -XX:+ExplicitGCInvokesConcurrentAndUnloadsClasses

​	传递此参数时，GC 回收会与程序线程并发运行，进而缩短冗长的停顿时间

**d.RMI**

如果您的程序使用了 RMI，则可控制其中进行“System.gc()”调用的频率。您可在程序启动时启用以下 JVM 参数来对频率进行配置：

-Dsun.rmi.dgc.server.gcInterval=n

-Dsun.rmi.dgc.client.gcInterval=n

这些属性的默认值为：

JDK 1.4.2 和 5.0 为 60000 毫秒（即 60 秒）

JDK 6 及更高版本为 3600000 毫秒（即 60 分钟）

您可将这些属性设置尽可能大的值以最大限度地减少影响。



**-XX:OnOutOfMemoryError**

 JVM 进行配置以使其在抛出 OutOfMemoryError 时调用任意脚本

-XX:OnOutOfMemoryError=/scripts/restart-myapp.sh

# 通用GC 日志分析器

https://gceasy.ycrash.cn/gc-index.jsp

#### 启用 GC 日志

Until Java 8: **-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:<file-path>**
Java 9 & above: -Xlog:gc*:file=<gc-log-file-path>
