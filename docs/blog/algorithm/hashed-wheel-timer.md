# 时间轮HashedWheelTimer 使用及源码分析<!-- {docsify-ignore-all} -->



本文介绍的 HashedWheelTimer 是来自于 Netty 的工具类，在 netty-common 包中。它用于实现延时任务。另外，下面介绍的内容和 Netty 无关。

如果你看过 Dubbo 的源码，一定会在很多地方看到它。在需要失败重试的场景中，它是一个非常方便好用的工具。

本文将会介绍 HashedWheelTimer 的使用，以及在后半部分分析它的源码实现。



## 接口概览

在介绍它的使用前，先了解一下它的接口定义，以及和它相关的类。

`HashedWheelTimer` 是接口 `io.netty.util.Timer` 的实现，从面向接口编程的角度，我们其实不需要关心 HashedWheelTimer，只需要关心接口类 Timer 就可以了。这个 Timer 接口只有两个方法：

```java
public interface Timer {

    // 创建一个定时任务
    Timeout newTimeout(TimerTask task, long delay, TimeUnit unit);

    // 停止所有的还没有被执行的定时任务
    Set<Timeout> stop();
}
```

Timer 是我们要使用的任务调度器，我们可以从方法上看出，它提交一个任务 TimerTask，返回的是一个 Timeout 实例。所以这三个类之间的关系大概是下面这样的：

![1](https://assets.javadoop.com/imgs/20510079/HashedWheelTimer/1.png)

TimerTask 非常简单，就一个 `run()` 方法：

```java
public interface TimerTask {
    void run(Timeout timeout) throws Exception;
}
```

> 当然这里有点意思的是，它把 Timeout 的实例也传进来了，我们平时的代码习惯，都是单向依赖。
>
> 这样做也有好处，那就是在任务执行过程中，可以通过 timeout 实例来做点其他的事情。

Timeout 也是一个接口类：

```java
public interface Timeout {
    Timer timer();
    TimerTask task();
    boolean isExpired();
    boolean isCancelled();
    boolean cancel();
}
```

它持有上层的 Timer 实例，和下层的 TimerTask 实例，然后取消任务的操作也在这里面。



## HashedWheelTimer 使用

有了第一节介绍的接口信息，其实我们很容易就可以使用它了。我们先来随意写几行：

```java
// 构造一个 Timer 实例
Timer timer = new HashedWheelTimer();

// 提交一个任务，让它在 5s 后执行
Timeout timeout1 = timer.newTimeout(new TimerTask() {
    @Override
    public void run(Timeout timeout) {
        System.out.println("5s 后执行该任务");
    }
}, 5, TimeUnit.SECONDS);

// 再提交一个任务，让它在 10s 后执行
Timeout timeout2 = timer.newTimeout(new TimerTask() {
    @Override
    public void run(Timeout timeout) {
        System.out.println("10s 后执行该任务");
    }
}, 10, TimeUnit.SECONDS);

// 取消掉那个 5s 后执行的任务
if (!timeout1.isExpired()) {
    timeout1.cancel();
}

// 原来那个 5s 后执行的任务，已经取消了。这里我们反悔了，我们要让这个任务在 3s 后执行
// 我们说过 timeout 持有上、下层的实例，所以下面的 timer 也可以写成 timeout1.timer()
timer.newTimeout(timeout1.task(), 3, TimeUnit.SECONDS);
```

通过这几行代码，大家就可以非常熟悉这几个类的使用了，因为它们真的很简单。

我们来看一下 Dubbo 中的一个例子。

下面这个代码修改自 Dubbo 的集群调用策略 `FailbackClusterInvoker` 中：

> 它在调用 provider 失败以后，返回空结果给消费端，然后由后台线程执行定时任务重试，多用于消息通知这种场景。

```java
public class Application {

    public static void main(String[] args) {
        Application app = new Application();
        app.invoke();
    }

    private static final Logger log = LoggerFactory.getLogger(Application.class);

    private volatile Timer failTimer = null;

    public void invoke() {
        try {
            doInvoke();
        } catch (Throwable e) {
            log.error("调用 doInvoke 方法失败，5s 后将进入后台的自动重试，异常信息: ", e);
            addFailed(() -> doInvoke());
        }
    }

    // 实际的业务实现
    private void doInvoke() {
        // 这里让这个方法故意失败
        throw new RuntimeException("故意抛出异常");
    }

    private void addFailed(Runnable task) {
        // 延迟初始化
        if (failTimer == null) {
            synchronized (this) {
                if (failTimer == null) {
                    failTimer = new HashedWheelTimer();
                }
            }
        }
        RetryTimerTask retryTimerTask = new RetryTimerTask(task, 3, 5);
        try {
            // 5s 后执行第一次重试
            failTimer.newTimeout(retryTimerTask, 5, TimeUnit.SECONDS);
        } catch (Throwable e) {
            log.error("提交定时任务失败，exception: ", e);
        }
    }
}
```

下面是里面使用到的 RetryTimerTask 类，当然，你也可以选择写成内部类：

```java
public class RetryTimerTask implements TimerTask {

    private static final Logger log = LoggerFactory.getLogger(RetryTimerTask.class);

    // 每隔几秒执行一次
    private final long tick;

    // 最大重试次数
    private final int retries;

    private int retryTimes = 0;

    private Runnable task;

    public RetryTimerTask(Runnable task, long tick, int retries) {
        this.tick = tick;
        this.retries = retries;
        this.task = task;
    }

    @Override
    public void run(Timeout timeout) {
        try {
            task.run();
        } catch (Throwable e) {
            if ((++retryTimes) >= retries) {
                // 重试次数超过了设置的值
                log.error("失败重试次数超过阈值: {}，不再重试", retries);
            } else {
                log.error("重试失败，继续重试");
                rePut(timeout);
            }
        }
    }

    // 通过 timeout 拿到 timer 实例，重新提交一个定时任务
    private void rePut(Timeout timeout) {
        if (timeout == null) {
            return;
        }
        Timer timer = timeout.timer();
        if (timeout.isCancelled()) {
            return;
        }
        timer.newTimeout(timeout.task(), tick, TimeUnit.SECONDS);
    }
}
```

上面的代码也非常简单，在调用 `doInvoke()` 方法失败以后，提交一个定时任务在 5s 后执行重试，如果还是失败，之后每 3s 重试一次，最多重试 5 次，如果重试 5 次都失败，记录错误日志，不再重试。

打印的日志如下：

```
15:47:36.232 [main] ERROR c.j.n.timer.Application - 调用 doInvoke 方法失败，5s 后将进入后台的自动重试，异常信息: 
java.lang.RuntimeException: 故意抛出异常
    at com.javadoop.nettylearning.timer.Application.doInvoke(Application.java:36)
    at com.javadoop.nettylearning.timer.Application.invoke(Application.java:28)
    at com.javadoop.nettylearning.timer.Application.main(Application.java:19)
15:47:41.793 [pool-1-thread-1] ERROR c.j.n.timer.RetryTimerTask - 重试失败，继续重试
15:47:44.887 [pool-1-thread-1] ERROR c.j.n.timer.RetryTimerTask - 重试失败，继续重试
15:47:47.986 [pool-1-thread-1] ERROR c.j.n.timer.RetryTimerTask - 重试失败，继续重试
15:47:51.084 [pool-1-thread-1] ERROR c.j.n.timer.RetryTimerTask - 重试失败，继续重试
15:47:54.186 [pool-1-thread-1] ERROR c.j.n.timer.RetryTimerTask - 失败重试次数超过阈值: 5，不再重试
```

HashedWheelTimer 的使用确实非常简单，如果你是来学习怎么使用它的，那么看到这里就可以了。



## HashedWheelTimer 源码分析

大家肯定都知道或听说过，它用的是一个叫做时间轮([下载算法介绍PPT](http://www.cse.wustl.edu/~cdgill/courses/cs6874/TimingWheels.ppt))的算法，看下面我画的图：

![4](https://assets.javadoop.com/imgs/20510079/HashedWheelTimer/4.png)

我这里先说说大致的执行流程，之后再进行细致的源码分析。

默认地，时钟每 100ms 滴答一下（tick），往前走一格，共 512 格，走完一圈以后继续下一圈。把它想象成生活中的钟表就可以了。

内部使用一个长度为 512 的数组存储，数组元素（bucket）的数据结构是链表，链表每个元素代表一个任务，也就是我们前面介绍的 Timeout 的实例。

提交任务的线程，只要把任务往虚线上面的**任务队列**中存放即可返回。**工作线程是单线程**，一旦开启，不停地在时钟上绕圈圈。

仔细看下面的介绍：

- 工作线程到达每个时间**整点**的时候，开始工作。在 HashedWheelTimer 中，时间都是相对时间，工作线程的启动时间，定义为时间的 0 值。因为一次 tick 是 100ms(默认值)，所以 100ms、200ms、300ms... 就是我说的这些整点。

- 如上图，当时间到 200ms 的时候，发现任务队列有任务，**取出所有的任务**。

- 按照任务指定的执行时间，将其分配到相应的 bucket 中。如上图中，任务2 和任务6指定的时间为 100ms~200ms 这个区间，就被分配到第二个 bucket 中，形成链表，其他任务同理。

  > 这里还有轮次的概念，不过不用着急，比如任务 6 指定的时间可能是 150ms + (512*100ms)，它也会落在这个 bucket 中，但是它是下一个轮次才能被执行的。

- 任务分配到 bucket 完成后，执行该次 tick 的**真正**的任务，也就是落在**第二个** bucket 中的任务 2 和任务 6。

- 假设执行这两个任务共消耗了 50ms，到达 250ms 的时间点，那么工作线程会休眠 50ms，等待进入到 300ms 这个整点。

  > 如果这两个任务执行的时间超过 100ms 怎么办？这个问题就要看源码来解答了。

开始源码分析。我们从它的默认构造器开始，一步步到达最后一个最复杂的构造器：

```java
public HashedWheelTimer(
        ThreadFactory threadFactory,
        long tickDuration, TimeUnit unit, int ticksPerWheel, boolean leakDetection,
        long maxPendingTimeouts) {
  ......
}
```

简单说一下各个参数：

- threadFactory：定时任务都是后台任务，需要开启线程，我们通常会通过自定义 threadFactory 来命名线程，嫌麻烦就使用 `Executors.defaultThreadFactory()`。
- tickDuration 和 timeUnit 定义了一格的时间长度，默认的就是 100ms。
- ticksPerWheel 定义了一圈有多少格，默认的就是 512；
- leakDetection：用于追踪内存泄漏，本文不会介绍它，感兴趣的读者请自行去了解它。
- maxPendingTimeouts：最大允许等待的 Timeout 实例数，也就是我们可以设置不允许太多的任务等待。如果**未执行任务数**达到阈值，那么再次提交任务会抛出 RejectedExecutionException 异常。默认不限制。



### 初始化 HashedWheelTimer

```java
public HashedWheelTimer(
        ThreadFactory threadFactory,
        long tickDuration, TimeUnit unit, int ticksPerWheel, boolean leakDetection,
        long maxPendingTimeouts) {

    // ...... 参数检查

    // 初始化时间轮，这里做了向上"取整"，保持数组长度为 2 的 n 次方
    wheel = createWheel(ticksPerWheel);
    // 掩码，用来做取模
    mask = wheel.length - 1;

    // 100ms 转换为纳秒 100*10^6
    this.tickDuration = unit.toNanos(tickDuration);

    // 防止溢出
    if (this.tickDuration >= Long.MAX_VALUE / wheel.length) {
        throw new IllegalArgumentException(String.format(
                "tickDuration: %d (expected: 0 < tickDuration in nanos < %d",
                tickDuration, Long.MAX_VALUE / wheel.length));
    }
    // 创建工作线程，这里没有启动线程。后面会看到，在第一次提交任务的时候会启动线程
    workerThread = threadFactory.newThread(worker);

    leak = leakDetection || !workerThread.isDaemon() ? leakDetector.track(this) : null;

    // 赋值最大允许等待任务数
    this.maxPendingTimeouts = maxPendingTimeouts;

    // 如果超过 64 个 HashedWheelTimer 实例，它会打印错误日志提醒你
    // Netty 是真的到位，就怕你会用错这个工具，到处实例化它。而且它只会报错一次。
    if (INSTANCE_COUNTER.incrementAndGet() > INSTANCE_COUNT_LIMIT &&
        WARNED_TOO_MANY_INSTANCES.compareAndSet(false, true)) {
        reportTooManyInstances();
    }
}
```

上面，HashedWheelTimer 完成了初始化，初始化了时间轮数组 `HashedWheelBucket[]`，稍微看一下内部类 `HashedWheelBucket`，可以看到它是一个链表的结构。这个很好理解，因为每一格可能有多个任务。



### 提交第一个任务

```java
@Override
public Timeout newTimeout(TimerTask task, long delay, TimeUnit unit) {
    if (task == null) {
        throw new NullPointerException("task");
    }
    if (unit == null) {
        throw new NullPointerException("unit");
    }

    // 校验等待任务数是否达到阈值 maxPendingTimeouts
    long pendingTimeoutsCount = pendingTimeouts.incrementAndGet();
    if (maxPendingTimeouts > 0 && pendingTimeoutsCount > maxPendingTimeouts) {
        pendingTimeouts.decrementAndGet();
        throw new RejectedExecutionException("Number of pending timeouts ("
            + pendingTimeoutsCount + ") is greater than or equal to maximum allowed pending "
            + "timeouts (" + maxPendingTimeouts + ")");
    }

    // 如果工作线程没有启动，这里负责启动
    start();

    /** 下面的代码，构建 Timeout 实例，将其放到任务队列中。 **/

    // deadline 是一个相对时间，相对于 HashedWheelTimer 的启动时间
    long deadline = System.nanoTime() + unit.toNanos(delay) - startTime;

    // Guard against overflow.
    if (delay > 0 && deadline < 0) {
        deadline = Long.MAX_VALUE;
    }
    // timeout 实例，一个上层依赖 timer，一个下层依赖 task，另一个是任务到期时间
    HashedWheelTimeout timeout = new HashedWheelTimeout(this, task, deadline);
    // 放到 timeouts 队列中
    timeouts.add(timeout);
    return timeout;
}
```

提交任务的操作非常简单，实例化 Timeout，然后放到任务队列中。

我们可以看到，这里使用的优先级队列是一个 MPSC（Multiple Producer Single Consumer）的队列，刚好适用于这里的多生产线程，单消费线程的场景。而在 Dubbo 中，使用的队列是 LinkedBlockingQueue，它是一个以链表方式组织的线程安全的队列。

另外就是注意这里调用的 `start()` 方法，如果该任务是第一个提交的任务，它会负责工作线程的启动。



### 工作线程开始工作

其实只要看懂下面的几行代码，HashedWheelTimer 的源码就非常简单了。

```java
private final class Worker implements Runnable {
    private final Set<Timeout> unprocessedTimeouts = new HashSet<Timeout>();

    // tick 过的次数，前面说过，时针每 100ms tick 一次
    private long tick;

    @Override
    public void run() {
        // 在 HashedWheelTimer 中，用的都是相对时间，所以需要启动时间作为基准，并且要用 volatile 修饰
        startTime = System.nanoTime();
        if (startTime == 0) {
            // 这里不是很看得懂...请知道的读者不吝赐教
            startTime = 1;
        }

        // 第一个提交任务的线程正 await 呢，唤醒它
        startTimeInitialized.countDown();

        // 接下来这个 do-while 是真正执行任务的地方，非常重要
        do {
            // 往下滑，就在当前的代码块里面，倒数第二个方法
            // 比如之前介绍的图，那返回值 deadline 就是 200ms
            final long deadline = waitForNextTick();

            if (deadline > 0) {
                // 该次 tick，bucket 数组对应的 index
                int idx = (int) (tick & mask);

                // 处理一下已经取消的任务，可忽略。
                processCancelledTasks();

                // bucket
                HashedWheelBucket bucket = wheel[idx];

                // 将队列中所有的任务转移到相应的 buckets 中。细节往下看，这个代码块的下一个方法。
                transferTimeoutsToBuckets();

                // 执行进入到这个 bucket 中的任务
                bucket.expireTimeouts(deadline);

                tick++;
            }
        } while (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_STARTED);

        /* 到这里，说明这个 timer 要关闭了，做一些清理工作 */

        // 将所有 bucket 中没有执行的任务，添加到 unprocessedTimeouts 这个 HashSet 中，
        // 主要目的是用于 stop() 方法返回
        for (HashedWheelBucket bucket: wheel) {
            bucket.clearTimeouts(unprocessedTimeouts);
        }
        // 将任务队列中的任务也添加到 unprocessedTimeouts 中
        for (;;) {
            HashedWheelTimeout timeout = timeouts.poll();
            if (timeout == null) {
                break;
            }
            if (!timeout.isCancelled()) {
                unprocessedTimeouts.add(timeout);
            }
        }
        processCancelledTasks();
    }

    private void transferTimeoutsToBuckets() {
        // 这里一个 for 循环，还特地限制了 10 万次，就怕你写错代码，一直往里面扔任务，可能实际也没什么用吧
        for (int i = 0; i < 100000; i++) {
            HashedWheelTimeout timeout = timeouts.poll();
            if (timeout == null) {
                // 没有任务了
                break;
            }
            if (timeout.state() == HashedWheelTimeout.ST_CANCELLED) {
                // 该任务刚刚被取消了（在transfer之前其实已经做过一次了）
                continue;
            }

            // 下面就是将任务放到相应的 bucket 中，这里还计算了每个任务的 remainingRounds

            long calculated = timeout.deadline / tickDuration;
            timeout.remainingRounds = (calculated - tick) / wheel.length;

            final long ticks = Math.max(calculated, tick); // Ensure we don't schedule for past.
            int stopIndex = (int) (ticks & mask);

            HashedWheelBucket bucket = wheel[stopIndex];
            // 单个 bucket，是由 HashedWheelTimeout 实例组成的一个链表，
            // 大家可以看一下这里面的代码，由于是单线程操作，不存在并发，所以代码很简单
            bucket.addTimeout(timeout);
        }
    }

    private void processCancelledTasks() {
        //... 太简单，忽略
    }

    /**
     * 下面这个方法大家多看几遍，注意它的返回值
     * 前面说过，我们用的都是相对时间，所以：
     *   第一次进来的时候，工作线程会在 100ms 的时候返回，返回值是 100*10^6
     *   第二次进来的时候，工作线程会在 200ms 的时候返回，依次类推
     * 另外就是注意极端情况，比如第二次进来的时候，由于被前面的任务阻塞，导致进来的时候就已经是 250ms，
     *   那么，一进入这个方法就要立即返回，返回值是 250ms，而不是 200ms
     * 剩下的自己看一下代码
     */
    private long waitForNextTick() {
        long deadline = tickDuration * (tick + 1);

        // 嵌套在一个死循环里面
        for (;;) {
            final long currentTime = System.nanoTime() - startTime;
            long sleepTimeMs = (deadline - currentTime + 999999) / 1000000;

            if (sleepTimeMs <= 0) {
                if (currentTime == Long.MIN_VALUE) {
                    // 什么情况会进到这个分支？我也不清楚
                    return -Long.MAX_VALUE;
                } else {
                    // 这里是出口，所以返回值是当前时间(相对时间)
                    return currentTime;
                }
            }

            // Check if we run on windows, as if thats the case we will need
            // to round the sleepTime as workaround for a bug that only affect
            // the JVM if it runs on windows.
            //
            // See https://github.com/netty/netty/issues/356
            if (PlatformDependent.isWindows()) {
                sleepTimeMs = sleepTimeMs / 10 * 10;
            }

            try {
                Thread.sleep(sleepTimeMs);
            } catch (InterruptedException ignored) {
                // 如果 timer 已经 shutdown，那么返回 Long.MIN_VALUE
                if (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_SHUTDOWN) {
                    return Long.MIN_VALUE;
                }
            }
        }
    }

    public Set<Timeout> unprocessedTimeouts() {
        return Collections.unmodifiableSet(unprocessedTimeouts);
    }
}
```

接下来应该看看怎么执行 bucket 中的任务：

```java
/**
 * 这里会执行这个 bucket 中，轮次为 0 的任务，也就是到期的任务。
 * 这个方法的入参 deadline 其实没什么用，因为轮次为 0 的都是应该被执行的。
 */
public void expireTimeouts(long deadline) {
    HashedWheelTimeout timeout = head;

    // 处理链表上的所有 timeout 实例
    while (timeout != null) {
        HashedWheelTimeout next = timeout.next;
        if (timeout.remainingRounds <= 0) {
            next = remove(timeout);
            if (timeout.deadline <= deadline) {
                // 这行代码负责执行具体的任务
                timeout.expire();
            } else {
                // 这里的代码注释也说，不可能进入到这个分支
                // The timeout was placed into a wrong slot. This should never happen.
                throw new IllegalStateException(String.format(
                        "timeout.deadline (%d) > deadline (%d)", timeout.deadline, deadline));
            }
        } else if (timeout.isCancelled()) {
            next = remove(timeout);
        } else {
            // 轮次减 1
            timeout.remainingRounds --;
        }
        timeout = next;
    }
}
```

源码分析部分已经说完了，有一些简单的分支，这里就不花篇幅介绍了。

Worker 线程是一个 bucket 一个 bucket 顺次处理的，所以，即使有些任务执行时间超过了 100ms，“霸占”了之后好几个 bucket 的处理时间，也没关系，这些任务并不会被**漏掉**。但是有可能被延迟执行，毕竟工作线程是单线程。



### 说个有意思的点

在提交任务的 `newTimeout` 方法中，调用了启动线程的 `start()` 方法，它会保证线程真的启动以后并且赋值完了 `startTime` 以后，`start()` 方法再返回。因为在 newTimeout 方法的后半段中需要一个**正确的** startTime。

看下面两个代码片段。提交任务的线程：

```java
public void start() {
    ......

    // Wait until the startTime is initialized by the worker.
    while (startTime == 0) {
        try {
            startTimeInitialized.await();
        } catch (InterruptedException ignore) {
            // Ignore - it will be ready very soon.
        }
    }
}
```

工作线程 `Worker`：

```java
public void run() {
    // 初始化 startTime
    startTime = System.nanoTime();
    if (startTime == 0) {
        // We use 0 as an indicator for the uninitialized value here, so make sure it's not 0 when initialized.
        startTime = 1;
    }

    // 这里用了一个 CountDownLatch 实例来通信
    startTimeInitialized.countDown();
    ......
}
```

这里 startTime 是 volatile 修饰的属性，为了保证它的可见性。

可是大家有没有发现，其实有 startTimeInitialized 这个 CountDownLatch 实例，就能保证这里的并发先后问题。

这里简单讨论两个点：

1、第一个代码片段中的 while 换成 if 行不行？

> 不行。
>
> Object 中的 wait/notify 存在操作系统的假唤醒，所以一般都在 while 循环里，但是这里的 CountDownLatch 不会，所以看上去这里的 while 是没必要的。但是 CountDownLatch 没有提供 `awaitUninterruptibly()` 方法，所以这里其实在处理线程中断的情况。

2、这里 startTime 属性不用 volatile 修饰行不行？

> 个人认为是可以的。
>
> 因为 CountDownLatch 提供了语义：在 `countDown()` 之前的操作 happens-before `await()` 后的操作。
>
> 如果是我理解错误，欢迎大家指正。