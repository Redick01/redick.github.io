# Netty基础

## Netty核心对象
    1、BootStrap：启动类，挂载所有核心对象

    2、EventLoot：核心的Reactor

    3、Channel：网络通信主体

    4、Handler：处理器

## Netty核心对象
    1、高吞吐
    2、低延迟
    3、低开销
    4、零拷贝
    5、可扩容
    6、松耦合：网络和业务分离
    7、使用方便，可维护性好

## Netty基本概念
- Channel：通道，Java NIO 中的基础概念,代表一个打开的连接,可执行读取/写入 IO 操作。 Netty 对 Channel 的所有 IO 操作都是非阻塞的。
- ChannelFuture：Java 的 Future 接口，只能查询操作的完成情况, 或者阻塞当前线程等待操作完成。 Netty 封装一个 ChannelFuture 接口。
                 我们可以将回调方法传给 ChannelFuture，在操作完成时自动执行。
- Event & Handler：Netty 基于事件驱动，事件和处理器可以关联到入站和出站数据流。
- Encoder & Decoder：处理网络 IO 时，需要进行序列化和反序列化, 转换 Java 对象与字节流。 对入站数据进行解码, 基类是 ByteToMessageDecoder。 对出站数据进行编码, 基类是 MessageToByteEncoder。
- ChannelPipeline：数据处理管道就是事件处理器链。有顺序、同一 Channel 的出站处理器和入站处理器在同一个列表中。


## Netty运行原理
- BossGroup：负责接收请求，也是一个EventLoopGroup，多路复用的将请求分发给WorkerGroup。
- WorkerGroup：接收到BossGroup的请求，多路复用的检查Channel，其本身也是一个EventLoopGroup。
- Channel：通道，执行IO操作的通道，执行WorkerGroup分发下来的请求。
- ChannelHandler：处理Channel上的请求，是一个处理器，基于事件驱动的。
- ChannelPipeLine：处理器链，对整个Channel中出站、入站处理器的链。
- 当业务请求很大的时候，WorkerGroup不足以处理的时候，可以自定义线程池，处理Handler。

## Netty关键对象
- Bootstrap: 启动线程，开启socket 
- EventLoopGroup：BossGroup，WorkerGroup
- EventLoop：BossGroup或WorkerGroup中的多路复用的一个Reactor线程
- SocketChannel: 连接通道，被多路复用的Reactor打开的连接，可执行读取/写入操作
- ChannelInitializer: 通道初始化，初始化处理器链
- ChannelPipeline: 处理器链 
- ChannelHandler: 处理器

## Event & Handler
    入站事件:
    • 通道激活和停用 • 读操作事件
    • 异常事件
    • 用户事件 出站事件:
    • 打开连接
    • 关闭连接
    • 写入数据
    • 刷新数据
    事件处理程序接口:
    • ChannelHandler
    • ChannelOutboundHandler
    • ChannelInboundHandler 适配器(空实现，需要继承使用):
    • ChannelInboundHandlerAdapter
    • ChannelOutboundHandlerAdapter
    Netty应用组成:
    • 网络事件
    • 应用程序逻辑事件 
    • 事件处理程序

## Netty是如何实现高性能的
    1、我的理解是通过Reacotr模型，实现多路复用，并且在Channel中（理解为连接）的操作都是非阻塞的，实现事件驱动模型处理数据。
    2、零拷贝：使用直接内存代替接收、发送缓冲区，避免了内存复制，提升了I/O读取和写入性能
    3、支持通过内存池的方式循环利用ByteBuf
    4、系统参数优化：TCP参数，I/O线程数可配置，满足不同业务场景
    5、心跳周期优化
    6、关键资源的处理使用单线程串行化的方式,避免多线程并发访问带来的锁竞争和额外的CPU资源损耗问题
    7、通过引用计数器及时的申请释放不再被引用的对象,细粒度的内存管理降低了GC的频率,减少了频繁GC带来的时延增大和CPU损耗
    8、合理地使用线程安全容器,原子类等,提升系统的并发处理能力
    9、采用环形数组缓冲区实现无锁化并发编程,替代传统的线程安全容器或锁

## Reactor三种模式
    1、单线程：一个线程什么都做
    2、多线程：多个线程做
    3、主从Reactor多线程模式：一个或多个线程专门hold请求，然后分发请求的模式

    