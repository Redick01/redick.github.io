# GC算法演进与垃圾收集器 <!-- {docsify-ignore-all} -->

&nbsp; &nbsp;
&nbsp; &nbsp;
&nbsp; &nbsp;
&nbsp; &nbsp;

## GC算法演进

&nbsp; &nbsp;

###### 引用计数法

&nbsp; &nbsp; 简餐粗暴，但是有可能产生循环引用使得永远不会GC就会导致内存泄漏，最终可能导致内存溢出

###### 标记清除法

&nbsp; &nbsp; 遍历所有可达对象并进行标记，清除不可达对象，但是会产生内存碎片，造成内存不连续

###### 标记清除整理法

&nbsp; &nbsp; 在标记清除基础上进行内存压缩，防止产生内存碎片

###### 分代收集

&nbsp; &nbsp; 将堆分为新生代和老年代，使用不同的垃圾回收算法，在新生代中分为Eden区和两个Survivor区，两个存活区from和to，互换角色。对象存活到一定周期会提升到老年代；晋升到老年代的策略由-XX:+MaxTenuringThreshold参数控制

&nbsp; &nbsp; 老年代默认都是存活的对象，采用移动方式：
1. 标记所有通过 GC roots 可达的对象；
2. 删除所有不可达对象；
3. 整理老年代空间中的内容，方法是将所有的存活对象复制，从老年代空间开始的地方依次存放。

###### 可以作为GC Roots的对象

1. 当前正在执行的方法里的局部变量和输入参数
2. 活动线程（Active threads）
3. 所有类的静态字段（static field）
4. JNI引用此阶段暂停的时间，与堆内存大小,对象的总数没有直接关系，而是由存活对象（alive objects）的数量来决定。所以增加堆内存的大小并不会直接影响标记阶段占用的时间。

&nbsp; &nbsp;
&nbsp; &nbsp;

## 垃圾收集器
    
&nbsp; &nbsp;

###### 串行GC

- 开启串行GC

```shell
-XX:+UseSerialGC
```

串行GC对年轻代使用mark-copy（标记-复制）算法，对老年代使用mark-sweep-compact（标记-清除-整理）算法。两者都是单线程的垃圾收集器，不能进行并行处理，所以都会触发全线暂停（STW），停止所有的应用线程。-XX:+UserParNewGC是改进版的SerialGC，配合CMS使用。

###### 并行GC

- 开启并行GC

```shell
-XX：+UseParallelGC 年轻代指定并行GC
-XX：+UseParallelOldGC 老年代指定并行GC
-XX：+UseParallelGC -XX:+UseParallelOldGC
```

年轻代和老年代的垃圾回收都会触发STW事件。在年轻代使用`标记-复制（mark-copy）`算法，在老年代使用`标记-清除-整理（mark-sweepcompact）`算法。 

-XX：ParallelGCThreads=N来指定`GC`线程数， 其默认值为`CPU`核心数。
            
并行垃圾收集器适用于多核服务器，主要目标是增加吞吐量。因为对系统资源的有效使用，能达到更高的吞吐量: 
- 在`GC`期间，所有`CPU`内核都在并行清理垃圾，所以总暂停时间更短；
- 在两次`GC`周期的间隔期，没有`GC`线程在运行，不会消耗任何系统资源。


###### CMS GC

- 开启CMS GC

```shell
-XX:+UseConcMarkSweepGC
```

- CMS重要参数
```Shell
-XX:+UseConcMarkSweepGC：
启用CMS垃圾收集器。
-XX:ConcGCThreads：
指定并发标记阶段使用的线程数量。可以根据系统配置调整。
-XX:CMSInitiatingOccupancyFraction：
设置触发CMS收集的老年代占用百分比阈值。当老年代占用达到该阈值时，CMS将启动垃圾收集。例如，-XX:CMSInitiatingOccupancyFraction=75 表示当老年代占用达到75%时启动CMS收集。
-XX:+UseCMSInitiatingOccupancyOnly：
使用此参数时，CMS将仅根据 CMSInitiatingOccupancyFraction 触发垃圾收集，而不考虑其他收集策略。这有助于固定CMS收集周期。
-XX:+UseParNewGC：
启用ParNew并行收集器用于年轻代。CMS通常与ParNew一起使用来实现并发标记-清理。
-XX:MaxTenuringThreshold：
设置对象在年轻代中经历多少次垃圾收集后晋升到老年代。可以根据应用程序的特性进行调整。
-XX:CMSFullGCsBeforeCompaction：
设置进行Full GC之前触发多少次CMS垃圾收集。在一些情况下，Full GC 可以帮助合并碎片。例如，-XX:CMSFullGCsBeforeCompaction=0 表示每次Full GC前都会执行CMS收集。
-XX:+UseCMSCompactAtFullCollection：
启用Full GC时进行碎片整理。此选项默认是开启的，可以通过 -XX:-UseCMSCompactAtFullCollection 关闭。 
```

其对年轻代采用并行`STW`方式的`mark-copy (标记-复制)`算法，对老年代主要使用并发`marksweep (标记-清除)`算法。
              
CMS GC 的设计目标是避免在老年代垃圾收集时出现长时间的卡顿，主要通过两种手段来达成此目标：

1. 不对老年代进行整理，而是使用空闲列表（free-lists）来管理内存空间的回收。
2. 在 mark-and-sweep （标记-清除） 阶段的大部分工作和应用线程一起并发执行。

- CMS GC的六个阶段

       阶段 1: Initial Mark（初始标记）：该阶段STW，初始标记的目标是标记所有的根对象，包括根对象直接引用的对象，以及被年轻代中所有存活对象所引用的对象（老年代单独回收）。
       阶段 2: Concurrent Mark（并发标记）：此阶段，CMS遍历老年代，标记所有存活对象。
       阶段 3: Concurrent Preclean（并发预清理）：此阶段同样是与应用线程并发执行的，不需要停止应用线
                                        程。 因为前一阶段【并发标记】与程序并发运行，可能
                                        有一些引用关系已经发生了改变。如果在并发标记过程中
                                        引用关系发生了变化，JVM 会通过“Card（卡片）”的方
                                        式将发生了改变的区域标记为“脏”区，这就是所谓的 卡片
                                        标记（Card Marking）。
       阶段 4: Final Remark（最终标记）：最终标记阶段是此次GC事件中的第二次（也是最后一
                                次）STW停顿。本阶段的目标是完成老年代中所有存活
                                对象的标记. 因为之前的预清理阶段是并发执行的，有可
                                能GC线程跟不上应用程序的修改速度。所以需要一次
                                STW 暂停来处理各种复杂的情况。
                                通常CMS会尝试在年轻代尽可能空的情况下执行Final 
                                Remark 阶段，以免连续触发多次STW事件。
       阶段 5: Concurrent Sweep（并发清除）：此阶段与应用程序并发执行，不需要STW停顿。JVM在此阶段删除不再使用的对象，并回收他们占用的内存空间。
       阶段 6: Concurrent Reset（并发重置）：此阶段与应用程序并发执行，重置CMS算法相关的内部数据，为下一次GC循环做准备。


###### G1 GC

> 开启G1 GC

```shell
XX:+UseG1GC 启动G1 GC
-XX:MaxGCPauseMillis=50 期望最大停顿时间
```

G1收集器的设计目标是取代CMS，G1相比CMS存在两个出色表现：G1有一个内存整理过程，不会产生过多的内存碎片；G1的STW时间更加可控，G1的STW时间上添加了预测机制，用户可以指定期望停顿时间。

G1收集器引入了以下概念：

1. Region：传统的收集器将连续的堆内存进行分代（年轻代，老年代。。。。），这种划分下，各代的存储地址是连续的，如下描述。
｜<-新生代->|<-老年代->|<-matespace（或永久代））->|
而G1收集器各个代的存储地址是不连续的，每个代都是用了n个不连续的大小相同的Region，每个Region占有一块连续的虚拟地址。大小大于等于Region一般的对象称作巨型对象（humongous object），巨型对象会直接分配在old gen中，巨型对象在并发标记和full GC阶段回收，在分配巨型对象之前先检查是否超过，initiating heap occupancy percent和the marking threshold，如果超过的话，会进行并发标记提早进行垃圾回收，防止出现full GC。为了减少连续H-objs分配对GC的影响，需要把大对象变为普通的对象，建议增大Region size。
2. SATB：全称是Snapshot-At-The-Beginning，GC开始时存活对象的快照，作用是维持并发GC的正确性。
3. RSet：全称是Remembered Set，是辅助GC过程的一种结构，典型的空间换时间工具，和Card Table有些类似。还有一种数据结构也是辅助GC的：Collection Set（CSet），它记录了GC要收集的Region集合，集合里的Region可以是任意年代的。在GC的时候，对于old->young和old->old的跨代对象引用，只要扫描对应的CSet中的RSet即可。
4. Pause Prediction Model：即停顿预测模型 

> G1 GC配置参数

```github
-XX：+UseG1GC：启用 G1 GC； -XX：G1NewSizePercent：初始年轻代占整个 Java Heap 的大小，默认值为 5%； -XX：G1MaxNewSizePercent：最大年轻代占整个Java Heap的大小，默认值为60%；
-XX：G1HeapRegionSize：设置每个 Region 的大小，单位 MB，需要为 1，2，4，8，16，32 中的某个值，默 认是堆内存的 1/2000。如果这个值设置比较大，那么大对象就可以进入Region了。

-XX：ConcGCThreads：与Java应用一起执行的 GC 线程数量，默认是Java线程的1/4，减少这个参数的数值可能会提升并行回收的效率，提高系统内部吞吐量。如果这个数值过低，参与回收垃圾的线程
不足，也会导致并行回收机制耗时加长。

-XX：+InitiatingHeapOccupancyPercent（简称 IHOP）：G1内部并行回收循环启动的阈值，默认为Java Heap的45%。这个可以理解为老年代使用大于等于45%的时候，JVM会启动垃圾回收。这个
值非常重要，它决定了在什么时间启动老年代的并行回收。

-XX：G1HeapWastePercent：G1停止回收的最小内存大小，默认是堆大小的5%。GC会收集所有的Region中 的对象，但是如果下降到了5%，就会停下来不再收集了。就是说，不必每次回收就把所有的垃圾
都处理完，可以遗留少量的下次处理，这样也降低了单次消耗的时间。

-XX：G1MixedGCCountTarget：设置并行循环之后需要有多少个混合GC启动，默认值是8个。老年代Regions的回收时间通常比年轻代的收集时间要长一些。所以如果混合收集器比较多，可以允许G1延长
老年代的收集时间。

-XX：G1ReservePercent参数的一些关键信息：示例：-XX:G1ReservePercent=15

含义： G1ReservePercent参数用于指定G1垃圾收集器应该为未来的垃圾收集保留多少堆内存空间。这个保留的空间不会被用于Java对象的分配，而是用于在垃圾收集期间进行整理、复制和移动对象。
默认值： 如果不显式设置该参数，G1垃圾收集器将使用默认值。默认值通常是堆内存的5%。
单位： 该参数的值是一个百分比，表示为一个整数。例如，G1ReservePercent=5表示将5%的堆内存保留给G1垃圾收集器。
使用场景： 适当调整G1ReservePercent参数可以影响G1垃圾收集器的性能，尤其是在长时间运行的大型应用程序中。通过增加保留的堆内存百分比，可以减少垃圾收集期间的复制和移动对象的次数，从而降低停顿时间。

G1HeapRegionSize参数的一些关键信息：示例：-XX:G1HeapRegionSize=2M

含义： G1HeapRegionSize参数用于设置G1垃圾收集器中的堆区域的大小。G1垃圾收集器采用了基于区域的内存管理模型，通过将整个堆分割成许多小的连续区域，以更好地实现可预测的停顿时间和更高的吞吐量。
默认值： 如果不显式设置该参数，G1垃圾收集器将使用默认值。默认值通常是1MB。
单位： 该参数的值是以字节为单位的整数。例如，G1HeapRegionSize=2M表示将堆区域的大小设置为2兆字节。
影响： 堆区域的大小直接影响到G1垃圾收集器的行为，包括垃圾收集的粒度、并行度等。更小的区域大小可能会导致更频繁但更短的垃圾收集暂停，而更大的区域大小可能会导致较长的垃圾收集暂停，但更少的频率。

 -XX:InitiatingHeapOccupancyPercent 参数的一些关键信息：示例：-XX:InitiatingHeapOccupancyPercent=50

含义： 该参数指定了堆内存的占用百分比，当达到该百分比时，G1 将启动 Mixed GC。Mixed GC 是 G1 垃圾收集器中的一种混合收集，旨在对老年代和年轻代的垃圾进行同时处理。
默认值： 如果不显式设置该参数，G1 垃圾收集器将使用默认值。默认值通常是 45%，即当堆内存占用达到 45% 时，G1 将启动 Mixed GC。
单位： 该参数的值是一个整数，表示堆内存占用的百分比。例如，-XX:InitiatingHeapOccupancyPercent=50 表示当堆内存占用达到 50% 时，G1 将启动 Mixed GC。
影响： 调整该参数的值可以影响 G1 垃圾收集器的行为，特别是与 Mixed GC 相关的操作。增加百分比可能会导致 G1 更早地启动 Mixed GC，这有助于减少 Full GC 的频率。

```

> G1 GC处理步骤

- **1、年轻代模式转移暂停（Evacuation Pause）**

&nbsp; &nbsp; G1 GC 会通过前面一段时间的运行情况来不断的调整自己的回收策略和行为，以此来比较稳定地控制暂停时间。在应用程序刚启动时，G1 还没有采集到什么足够的信息，这时候就处于初始的 fullyyoung 模式。当年轻代空间用满后，应用线程会被暂停，年轻代内存块中的存活对象被拷贝到存活区。如果还没有存活区，则任意选择一部分空闲的内存块作为存活区。拷贝的过程称为转移（Evacuation)，这和前面介绍的其他年轻代收集器是一样的工作原理。

- **2、并发标记-当发生（Humongous Allocation）时**

G1 GC的很多概念建立在CMS的基础上，所以下面的内容需要对CMS有一定的理解。G1并发标记的过程与CMS基本上是一样的。G1的并发标记通过Snapshot-At-The-Beginning（起始快照）的方式，在标记阶段开始时记下所有的存活对象。即使在标记的同时又有一些变成了垃圾。通过对象的存活信息，可以构建出每个小堆块的存活状态，以便回收集能高效地进行选择。这些信息在接下来的阶段会用来执行老年代区域的垃圾收集。有两种情况是可以完全并发执行的：

一、如果在标记阶段确定某个小堆块中没有存活对象，只包含垃圾；
二、在 STW 转移暂停期间，同时包含垃圾和存活对象的老年代小堆块。

当堆内存的总体使用比例达到一定数值，就会触发并发标记。这个默认比例是 45%，但也可以通过 JVM参数 InitiatingHeapOccupancyPercent 来设置。和 CMS 一样，G1 的并发标记也是由多个阶段组成，其中一些阶段是完全并发的，还有一些阶段则会暂停应用线程。

1. 阶段一：**Initial Mark**（初始标记）此阶段标记所有从GC 跟对象直接可达的对象
2. 阶段二：**Root Region Scan**（Root区扫描）此阶段标记所有从根区域可达的对象；根区域：非空的区域，以及在标记过程中不得不收集的区域
3. 阶段三：**Concurrent Mark**（并发标记）此阶段和CMS的并发标记类似，只遍历对象图并在一个特殊位置标记能访问到的对象
4. 阶段四：**Remark**（再次标记）和CMS类似，该阶段不是并发执行，会发生STW，G1 GC短暂的停止应用线程，停止并发更新信息写入，处理器中少量信息，并标记所有在并发标记开始时未被标记的存活对象。
5. 阶段五：**Cleanup**（清理）最后这个清理阶段为即将到来的转移阶段做准备，统计小堆块中所有存活的对象，并将小堆块进行排序，以提升
GC 的效率，维护并发标记的内部状态。 所有不包含存活对象的小堆块在此阶段都被回收了。有一部分任务是并发的：例如空堆区的回收，还有大部分的存活率计算。此阶段也需要一个短暂的 STW 暂停。

- **3、转移暂停：混合模式（Evacuation Pause（mixed））**

并发标记完成之后，G1将执行一次混合收集（mixed collection），就是不只清理年轻代，还将一部分老年代区域也加入到回收集中。混合模式的转移暂停不一定紧跟并发标记阶段。有很多规则和历史数据会影响混合模式的启动时机。比如，假若在老年代中可以并发地腾出很多的小堆块，就没有必要启动混合模式。因此，在并发标记与混合转移暂停之间，很可能会存在多次`young`模式的转移暂停。具体添加到回收集的老年代小堆块的大小及顺序，也是基于许多规则来判定的。其中包括指定的软实时性能指标，存活性，以及在并发标记期间收集的`GC`效率等数据，外加一些可配置的JVM选项。混合收集的过程，很大程度上和前面的`fully-young gc`是一样的。

> G1 GC的注意事项

特别需要注意的是，某些情况下 G1 触发了 Full GC，这时 G1 会退化使用 Serial 收集器来完成垃圾的清理工作， 它仅仅使用单线程来完成 GC 工作，GC 暂停时间将达到秒级别的。
1. 并发模式失败
G1 启动标记周期，但在 Mix GC 之前，老年代就被填满，这时候 G1 会放弃标记周期。解决办法：增加堆大小， 或者调整周期（例如增加线程数-XX：ConcGCThreads 等）。

2. 晋升失败
没有足够的内存供存活对象或晋升对象使用，由此触发了 Full GC(to-space exhausted/to-space overflow）。

解决办法：

a) 增加 –XX：G1ReservePercent 选项的值（并相应增加总的堆大小）增加预留内存量。
b) 通过减少 –XX：InitiatingHeapOccupancyPercent 提前启动标记周期。
c) 也可以通过增加 –XX：ConcGCThreads 选项的值来增加并行标记线程的数目。

3. 巨型对象分配失败
当巨型对象找不到合适的空间进行分配时，就会启动 Full GC，来释放空间。
解决办法：增加内存或者增大 -XX：G1HeapRegionSize
4. g1垃圾收集器Evacuation Pause时间长原因

内存碎片： G1垃圾收集器会尽量减少内存碎片，但是在长时间运行后，由于对象的分配和释放，可能会导致一些内存碎片的产生。这可能会增加垃圾收集的复杂性，导致Evacuation Pause时间增长。
晋升失败： 当一个对象在年轻代经历了一定次数的垃圾收集后仍然存活，它会被晋升到老年代。如果老年代的空间不足，可能会触发Full GC（Full Garbage Collection），导致较长的停顿时间。
应用程序负载： 如果应用程序的负载突然增加，导致更多的对象被分配到堆中，G1垃圾收集器可能需要更频繁地执行垃圾收集操作，从而增加Evacuation Pause时间。
G1收集器配置不当： G1垃圾收集器有许多配置选项，包括堆大小、并行度、阈值等等。如果配置不当，可能会导致垃圾收集器无法有效地执行垃圾收集操作，从而影响Evacuation Pause时间。

- 为了解决Evacuation Pause时间较长的问题，你可以考虑进行以下操作：

调整堆大小： 确保为应用程序分配足够的堆内存，以减少频繁的垃圾收集操作。
调整G1垃圾收集器的参数： 根据应用程序的需求，调整G1垃圾收集器的配置参数，以优化垃圾收集性能。
监控和分析： 使用监控工具和分析工具来监视应用程序的垃圾收集行为，识别性能瓶颈并进行优化。


#### JVM参数配置visualvm远程

-Djava.rmi.server.hostname=172.18.215.59
-Dcom.sun.management.jmxremote.port=1232
-Dcom.sun.management.jmxremote.rmi.port=1240
-Dcom.sun.management.jmxremote.ssl=false
-Dcom.sun.management.jmxremote.authenticate=false