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

- 开启G1 GC

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

- G1 GC配置参数

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
```