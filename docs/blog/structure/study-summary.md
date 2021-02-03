# Java 0期毕业总结

## JVM

### java字节码分类

- 栈操作指令，包括与局部变量交互的指令
- 程序流程控制指令
- 对象操作指令，包括方法调用指令
- 算术运算以及类型转换指令

### JVM类加载器

- Java类加载器

	- 启动类加载器（BootStrapClassLoader），负责加载核心java包，如：jdk的lib包下rt.jar
	- 扩展类加载器（ExtClassLoader）：负责加载扩展的java包，如：jdk的lib/ext下的jar
	- 应用类加载器（AppClassLoader）：我们写的类由它加载

- Java类加载器特点

	- 双亲委派：简单说就是加载类的时候如果当前加载器没有加载会问父加载器加载没有，加载了就直接用，没加载继续向上问都没加载就加载
	- 负责依赖
	- 缓存加载

- Java类加载过程

	- 加载（Loading）：找class文件在哪
	- 验证（Verification）：验证格式、依赖
	- 准备（Preparation）：静态字段、方法表、类结构
	- 解析（Resolution）：符号解析为引用
	- 初始化（Initialization）：构造器、静态变量赋值、静态代码块
	- 使用（Using）
	- 卸载（Unloading）

- Java类加载时机

	- 当虚拟机启动时，初始化用户指定的主类，就是启动执行的 main方法所在的类；
	- 当遇到用以新建目标类实例的 new 指令时，初始化 new 指令的目标类，就是new一
个类的时候要初始化；
	- 当遇到调用静态方法的指令时，初始化该静态方法所在的类；
	- 当遇到访问静态字段的指令时，初始化该静态字段所在的类；
	- 子类的初始化会触发父类的初始化；
	- 如果一个接口定义了 default 方法，那么直接实现或者间接实现该接口的类的初始化，
会触发该接口的初始化；
	- 使用反射 API 对某个类进行反射调用时，初始化这个类，其实跟前面一样，反射调用
要么是已经有实例了，要么是静态方法，都需要初始化；
	- 当初次调用 MethodHandle 实例时，初始化该 MethodHandle 指向的方法所在的
类。

- 类不会初始化（可能会加载）

	- 通过子类引用父类的静态字段，只会触发父类的初始化，而不会触发子类的初始化。
	- 定义对象数组，不会触发该类的初始化。
	- 常量在编译期间会存入调用类的常量池中，本质上并没有直接引用定义常量的类，不
会触发定义常量所在的类。
	- 通过类名获取Class对象，不会触发类的初始化，Hello.class不会让Hello类初始化。
	- 通过Class.forName加载指定类时，如果指定参数initialize为false时，也不会触发
类初始化，其实这个参数是告诉虚拟机，是否要对类进行初始化。
	- 通过ClassLoader默认的loadClass方法，也不会触发初始化动作（加载了，但是不
初始化）。

### JVM内存模型

- 堆（Heap）：线程共享的区域，所有实例对象都在堆上分配内存，该区域也是GC主要管理的内存区，堆内存又将内存空间分成新生代（Eden Space）、老年代（Old Space）、幸存者区（Surviror），这么做的目的是GC算法的分代收集
- 虚拟机栈：JVM虚拟机栈属于线程私有的，每个线程只能访问自己的线程栈，主要存储，类信息、存储局部变量表、操作栈、动态链接、方法出口，对象指针
- 本地方法栈：线程私有。为虚拟机使用到的Native 方法服务。如Java使用c或者c++编写的接口服务时，代码在此区运行
- 程序计数器：线程私有。有些文章也翻译成PC寄存器（PC Register），同一个东西。它可以看作是当前线程所执行的字节码的行号指示器。指向下一条要执行的指令
- 元数据空间（MateSpace）：java8以后用于替代方法区的内存区域，线程共享的，主要用于存储类信息，常量，方法数据等，元数据空间的大小取决于本地内存，也就是说只要本地内存够大就不会出现内存不够用的情况，替代方法区也避免了“java.lang.OutOfMemoryError: PermGen space”
- 堆外内存：堆外内存主要是通过本地方法申请的直接内存

### JVM命令行工具

- jps/jinfo：查看JVM进程信息命令
- jstat：查看JVM内部GC相关信息命令
- jmap：查看堆或类占用空间统计
- jstack：查看java线程情况
- jcmd：可以综合以上命令，是一个整合命令，可以通过 jcm pid help查看可以使用的命令
- jrunscript/jjs：jrunscript可以执行js文件、脚本段并且可以充当curl使用，jjs很有意思可以执行脚本

### JVM图形化工具

- JConsole
- Jvisualvm
- VisualGC：有ieda插件
- jmc

### GC算法

- GC算法演进

	- 引用计数法：简餐粗暴，但是有可能产生循环引用使得永远不会GC就会导致内存泄漏，最终可能导致内存溢出；
	- 标记清除法：遍历所有可达对象并进行标记，清除不可达对象，但是会产生内存随便，造成内存不连续；
	- 标记清楚整理法：在标记清楚基础上进行内存压缩，防止产生内存碎片

- 分代收集

	- 将堆分为新生代和老年代，使用不同的垃圾回收算法，在新生代中分为Eden区和两个Survivor区，两个存活区 from 和 to，互换角色。对象存活到一定周期会提升到老年代，晋升到老年代的策略有-XX:+MaxTenuringThreshold参数控制
老年代默认都是存活的对象象，采用移动方式

		- 标记所有通过 GC roots 可达的对象；
		- 删除所有不可达对象；
		- 整理老年代空间中的内容，方法是将所有的存活对象复制，从老年代空间开始的地方依次存放。

	- 可以作为GC Roots的对象

		- 当前正在执行的方法里的局部变量和输入参数
		- 活动线程（Active threads）
		- 所有类的静态字段（static field）
		- JNI引用此阶段暂停的时间，与堆内存大小,对象的总数没有直接关系，而是由存活对象（alive objects）的数量来决定。所以增加堆内存的大小并不会直接影响标记阶段占用的时间。

### GC垃圾收集器

- 串行GC（SerialGC）

	- 单线程
	- 年轻代使用 mark-copy（标记-复制） 算法
	- 老年代使用 mark-sweep-compact（标记-清除-整理）算法。两者都是单线程的垃圾收集器，
不能进行并行处理，所以都会触发全线暂停（STW）

- 并行GC（ParallelGC）

	- 多线程并行收集
	- 年轻代使用 标记-复制（mark-copy）算法
	- 老年代使用 标记-清除-整理（mark-sweepcompact）算法

- CMS GC

	- 年轻代采用并行 STW 方式的 mark-copy (标记-复制)算法
	- 老年代主要使用并发 marksweep (标记-清除)算法

		- 阶段 1: Initial Mark（初始标记）：该阶段STW，初始标记的目标是标记所有的根对象，包括根对象直接引用的对象，以及被年轻代中所有存活对象所引用的对象（老年代单独回收）
		- 阶段 2: Concurrent Mark（并发标记）：此阶段，CMS遍历老年代，标记所有存活对象
		- 阶段 3: Concurrent Preclean（并发预清理）：此阶段同样是与应用线程并发执行的，不需要停止应用线程。 因为前一阶段【并发标记】与程序并发运行，可能有一些引用关系已经发生了改变。如果在并发标记过程中引用关系发生了变化，JVM 会通过“Card（卡片）”的方式将发生了改变的区域标记为“脏”区，这就是所谓的 卡片标记（Card Marking）
		- 阶段 4: Final Remark（最终标记）：最终标记阶段是此次 GC 事件中的第二次（也是最后一次）STW 停顿。本阶段的目标是完成老年代中所有存活对象的标记. 因为之前的预清理阶段是并发执行的，有可能 GC 线程跟不上应用程序的修改速度。所以需要一次STW 暂停来处理各种复杂的情况。通常 CMS 会尝试在年轻代尽可能空的情况下执行 Final Remark 阶段，以免连续触发多次 STW 事件
		- 阶段 5: Concurrent Sweep（并发清除）：此阶段与应用程序并发执行，不需要 STW 停顿。JVM 在此阶段删除不再使用的对象，并回收他们占用的内存空间
		- 阶段 6: Concurrent Reset（并发重置）：此阶段与应用程序并发执行，重置 CMS 算法相关的内部数据，为下一次 GC 循环做准备

- G1 GC

	- G1 GC新概念

		- Region：G1将堆内存分成很多个大小相同的【小区域、小块】(region)，region大小1MB到32MB，总数一般不会超过2048
		- SATB，（Snapshot-At-The-Beginning），在标记周期开始时，对堆内存中的存活对象信息进行一次快照，作用是维持并发GC的正确性
		- CSet，采用【增量并行复制】的方式来实现【堆内存碎片整理功能】，将回收集之中的存活对象拷贝到新region中，回收集的英文是 Collection Set，简称CSet，也就是本次GC涉及的region集合
		- RSet，G1为每个region都单独设置了一份【记忆集】，英文是 Remembered Set，简称 RSet, 用来跟踪记录从别的region指向这个region中的引用
		- Pause Prediction Model：即停顿预测模型 

	- G1 GC收集模式

		- Young GC：该种模式GC选定所有年轻代Region，通过控制年轻代Region的个数（或者说大小）来控制Young GC时间开销
		- Mixed GC：顾名思义，混合型GC，该种模式选定所有年轻代Region，外加根据global concurrent marking统计得出的收益较高的若干老年代Region，在用户指定的开销目标范围内尽可能选择收益高的老年代Region。

- ZGC

	- 实现

		- 通过着色指针和读屏障，实现几乎全部的并发执行，几毫秒级别的延迟，线性可扩展

	- 特点

		- GC 最大停顿时间不超过 10ms
		- 堆内存支持范围广，小至几百 MB 的堆空间，大至4TB 的超大堆内存（JDK13升至16TB）
		- 与 G1 相比，应用吞吐量下降不超过15%
		- 当前只支持 Linux/x64 位平台，JDK15后支持MacOS和Windows系统

## NIO

## Spring和ORM等框架

## 分布式消息队列

### JMS

- JMS优势

	- Asynchronous（异步）
	- Reliable（可靠）

- JMS消息模型

	- Point-to-Point Messaging Domain （点对点）
	- Publish/Subscribe Messaging Domain （发布/订阅模式）

- JMS消费方式

	- 同步（Synchronous）
	- 异步（Asynchronous）

- JMS编程模型

	- 管理对象（Administered objects）-连接工厂（Connection Factories）和目的地（Destination
	- 连接对象（Connections）
	- 会话（Sessions）
	- 消息生产者（Message Producers）
	- 消息消费者（Message Consumers）
	- 消息监听者（Message Listeners）

### 应用场景

- 削峰填谷

  比如如秒杀等大型活动时会带来较高的流量脉冲，如果没做相应的保护，将导致系统超负荷甚至崩溃。如果因限制太过导致请求大量失败而影响用户体验，可以利用MQ 超高性能的消息处理能力来解决

- 异步解耦
- 顺序消息

  与FIFO原理类似，MQ提供的顺序消息即保证消息的先进先出，可以应用于交易系统中的订单创建、支付、退款等流程

- 分布式事务消息

### ActiveMQ

- ActiveMQ的消息传递模式

	- P2P （Queue）

		- 客户端包括生产者和消费者
		- 队列中的消息只能被一个消费者消费
		- 消费者可以随时消费队列中的消息

	- Pub/Sub（发布/订阅，Publish/Subscribe） Topic模式

		- 客户端包括发布者和订阅者
		- Topic（主题）中消息被所有订阅者消费
		- 消费者不能消费在订阅之前就发送到主题中的消息，也就是说，消费者要先于生产者启动

### Kafka

- Kafka的基本概念

	- Broker：Kafka集群包含一个或多个服务器，这种服务器被称为broker
	- Topic：每条发布到Kafka集群的消息都有一个类别，这个类别被称为Topic
	- Partition：Partition是物理上的概念，每个Topic包含一个或多个Partition
	- Producer：负责发布消息到Kafkabroker
	- Consumer：消息消费者，向Kafkabroker读取消息的客户端
	- ConsumerGroup：每个Consumer属于一个特定的ConsumerGroup

- Kafka集群部署结构

	- Kafka集群部署需要一个zk来记录状态信息
	- Kafka一致性重要机制之ISR(kafka replica)
	- Kafka服务端rebalance

- Topic特性

	- 通过partition增加可扩展性，线上操作会引起性能抖动
	- 通过顺序写入达到高吞吐
	- 多副本增加容错性

- kafka高级特性

	- 生产者高级特性

		- 生产者-确认模式

		  ack=0 : 只发送不管有没有写入到broker 
		  ack=1：写入到leader就认为成功 
		  ack=all：写入到最小的复本数则认为成功

		- 生产者特性-同步发送

		  KafkaProducer kafkaProducer = new KafkaProducer(pro); 
		  ProducerRecord record = new ProducerRecord("topic", "key", "value"); 
		  Future future = kafkaProducer.send(record); 
		  //同步发送方法1
		  Object o = future.get(); 
		  //同步发送方法2 
		  kafkaProducer.flush();

		- 生产者特性-异步发送

		  pro.put("linger.ms", “1"); 
		  pro.put("batch.size", "10240"); 
		  KafkaProducer kafkaProducer = new KafkaProducer(pro); 
		  ProducerRecord record = new ProducerRecord("topic", "key", "value"); 
		  Future future = kafkaProducer.send(record); 
		  //异步发送方法1 
		  kafkaProducer.send(record, (metadata, exception) -> { if (exception == null) System.out.println("record = " + record); }); 
		  //异步发送方法2 
		  kafkaProducer.send(record);

		- 生产者特性-顺序保证

		  顺序保证 
		  pro.put("max.in.flight.requests.per.connection", “1"); 
		  KafkaProducer kafkaProducer = new KafkaProducer(pro); 
		  ProducerRecord record = new ProducerRecord("topic", "key", "value"); 
		  Future future = kafkaProducer.send(record); 
		  //同步发送 
		  kafkaProducer.send(record); 
		  kafkaProducer.flush();

		- 生产者特性-消息可靠性传递

		  顺序保证 
		  pro.put("max.in.flight.requests.per.connection", “1"); 
		  KafkaProducer kafkaProducer = new KafkaProducer(pro); 
		  ProducerRecord record = new ProducerRecord("topic", "key", "value"); 
		  Future future = kafkaProducer.send(record); 
		  //同步发送 
		  kafkaProducer.send(record); 
		  kafkaProducer.flush();

	- 消费者高级特性

		- offset同步提交

		  props.put("enable.auto.commit","false"); 
		  while (true) { 
		      //拉取数据 
		      ConsumerRecords poll = consumer.poll(Duration.ofMillis(100)); 
		      poll.forEach(o -> { 
		          ConsumerRecord<String, String> record = (ConsumerRecord) 
		          o; Order order = JSON.parseObject(record.value(), Order.class); 
		          System.out.println("order = " + order); }); 
		      consumer.commitSync(); 
		  }

		- offset异步提交

		  props.put(“enable.auto.commit","false"); 
		  while (true) { 
		      //拉取数据 
		      ConsumerRecords poll = consumer.poll(Duration.ofMillis(100)); 
		      poll.forEach(o -> { 
		          ConsumerRecord<String, String> record = (ConsumerRecord) o; 
		          Order order = JSON.parseObject(record.value(), Order.class); 
		          System.out.println("order = " + order); 
		      }); 
		      consumer.commitAsync(); 
		  }

		- offset自动提交

		  props.put("enable.auto.commit","true"); 
		  props.put(“auto.commit.interval.ms”,”5000"); 
		  while (true) { 
		      //拉取数据 
		      ConsumerRecords poll = consumer.poll(Duration.ofMillis(100)); 
		      poll.forEach(o -> { 
		          ConsumerRecord<String, String> record = (ConsumerRecord) o; 
		          Order order = JSON.parseObject(record.value(), Order.class); 
		          System.out.println("order = " + order); 
		      }); 
		  }

		- offset Seek

		  props.put("enable.auto.commit","true"); 
		  //订阅topic 
		  consumer.subscribe(Arrays.asList("demo-source"), new ConsumerRebalanceListener() { 
		      @Override 
		      public void onPartitionsRevoked(Collection<TopicPartition> partitions) { 
		          commitOffsetToDB(); 
		      }
		      @Override 
		      public void onPartitionsAssigned(Collection<TopicPartition> partitions) { 
		          partitions.forEach(topicPartition -> consumer.seek(topicPartition, getOffsetFromDB(topicPartition)));
		      } 
		  });
		  
		  while (true) { 
		      //拉取数据 
		      ConsumerRecords poll = consumer.poll(Duration.ofMillis(100)); 
		      poll.forEach(o -> { 
		          ConsumerRecord<String, String> record = (ConsumerRecord) o; 
		          processRecord(record); saveRecordAndOffsetInDB(record, record.offset());
		      }); 
		  }

### RabbitMQ

- queue
- exchange
- routekey
- binding

### RocketMQ

- RocketMQ组成

	- NameServer：主要负责对于源数据的管理，包括了对于Topic和路由信息的管理
	- Producer消息生产者，负责产生消息，一般由业务系统负责产生消息
	- Broker消息中转角色，负责存储消息，转发消息
	- Consumer消息消费者，负责消费消息，一般是后台系统负责异步消费

- RocketMQ的消息领域模型

	- Topic：消息的第一级类型，最细粒度的订阅单位，一个Group可以订阅多个Topic的消息
	- Tag：消息的第二级类型，RocketMQ提供2级消息分类，方便灵活控制
	- Group：组，一个组可以订阅多个Topic
	- Message Queue：消息的物理管理单位

- RocketMQ的关键特性

	- 消息的顺序
	- 消息重复 消息投递Qos

		- 最多一次（At most once）
		- 至少一次（At least once）
		- 仅一次（ Exactly once）

	- 消息去重

		- 业务端实现幂等（RoilingBitmap）
		- 保证每条消息都有唯一编号(比如唯一流水号)，且保证消息处理成功与去重表的日志同时出现

## 数据库 MySQL

### 存储引擎

- MyISAM
- InnoDB
- MEMORY（内存型）
- Archive（归档）

### 索引

- InnoDB使用B+树实现

### 事务

InnoDB: 
双写缓冲区、故障恢复、操作系统、fsync() 、磁盘存储、缓存、UPS、网络、备份策略

- 事务特性

	- Atomicity: 原子性, 一次事务中的操作要么全部成功, 要么全部失败
	- Consistency: 一致性, 跨表、跨行、跨事务, 数据库始终保持一致状态
	- Isolation: 隔离性, 可见性, 保护事务不会互相干扰, 包含4种隔离级别
	- Durability:, 持久性, 事务提交成功后,不会丢数据。如电源故障, 系统崩溃

- 锁

	- 表级锁
	- 行级锁（InnoDB）
	- 乐观锁
	- 悲观锁

- 事务隔离级别

	- 读未提交: READ UNCOMMITTED
	- 读已提交: READ COMMITTED
	- 可重复读: REPEATABLE READ
	- 可串行化: SERIALIZABLE

- 事务实现

	- undo log: 撤销日志
	- redo log: 重做日志
	- MVCC: 多版本并发控制

### 高可用

- Master/Slave

  核心是
  1、主库写 binlog
  2、从库 relay log

- MGR（组复制）

	- 特点

		- 高一致性：基于分布式Paxos协议实现，保证数据一致性
		- 高容错性：自定检测机制，只要不是大多数节点宕机就可以继续工作，内置防脑裂保护机制
		- 高扩展性：节点的增加与移除会自动更新组员信息，新节点加入后自动从其他节点同步增量数据直到与其他节点数据一致
		- 高灵活性：提供单主模式和多主模式

	- 使用场景

		- 弹性复制
		- 高可用分片

- MHA
- Cluster

### 分库分表

- 拆分思路

	- 垂直拆分（拆库）
	- 水平拆分（拆表）

- 中间件

	- apache-shardingsphere-proxy
	- DRDS（商业闭源）
	- MyCat/DBLE
	- Cobar
	- Vitness
	- KingShard

### 分布式事务

- 强一致性：XA
- BASE柔性事务

	- TCC（Try，Confirm，Cancel）
	- AT（二段提交，自动生成反向SQL）
	- 柔性事务特性 

		- 原子性（Atomicity）：正常情况下保证
		- 一致性（Consistency），在某个时间点，会出现A库和B库的数据违反一致性要求的情况，但是最终是一 致的
		- 隔离性（Isolation），在某个时间点，A事务能够读到B事务部分提交的结果
		- 持久性（Durability），和本地事务一样，只要commit则数据被持久

	- 隔离级别

		- 一般情况下都是读已提交（全局锁）、读未提交（无全局锁）

## RPC和微服务

## 分布式缓存

### 缓存中间件

- Redis

	- Redis的5种基本数据结构

		- 字符串
		- 散列
		- 列表，类比java的LinkedList
		- 集合，类比java的set，不允许存在重复元素
		- 有序集合（sorted set）

	- Redis的3种高级数据结构

		- Bitmaps

		  bitmaps不是一个真实的数据结构。而是String类型上的一组面向bit操作的集合。由于 strings是二进制安全的blob，并且它们的最大长度是512m，所以bitmaps能最大设置 2^32个不同的bit。

		- Hyperloglogs

		  在redis的实现中，您使用标准错误小于1％的估计度量结束。这个算法的神奇在于不再 需要与需要统计的项相对应的内存，取而代之，使用的内存一直恒定不变。最坏的情况 下只需要12k，就可以计算接近2^64个不同元素的基数

		- GEO

		  Redis的GEO特性在 Redis3.2版本中推出，这个功能可以将用户给定的地理位置（经 度和纬度）信息储存起来，并对这些信息进行操作。

	- Redis线程模型

		- 内存处理单线程，高性能的核心
		- IO线程

			- redis6之前单线程
			- redis6之后，多线程，NIO模型

	- Redis使用场景

		- 业务数据缓存
		- 业务数据处理

		  1、非严格一致性要求的数据：评论，点击等。 
		  2、业务数据去重：订单处理的幂等校验等。 
		  3、业务数据排序：排名，排行榜等。

		- 全局一致计数

		  1、全局流控计数
		  2、秒杀的库存计算 
		  3、抢红包 
		  4、全局ID生成

		- 高效统计计数
		- 分布式锁
		- 发布订阅与Stream，主要用于消息队列

	- Redis高级功能

		- Redis事务

		  开启事务：multi
		  执行事务：exec
		  撤销事务：discard
		  
		  watch实现乐观锁
		  watch一个key，发生变化则事务执行失败

		- Lua脚本支持
		- Redis管道技术，合并操作批量处理，且不阻塞前续命令
		- RDB数据备份与恢复
		- AOF数据备份与恢复

	- Redis集群与高可用

		- Master/Slave
		- Sentinel（哨兵）
		- Cluster集群

- Memcached
- Hazelcast

### 缓存策略

- 容量
- 峰值
- 过期策略

	- FIFO或LRU
	- 按固定日期过期
	- 按业务时间加权

### 缓存常见问题

- 缓存穿透

  大量并发查询不存在的KEY，导致都直接将压力透传到数据库
  
  分析为什么会多次缓存穿透？需要让缓存能够区分KEY不存在和查询到一个空值
  
  解决办法： 
  1、缓存空值的KEY，这样第一次不存在也会被加载会记录，下次拿到有这个KEY。 
  2、Bloom过滤或RoaringBitmap 判断KEY是否存在。
  3、完全以缓存为准，使用延迟异步加载 的按固定时间过期策略，这样就不会触发更新。

- 缓存击穿

  问题：某个KEY实现的时候正好有大量并发请求访问这个KEY
  
  解决办法：
  1.KEY的更新添加全局互斥锁
  2.完全以缓存为准

- 缓存雪崩

  问题：当某一时刻发生大规模的缓存失效的情况，会有大量的请求进来直接打到数据库，导致数 据库压力过大升值宕机。 
  
  分析：一般来说，由于更新策略、或者数据热点、缓存服务宕机等原因，可能会导致缓存数据同 一个时间点大规模不可用，或者都更新。所以，需要我们的更新策略要在时间上合适，数据要均 匀分散，缓存服务器要多台高可用。
  
   解决办法：
  1、更新策略在时间上做到比较均匀。 2、使用的热数据尽量分散到不同的机器上。 
  3、多台机器做主从复制或者多副本，实现高可用。
  4、实现熔断限流机制，对系统进行负载能力控制。

## 并发编程

*XMind: ZEN - Trial Version*