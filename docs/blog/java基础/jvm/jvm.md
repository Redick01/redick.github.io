# JAVA基础

## java字节码

    字节码的本质是java代码编译后的，给jvm虚拟机运行的操作指令。

    Java bytecode 由单字节(byte)的指令组成，理论上最多支持 256 个操作码(opcode)。实
    际上Java只使用了200左右的操作码， 还有一些操作码则保留给调试操作。
    根据指令的性质，主要分为四个大类：
    1. 栈操作指令，包括与局部变量交互的指令
    2. 程序流程控制指令
    3. 对象操作指令，包括方法调用指令
    4. 算术运算以及类型转换指令

### java字节码指令
    invokestatic，顾名思义，这个指令用于调用某个类的静态方法，这也是方法调用指令中
    最快的一个。
    invokespecial, 我们已经学过了, invokespecial 指令用来调用构造函数，但也可以用于调
    用同一个类中的 private 方法, 以及可见的超类方法。
    invokevirtual，如果是具体类型的目标对象，invokevirtual用于调用公共，受保护和打包
    私有方法。
    invokeinterface，当要调用的方法属于某个接口时，将使用 invokeinterface 指令。
    invokedynamic,，JDK7新增加的指令是实现“动态类型语言”（Dynamically Typed 
    Language）支持而进行的改进之一，同时也是JDK 8以后支持的lambda表达式的实现基
    础。

## java字节码运行命令
    java -verbose

## Java类加载

### Java类加载过程
    1. 加载（Loading）：找class文件在哪
    2. 验证（Verification）：验证格式、依赖
    3. 准备（Preparation）：静态字段、方法表、类结构
    4. 解析（Resolution）：符号解析为引用
    5. 初始化（Initialization）：构造器、静态变量赋值、静态代码块
    6. 使用（Using）
    7. 卸载（Unloading）

### Java类加载时机
    1. 当虚拟机启动时，初始化用户指定的主类，就是启动执行的 main方法所在的类；
    2. 当遇到用以新建目标类实例的 new 指令时，初始化 new 指令的目标类，就是new一
    个类的时候要初始化；
    1. 当遇到调用静态方法的指令时，初始化该静态方法所在的类；
    2. 当遇到访问静态字段的指令时，初始化该静态字段所在的类；
    3. 子类的初始化会触发父类的初始化；
    4. 如果一个接口定义了 default 方法，那么直接实现或者间接实现该接口的类的初始化，
    会触发该接口的初始化；
    1. 使用反射 API 对某个类进行反射调用时，初始化这个类，其实跟前面一样，反射调用
    要么是已经有实例了，要么是静态方法，都需要初始化；
    1. 当初次调用 MethodHandle 实例时，初始化该 MethodHandle 指向的方法所在的
    类。

### 类不会初始化（可能会加载）

    1. 通过子类引用父类的静态字段，只会触发父类的初始化，而不会触发子类的初始化。
    2. 定义对象数组，不会触发该类的初始化。
    3. 常量在编译期间会存入调用类的常量池中，本质上并没有直接引用定义常量的类，不
    会触发定义常量所在的类。
    1. 通过类名获取Class对象，不会触发类的初始化，Hello.class不会让Hello类初始化。
    2. 通过Class.forName加载指定类时，如果指定参数initialize为false时，也不会触发
    类初始化，其实这个参数是告诉虚拟机，是否要对类进行初始化。
    Class.forName(“jvm.Hello”)默认会加载Hello类。
    1. 通过ClassLoader默认的loadClass方法，也不会触发初始化动作（加载了，但是不
    初始化）。

### Java类加载器
    1、启动类加载器（BootStrapClassLoader）：负责加载核心java包，如：jdk的lib包下rt.jar

    2、扩展类加载器（ExtClassLoader）：负责加载扩展的java包，如：jdk的lib/ext下的jar

    3、应用类加载器（AppClassLoader）：我们写的类由它加载

### Java类加载器特点
    1、双亲委派：简单说就是加载类的时候如果当前加载器没有加载会问父加载器加载没有，加载了就直接用，没加载继续向上问都没加载就加载

    2、负责依赖

    3、缓存加载

    URLClassLoader加载我们自己打的jar或依赖的jar，可以通过该加载器查看我们的程序加载了哪些jar
    

## JVM

### JVM内存模型
    JVM内存主要包括堆和非堆
- 堆（Heap）：线程共享的区域，所有实例对象都在堆上分配内存，该区域也是GC主要管理的内存区，堆内存又将内存空间分成新生代（Eden Space）、老年代（Old Space）、幸存者区（Surviror），这么做的目的是GC算法的分代收集，后面会介绍到GC算法。
- 虚拟机栈：JVM虚拟机栈属于线程私有的，每个线程只能访问自己的线程栈，主要存储，类信息、存储局部变量表、操作栈、动态链接、方法出口，对象指针。
- 本地方法栈：线程私有。为虚拟机使用到的Native 方法服务。如Java使用c或者c++编写的接口服务时，代码在此区运行。
- 程序计数器：线程私有。有些文章也翻译成PC寄存器（PC Register），同一个东西。它可以看作是当前线程所执行的字节码的行号指示器。指向下一条要执行的指令。
- 元数据空间（MateSpace）：java8以后用于替代方法区的内存区域，线程共享的，主要用于存储类信息，常量，方法数据等，元数据空间的大小取决于本地内存，也就是说只要本地内存够大就不会出现内存不够用的情况，替代方法区也避免了“java.lang.OutOfMemoryError: PermGen space”

除此之外在在Java程序中还有一部分内存不在JVM内存模型中，这就是堆外内存，堆外内存主要是通过本地方法申请的直接内存，例如Netty中的DirectByteBuffer。

### JVM命令行工具

    1、jps/jinfo：查看JVM进程信息命令

    2、jstat：查看JVM内部GC相关信息命令

    -class 类加载(Class loader)信息统计. -compiler JIT 即时编译器相关的统计信息。
    -gc GC 相关的堆内存信息. 用法: jstat -gc -h 10 -t 864 1s 20
    -gccapacity 各个内存池分代空间的容量。
    -gccause 看上次 GC, 本次 GC（如果正在GC中）的原因, 其他 输出和 -gcutil 选项一致。
    -gcnew 年轻代的统计信息. （New = Young = Eden + S0 + S1） -gcnewcapacity 年轻代空间大小统计. -gcold 老年代和元数据区的行为统计。
    -gcoldcapacity old 空间大小统计. -gcmetacapacity meta 区大小统计. -gcutil GC 相关区域的使用率(utilization)统计。
    -printcompilation 打印 JVM 编译统计信息。

    3、jmap：查看堆或类占用空间统计

    -heap 打印堆内存（/内存池）的配置和
    使用信息。
    -histo 看哪些类占用的空间最多, 直方图
    -dump:format=b,file=xxxx.hprof 
    Dump 堆内存。

    4、jstack：查看java线程情况

    -F 强制执行 thread dump. 可在 Java 进程卡死
    （hung 住）时使用, 此选项可能需要系统权限。
    -m 混合模式(mixed mode),将 Java 帧和 native
    帧一起输出, 此选项可能需要系统权限。
    -l 长列表模式. 将线程相关的 locks 信息一起输出，
    比如持有的锁，等待的锁。

    5、jcmd：可以综合以上命令，是一个整合命令，可以通过 jcm pid help查看可以使用的命令

    6、jrunscript/jjs：jrunscript可以执行js文件、脚本段并且可以充当curl使用，jjs很有意思可以执行脚本

### JVM图形化工具

    1、JConsole

    2、Jvisualvm

    3、VisualGC：有ieda插件

    4、jmc

## GC

### GC算法演进
    1、引用计数法简餐粗暴，但是有可能产生循环引用使得永远不会GC就会导致内存泄漏，最终可能导致内存溢出；
    2、标记清除法：遍历所有可达对象并进行标记，清除不可达对象，但是会产生内存随便，造成内存不连续；
    3、标记清楚整理法：在标记清楚基础上进行内存压缩，防止产生内存碎片。
### 分代收集
    将堆分为新生代和老年代，使用不同的垃圾回收算法，在新生代中分为Eden区和两个Survivor区，两个存活区 from 和 to，互换角色。对象存活到一定周期会提升到老年代；
    晋升到老年代的策略有-XX:+MaxTenuringThreshold参数控制
    老年代默认都是存活的对象象，采用移动方式：
        1. 标记所有通过 GC roots 可达的对象；
        2. 删除所有不可达对象；
        3. 整理老年代空间中的内容，方法是将所有的存活对象复制，从老年代空间开始的地方依次存放。
    注意：为什么是复制，不是移动？因为新生代要频繁的进行FC复制效率更高，而且新生代的分区结构也非常适合复制，类似于以空间换时间。
    
    对象存活到一定周期会晋升到老年代，晋升到老年代由-XX:+MaxTenuringThreshold参数控制
    老年代默认都是存活对象，采用移动方式：
    1. 标记所有通过 GC roots 可达的对象；
    2. 删除所有不可达对象；
    3. 整理老年代空间中的内容，方法是将所有的存活对象复制，从老年代空间开始的地方依次存放。

### 可以作为GC Roots的对象
    1. 当前正在执行的方法里的局部变量和输入参数
    2. 活动线程（Active threads）
    3. 所有类的静态字段（static field）
    4. JNI引用此阶段暂停的时间，与堆内存大小,对象的总数没有直接关系，而是由存活对象（alive objects）的数量来决定。所以增加堆内存的大小并不会直接影响标记阶段占用的时间。
   
### 垃圾收集器
#### 串行GC
    串行GC：-XX:+UseSerialGC 配置串行 GC串行 GC 对年轻代使用 mark-copy（标记-复制） 算法，对老年代使用 mark-sweep-compact（标记-清除-整理）算法。两者都是单线程的垃圾收集器，
    不能进行并行处理，所以都会触发全线暂停（STW），停止所有的应用线程。-XX:+UserParNewGC是改进版的SerialGC，配合CMS使用。
  
#### 并行GC
    并行GC：-XX：+UseParallelGC
            -XX：+UseParallelOldGC
            -XX：+UseParallelGC -XX:+UseParallelOldGC
            年轻代和老年代的垃圾回收都会触发 STW 事件。
            在年轻代使用 标记-复制（mark-copy）算法，在老年代使用 标记-清除-整理（mark-sweepcompact）算法。 
            -XX：ParallelGCThreads=N 来指定 GC 线程数， 其默认值为 CPU 核心数。
            并行垃圾收集器适用于多核服务器，主要目标是增加吞吐量。因为对系统资源的有效使用，能达到更高的吞吐量: 
            • 在 GC 期间，所有 CPU 内核都在并行清理垃圾，所以总暂停时间更短；
            • 在两次 GC 周期的间隔期，没有 GC 线程在运行，不会消耗任何系统资源。
#### CMS GC
    CMS GC：-XX:+UseConcMarkSweepGC
              其对年轻代采用并行 STW 方式的 mark-copy (标记-复制)算法，对老年代主要使用并发 marksweep (标记-清除)算法。
              CMS GC 的设计目标是避免在老年代垃圾收集时出现长时间的卡顿，主要通过两种手段来达成此目标：
              1. 不对老年代进行整理，而是使用空闲列表（free-lists）来管理内存空间的回收。
              2. 在 mark-and-sweep （标记-清除） 阶段的大部分工作和应用线程一起并发执行。
       CMS GC的六个阶段：
       阶段 1: Initial Mark（初始标记）：该阶段STW，初始标记的目标是标记所有的根对象，包括根对象直接引用的对象，以及被年轻代中所有存活对象所引用的对象（老年代单独回收）。
       阶段 2: Concurrent Mark（并发标记）：此阶段，CMS遍历老年代，标记所有存活对象。
       阶段 3: Concurrent Preclean（并发预清理）：此阶段同样是与应用线程并发执行的，不需要停止应用线
                                        程。 因为前一阶段【并发标记】与程序并发运行，可能
                                        有一些引用关系已经发生了改变。如果在并发标记过程中
                                        引用关系发生了变化，JVM 会通过“Card（卡片）”的方
                                        式将发生了改变的区域标记为“脏”区，这就是所谓的 卡片
                                        标记（Card Marking）。
       阶段 4: Final Remark（最终标记）：最终标记阶段是此次 GC 事件中的第二次（也是最后一
                                次）STW 停顿。本阶段的目标是完成老年代中所有存活
                                对象的标记. 因为之前的预清理阶段是并发执行的，有可
                                能 GC 线程跟不上应用程序的修改速度。所以需要一次
                                STW 暂停来处理各种复杂的情况。
                                通常 CMS 会尝试在年轻代尽可能空的情况下执行 Final 
                                Remark 阶段，以免连续触发多次 STW 事件。
       阶段 5: Concurrent Sweep（并发清除）：此阶段与应用程序并发执行，不需要 STW 停顿。JVM 在此阶段删除不再使用的对象，并回收他们占用的内存空间。
       阶段 6: Concurrent Reset（并发重置）：此阶段与应用程序并发执行，重置 CMS 算法相关的内部数据，为下一次 GC 循环做准备。

#### G1 GC
    G1 GC：-XX:+UseG1GC G1收集器的设计目标是取代CMS，G1相比CMS存在两个出色表现：G1有一个内存整理过程，不会产生过多的内存碎片；G1的STW时间更加可控，G1的STW时间上添加了预测机制，用户可以指定期望停顿时间。
       G1收集器引入了以下概念：
       （1）Region：传统的收集器将连续的堆内存进行分代（年轻代，老年代。。。。），这种划分下，各代的存储地址是连续的，如下描述。
                  ｜<-新生代->|<-老年代->|<-matespace（或永久代））->|
                  而G1收集器各个代的存储地址是不连续的，每个代都是用了n个不连续的大小相同的Region，每个Region占有一块连续的虚拟地址。大小大于等于Region一般的对象称作巨型对象（humongous object），巨型对象
                  会直接分配在old gen中，巨型对象在并发标记和full GC阶段回收，在分配巨型对象之前先检查是否超过，initiating heap occupancy percent和the marking threshold，如果超过的话，会进行并发标记
                  提早进行垃圾回收，防止出现full GC。为了减少连续H-objs分配对GC的影响，需要把大对象变为普通的对象，建议增大Region size。
       （2）SATB：全称是Snapshot-At-The-Beginning，GC开始时存活对象的快照，作用是维持并发GC的正确性。
       （3）RSet：全称是Remembered Set，是辅助GC过程的一种结构，典型的空间换时间工具，和Card Table有些类似。还有一种数据结构也是辅助GC的：Collection Set（CSet），它记录了GC要收集的Region集合，集合里的Region可以是任意年代的。在GC的时候，对于old->young和old->old的跨代对象引用，只要扫描对应的CSet中的RSet即可。
       （4）Pause Prediction Model：即停顿预测模型 
#### ZGC
    ZGC：-XX:+UnlockExperimentalVMOptions -XX:+UseZGC -Xmx16g 通过着色指针和读屏障，实现几乎全部的并发执行，几毫秒级别的延迟，线性可扩展；
           ZGC最主要的特点包括:
           1. GC 最大停顿时间不超过 10ms
           2. 堆内存支持范围广，小至几百 MB 的堆空间，大至4TB 的超大堆内存（JDK13升至16TB）
           3. 与 G1 相比，应用吞吐量下降不超过15%
           4. 当前只支持 Linux/x64 位平台，JDK15后支持MacOS和Windows系统

