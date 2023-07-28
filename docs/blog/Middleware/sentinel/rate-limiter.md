# RateLimiter 源码分析(Guava 和 Sentinel 实现) <!-- {docsify-ignore-all} -->



本文主要介绍关于流控的两部分内容。

第一部分介绍 Guava 中 RateLimiter 的源码，包括它的两种模式，目前网上大部分文章只分析简单的 SmoothBursty 模式，而没有分析带有预热的 SmoothWarmingUp。

第二部分介绍 Sentinel 中流控的实现，本文不要求读者了解 Sentinel，这部分内容和 Sentinel 耦合很低，所以读者不需要有阅读压力。

Sentinel 中流控设计是参考 Guava RateLimiter 的，所以阅读第二部分内容，需要有第一部分内容的背景。



## Guava RateLimiter

RateLimiter 基于漏桶算法，但它参考了令牌桶算法，这里不讨论流控算法，请自行查找资料。

本文使用 Guava 版本是 26.0-jre。



### RateLimiter 使用介绍

RateLimiter 的接口非常简单，它有两个静态方法用来实例化，实例化以后，我们只需要关心 `acquire` 就行了，甚至都没有 release 操作。

// RateLimiter 接口列表：

```java
// 实例化的两种方式：
public static RateLimiter create(double permitsPerSecond){}
public static RateLimiter create(double permitsPerSecond,long warmupPeriod,TimeUnit unit) {}

public double acquire() {}
public double acquire(int permits) {}

public boolean tryAcquire() {}
public boolean tryAcquire(int permits) {}
public boolean tryAcquire(long timeout, TimeUnit unit) {}
public boolean tryAcquire(int permits, long timeout, TimeUnit unit) {}

public final double getRate() {}
public final void setRate(double permitsPerSecond) {}
```

RateLimiter 的作用是用来限流的，我们知道 java 并发包中提供了 Semaphore，它也能够提供对资源使用进行控制，我们看一下下面的代码：

```java
// Semaphore
Semaphore semaphore = new Semaphore(10);
for (int i = 0; i < 100; i++) {
    executor.submit(new Runnable() {
        @Override
        public void run() {
            semaphore.acquireUninterruptibly(1);
            try {
                doSomething();
            } finally {
                semaphore.release();
            }
        }
    });
}
```

**Semaphore** 用来控制同时访问某个资源的并发数量，如上面的代码，我们设置 100 个线程工作，但是我们能做到最多只有 10 个线程能同时到 `doSomething()` 方法中。**它控制的是并发数量**。

而 RateLimiter 是用来控制访问资源的速率（rate）的，它强调的是控制速率。比如控制每秒只能有 100 个请求通过，比如允许每秒发送 1MB 的数据。

它的构造方法指定一个 `permitsPerSecond` 参数，代表每秒钟产生多少个 permits，这就是我们的速率。

RateLimiter 允许预占未来的令牌，比如，每秒产生 5 个 permits，我们可以单次请求 100 个 permits，这样，紧接着的下一个请求需要等待大概 20 秒才能获取到 permits。



### SmoothRateLimiter 介绍

RateLimiter 目前只有一个子类，那就是抽象类 SmoothRateLimiter，SmoothRateLimiter 有两个实现类，也就是我们这边要介绍的两种模式，我们先简单介绍下中间的抽象类 SmoothRateLimiter，然后后面分两个小节分别介绍它的两个实现类。

![1](https://assets.javadoop.com/imgs/20510079/rate-limiter/1.png)

RateLimiter 作为抽象类，只有两个属性：

```java
private final SleepingStopwatch stopwatch;

private volatile Object mutexDoNotUseDirectly;
```

stopwatch 非常重要，它用来“计时”，RateLimiter 把实例化的时间设置为 0 值，后续都是取相对时间，用微秒表示。

mutexDoNotUseDirectly 用来做锁，RateLimiter 依赖于 synchronized 来控制并发，所以我们之后可以看到，各个属性甚至都没有用 volatile 修饰。

然后我们来看 SmoothRateLimiter 的属性，分别代表什么意思。

```java
// 当前还有多少 permits 没有被使用，被存下来的 permits 数量
double storedPermits;

// 最大允许缓存的 permits 数量，也就是 storedPermits 能达到的最大值
double maxPermits;

// 每隔多少时间产生一个 permit，
// 比如我们构造方法中设置每秒 5 个，也就是每隔 200ms 一个，这里单位是微秒，也就是 200,000
double stableIntervalMicros;

// 下一次可以获取 permits 的时间，这个时间是相对 RateLimiter 的构造时间的，是一个相对时间，理解为时间戳吧
private long nextFreeTicketMicros = 0L; 
```

**其实，看到这几个属性，我们就可以大致猜一下它的内部实现了：**

`nextFreeTicketMicros` 是一个很关键的属性。我们每次获取 permits 的时候，先拿 storedPermits 的值，因为它是存货，如果够，storedPermits 减去相应的值就可以了，如果不够，那么还需要将 nextFreeTicketMicros 往前推，表示我预占了接下来多少时间的量了。那么下一个请求来的时候，如果还没到 nextFreeTicketMicros 这个时间点，需要 sleep 到这个点再返回，当然也要将这个值再往前推。

大家在这里可能会有疑惑，因为时间是一直往前走的，应该要一直往池中添加 permits，所以 storedPermits 的值需要不断往上添加，难道需要另外开启一个线程来添加 permits？其实不是的，只需要在关键的操作中同步一下，重新计算就好了。



### SmoothBursty 分析

我们先从比较简单的 SmoothBursty 出发，来分析 RateLimiter 的源码，之后我们再分析 SmoothWarmingUp。

> Bursty 是突发的意思，它说的**不是**下面这个意思：我们设置了 1k 每秒，而我们可以一次性获取 5k 的 permits，这个场景表达的不是突发，而是在说预先占有了接下来几秒产生的 permits。希望大家不要被误导了。
>
> 突发说的是，RateLimiter 会缓存一定数量的 permits 在池中，这样对于突发请求，能及时得到满足。想象一下我们的某个接口，很久没有请求过来，突然**同时**来了好几个请求，如果我们没有缓存一些 permits 的话，很多线程就需要等待了。
>
> SmoothBursty 默认缓存最多 1 秒钟的 permits，不可以修改。

RateLimiter 的静态构造方法：

```java
public static RateLimiter create(double permitsPerSecond) {
    return create(permitsPerSecond, SleepingStopwatch.createFromSystemTimer());
}
```

构造参数 permitsPerSecond 指定每秒钟可以产生多少个 permits。

```java
static RateLimiter create(double permitsPerSecond, SleepingStopwatch stopwatch) {
    RateLimiter rateLimiter = new SmoothBursty(stopwatch, 1.0 /* maxBurstSeconds */);
    rateLimiter.setRate(permitsPerSecond);
    return rateLimiter;
}
```

我们看到，这里实例化的是 SmoothBursty 的实例，它的构造方法很简单，而且它只有一个属性 `maxBurstSeconds`，这里就不贴代码了。

构造函数指定了 maxBurstSeconds 为 1.0，也就是说，最多会缓存 1 秒钟，也就是 (1.0 * permitsPerSecond) 这么多个 permits 到池中。

> 这个 1.0 秒，关系到 storedPermits 和 maxPermits：
>
> 0 <= storedPermits <= maxPermits = permitsPerSecond

我们继续往后看 setRate 方法：

```java
public final void setRate(double permitsPerSecond) {
  checkArgument(
      permitsPerSecond > 0.0 && !Double.isNaN(permitsPerSecond), "rate must be positive");
  synchronized (mutex()) {
    doSetRate(permitsPerSecond, stopwatch.readMicros());
  }
}
```

`setRate` 这个方法是一个 public 方法，它可以用来调整速率。我们这边继续跟的是初始化过程，但是大家提前知道这个方法是用来调整速率用的，对理解源码有很大的帮助。注意看，这里用了 synchronized 控制并发。

```java
@Override
final void doSetRate(double permitsPerSecond, long nowMicros) {
    // 同步
    resync(nowMicros);
    // 计算属性 stableIntervalMicros
    double stableIntervalMicros = SECONDS.toMicros(1L) / permitsPerSecond;
    this.stableIntervalMicros = stableIntervalMicros;
    doSetRate(permitsPerSecond, stableIntervalMicros);
}
```

resync 方法很简单，它用来调整 storedPermits 和 nextFreeTicketMicros。这就是我们说的，在关键的节点，需要先更新一下 storedPermits 到正确的值。

```java
void resync(long nowMicros) {
  // 如果 nextFreeTicket 已经过掉了，想象一下很长时间都没有再次调用 limiter.acquire() 的场景
  // 需要将 nextFreeTicket 设置为当前时间，重新计算 storedPermits
  if (nowMicros > nextFreeTicketMicros) {
    double newPermits = (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros();
    storedPermits = min(maxPermits, storedPermits + newPermits);
    nextFreeTicketMicros = nowMicros;
  }
}
```

> coolDownIntervalMicros() 这个方法大家先不用关注，可以看到，在 SmoothBursty 类中的实现是直接返回了 stableIntervalMicros 的值，也就是我们说的，每产生一个 permit 的时间长度。
>
> 当然了，细心的读者，可能会发现，此时的 stableIntervalMicros 其实没有设置，也就是说，上面发生了一次除以 0 值的操作，得到的 newPermits 其实是一个无穷大。而 maxPermits 此时还是 0 值，不过这里其实没有关系。

我们回到前面一个方法，resync 同步以后，会设置 stableIntervalMicros 为一个正确的值，然后进入下面的方法：

```java
@Override
void doSetRate(double permitsPerSecond, double stableIntervalMicros) {
  double oldMaxPermits = this.maxPermits;
  // 这里计算了，maxPermits 为 1 秒产生的 permits
  maxPermits = maxBurstSeconds * permitsPerSecond;
  if (oldMaxPermits == Double.POSITIVE_INFINITY) {
    // if we don't special-case this, we would get storedPermits == NaN, below
    storedPermits = maxPermits;
  } else {
    // 因为 storedPermits 的值域变化了，需要等比例缩放
    storedPermits =
        (oldMaxPermits == 0.0)
            ? 0.0 // initial state
            : storedPermits * maxPermits / oldMaxPermits;
  }
}
```

上面这个方法，我们要这么看，原来的 RateLimiter 是用某个 permitsPerSecond 值初始化的，现在我们要调整这个频率。对于 maxPermits 来说，是重新计算，而对于 storedPermits 来说，是做等比例的缩放。

到此，构造方法就完成了，我们得到了一个 RateLimiter 的实现类 SmoothBursty 的实例，可能上面的源码你还是会有一些疑惑，不过也没关系，继续往下看，可能你的很多疑惑就解开了。

接下来，我们来分析 acquire 方法：

```java
@CanIgnoreReturnValue
public double acquire() {
  return acquire(1);
}

@CanIgnoreReturnValue
public double acquire(int permits) {
  // 预约，如果当前不能直接获取到 permits，需要等待
  // 返回值代表需要 sleep 多久
  long microsToWait = reserve(permits);
  // sleep
  stopwatch.sleepMicrosUninterruptibly(microsToWait);
  // 返回 sleep 的时长
  return 1.0 * microsToWait / SECONDS.toMicros(1L);
}
```

我们来看 reserve 方法：

```java
final long reserve(int permits) {
  checkPermits(permits);
  synchronized (mutex()) {
    return reserveAndGetWaitLength(permits, stopwatch.readMicros());
  }
}

final long reserveAndGetWaitLength(int permits, long nowMicros) {
  // 返回 nextFreeTicketMicros
  long momentAvailable = reserveEarliestAvailable(permits, nowMicros);
  // 计算时长
  return max(momentAvailable - nowMicros, 0);
}
```

继续往里看：

```java
@Override
final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
  // 这里做一次同步，更新 storedPermits 和 nextFreeTicketMicros (如果需要)
  resync(nowMicros);
  // 返回值就是 nextFreeTicketMicros，注意刚刚已经做了 resync 了，此时它是最新的正确的值
  long returnValue = nextFreeTicketMicros;
  // storedPermits 中可以使用多少个 permits
  double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
  // storedPermits 中不够的部分
  double freshPermits = requiredPermits - storedPermitsToSpend;
  // 为了这个不够的部分，需要等待多久时间
  long waitMicros =
      storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend) // 这部分固定返回 0
          + (long) (freshPermits * stableIntervalMicros);
  // 将 nextFreeTicketMicros 往前推
  this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros);
  // storedPermits 减去被拿走的部分
  this.storedPermits -= storedPermitsToSpend;
  return returnValue;
}
```

我们可以看到，获取 permits 的时候，其实是获取了两部分，一部分来自于存量 **storedPermits**，存量不够的话，另一部分来自于预占未来的 **freshPermits**。

这里提一个关键点吧，我们看到，返回值是 nextFreeTicketMicros 的旧值，因为只要到这个时间点，就说明当次 acquire 可以成功返回了，而不管 storedPermits 够不够。如果 storedPermits 不够，会将 nextFreeTicketMicros 往前推一定的时间，预占了一定的量。

到这里，acquire 方法就分析完了，大家看到这里，逆着往前看就是了。应该说，SmoothBursty 的源码还是非常简单的。



### SmoothWarmingUp 分析

分析完了 SmoothBursty，我们再来分析 SmoothWarmingUp 会简单一些。我们说过，SmoothBursty 可以处理突发请求，因为它会缓存最多 1 秒的 permits，而待会我们会看到 SmoothWarmingUp 完全不同的设计。

SmoothWarmingUp 适用于资源需要预热的场景，比如我们的某个接口业务，需要使用到数据库连接，由于连接需要预热才能进入到最佳状态，如果我们的系统长时间处于低负载或零负载状态（当然，应用刚启动也是一样的），连接池中的连接慢慢释放掉了，此时我们认为连接池是冷的。

假设我们的业务在稳定状态下，正常可以提供最大 1000 QPS 的访问，但是如果连接池是冷的，我们就不能让系统达到 1000 个 QPS，要限制住突发流量，因为这会拖垮我们的系统，我们应该有个预热升温的过程。

对应到 SmoothWarmingUp 中，如果系统处于低负载状态，storedPermits 会一直增加，当请求来的时候，我们要从 storedPermits 中取 permits，最关键的点在于，从 storedPermits 中取 permits 的操作是比较耗时的，因为没有预热。

**回顾一下前面介绍的 SmoothBursty，它从 storedPermits 中获取 permits 是不需要等待时间的，因为它是存货，而这边洽洽相反，从 storedPermits 获取需要更多的时间，这是最大的不同，先理解这一点，能帮助你更好地理解源码。**

大家先有一些粗的概念，然后我们来看下面这个图：

![smooth-warm-up](https://assets.javadoop.com/imgs/20510079/rate-limiter/smooth-warm-up.png)

这个图不容易看懂，X 轴代表 storedPermits 的数量，Y 轴代表获取一个 permits 需要的时间。简单粗暴地说就是：存货越多，代表系统越冷，获取令牌所需时间越多。

> 假设指定 permitsPerSecond 为 10，那么 stableInterval 为 100ms，而 coldInterval 是 3 倍，也就是 300ms（coldFactor，3 倍是写死的，用户不能修改）。也就是说，当达到 maxPermits 时，此时处于系统最冷的时候，获取一个 permit 需要 300ms，而如果 storedPermits 小于 thresholdPermits 的时候，只需要 100ms。
>
> 利用 “获取**冷的** permits ” 需要等待更多时间，来限制突发请求通过，达到系统预热的目的。

想象有一条垂直线 x=k，它与 X 轴的交点 k 代表当前 storedPermits 的数量：

- 当系统在非常繁忙的时候，这条线停留在 x=0 处，此时 storedPermits 为 0
- 当 limiter 没有被使用的时候，这条线慢慢往右移动，直到 x=maxPermits 处；
- 如果 limiter 被重新使用，那么这条线又慢慢往左移动，直到 x=0 处；

当 【thresholdPermits <= **storedPermits** <= maxPermits】 状态时，我们认为 limiter 中的 permits 是冷的，此时获取一个 permit 需要较多的时间，因为需要预热，有一个关键的分界点是 **thresholdPermits**。

预热时间是我们在构造的时候指定的，图中梯形的面积就是预热时间，因为预热完成后，我们能进入到一个稳定的速率中（stableInterval），下面我们 **根据构造参数计算出 thresholdPermits 和 maxPermits 的值**。

有一个关键点，从 thresholdPermits 到 0 的时间，是从 maxPermits 到 thresholdPermits 时间的一半，也就是梯形的面积是长方形面积的 2 倍，梯形的面积是 warmupPeriod。

![7](https://assets.javadoop.com/imgs/20510079/rate-limiter/7.png)

> 之所以长方形的面积是 warmupPeriod/2，也就是梯形面积的一半，是因为 coldFactor 是硬编码的 **3**。具体的可以参考一下文章下面评论区的讨论。

梯形面积为 warmupPeriod，而长方形面积为 stableInterval * thresholdPermits，即：

```java
warmupPeriod = 2 * stableInterval * thresholdPermits
```

由此，我们得出 **thresholdPermits** 的值：

```java
thresholdPermits = 0.5 * warmupPeriod / stableInterval
```

然后我们根据梯形面积的计算公式：

```java
warmupPeriod = 0.5 * (stableInterval + coldInterval) * (maxPermits - thresholdPermits)
```

得出 **maxPermits** 为：

```java
maxPermits = thresholdPermits + 2.0 * warmupPeriod / (stableInterval + coldInterval)
```

这样，我们就得到了 thresholdPermits 和 maxPermits 的值。

接下来，我们来看一下冷却时间间隔，它指的是 storedPermits 中每个 permit 的增长速度，也就是我们前面说的 **x=k 这条垂直线往右的移动速度**，为了达到从 0 到 maxPermits 花费 warmupPeriodMicros 的时间，我们将其定义为：

```java
@Override
double coolDownIntervalMicros() {
    return warmupPeriodMicros / maxPermits;
}
```

贴一下代码，大家就知道了，在 resync 中用到的这个：

```java
void resync(long nowMicros) {
  if (nowMicros > nextFreeTicketMicros) {
    // coolDownIntervalMicros 在这里使用
    double newPermits = (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros();
    storedPermits = min(maxPermits, storedPermits + newPermits);
    nextFreeTicketMicros = nowMicros;
  }
}
```

基于上面的分析，我们来看 SmoothWarmingUp 的其他源码。

首先，我们来看它的 doSetRate 方法，有了前面的介绍，这个方法的源码非常简单：

```java
@Override
void doSetRate(double permitsPerSecond, double stableIntervalMicros) {
    double oldMaxPermits = maxPermits;
    // coldFactor 是固定的 3
    double coldIntervalMicros = stableIntervalMicros * coldFactor;
    // 这个公式我们上面已经说了
    thresholdPermits = 0.5 * warmupPeriodMicros / stableIntervalMicros;
    // 这个公式我们上面也已经说了
    maxPermits =
        thresholdPermits + 2.0 * warmupPeriodMicros / (stableIntervalMicros + coldIntervalMicros);
    // 计算那条斜线的斜率。数学知识，对边 / 临边
    slope = (coldIntervalMicros - stableIntervalMicros) / (maxPermits - thresholdPermits);
    if (oldMaxPermits == Double.POSITIVE_INFINITY) {
        // if we don't special-case this, we would get storedPermits == NaN, below
        storedPermits = 0.0;
    } else {
        storedPermits =
            (oldMaxPermits == 0.0)
                ? maxPermits // initial state is cold
                : storedPermits * maxPermits / oldMaxPermits;
    }
}
```

setRate 方法非常简单，接下来，我们要分析的是 storedPermitsToWaitTime 方法，我们回顾一下下面的代码：

![3](https://assets.javadoop.com/imgs/20510079/rate-limiter/3.png)

这段代码是 acquire 方法的核心，waitMicros 由两部分组成，一部分是从 storedPermits 中获取花费的时间，一部分是等待 freshPermits 产生花费的时间。在 SmoothBursty 的实现中，从 storedPermits 中获取 permits 直接返回 0，不需要等待。

而在 SmoothWarmingUp 的实现中，由于需要预热，所以从 storedPermits 中取 permits 需要花费一定的时间，其实就是要计算下图中，阴影部分的面积。

![4](https://assets.javadoop.com/imgs/20510079/rate-limiter/6.png)

```java
@Override
long storedPermitsToWaitTime(double storedPermits, double permitsToTake) {
  double availablePermitsAboveThreshold = storedPermits - thresholdPermits;
  long micros = 0;
  // 如果右边梯形部分有 permits，那么先从右边部分获取permits，计算梯形部分的阴影部分的面积
  if (availablePermitsAboveThreshold > 0.0) {
    // 从右边部分获取的 permits 数量
    double permitsAboveThresholdToTake = min(availablePermitsAboveThreshold, permitsToTake);
    // 梯形面积公式：(上底+下底)*高/2
    double length =
        permitsToTime(availablePermitsAboveThreshold)
            + permitsToTime(availablePermitsAboveThreshold - permitsAboveThresholdToTake);
    micros = (long) (permitsAboveThresholdToTake * length / 2.0);
    permitsToTake -= permitsAboveThresholdToTake;
  }
  // 加上 长方形部分的阴影面积
  micros += (long) (stableIntervalMicros * permitsToTake);
  return micros;
}

// 对于给定的 x 值，计算 y 值
private double permitsToTime(double permits) {
  return stableIntervalMicros + permits * slope;
}
```

到这里，SmoothWarmingUp 也已经说完了。

如果大家对于 Guava RateLimiter 还有什么疑惑，欢迎在留言区留言，对于 Sentinel 中的流控不感兴趣的读者，看到这里就可以结束了。



## Sentinel 中的流控

[Sentinel](https://github.com/alibaba/Sentinel) 是阿里开源的流控、熔断工具，这里不做过多的介绍，感兴趣的读者请自行了解。

在 Sentinel 的流控中，我们可以配置流控规则，主要是控制 QPS 和并发线程数，这里我们不讨论控制线程数，控制线程数的代码不在我们这里的讨论范围内，下面的介绍都是指控制 QPS。



### RateLimiterController

RateLimiterController 非常简单，它通过使用 latestPassedTime 属性来记录最后一次通过的时间，然后根据规则中 QPS 的限制，计算当前请求是否可以通过。它在 Sentinel 中的流控效果定义为 **“排队等待”**。

举个非常简单的例子：设置 QPS 为 10，那么每 100 毫秒允许通过一个，通过计算当前时间是否已经过了上一个请求的通过时间 **latestPassedTime** 之后的 100 毫秒，来判断是否可以通过。假设才过了 50ms，那么需要当前线程再 sleep 50ms，然后才可以通过。如果同时有另一个请求呢？那需要 sleep 150ms 才行。

![sentinel-3](https://assets.javadoop.com/imgs/20510079/rate-limiter/sentinel-3.png)

```java
public class RateLimiterController implements TrafficShapingController {

    // 排队最大时长，默认 500ms
    private final int maxQueueingTimeMs;
    // QPS 设置的值
    private final double count;
        // 上一次请求通过的时间
    private final AtomicLong latestPassedTime = new AtomicLong(-1);

    public RateLimiterController(int timeOut, double count) {
        this.maxQueueingTimeMs = timeOut;
        this.count = count;
    }

    @Override
    public boolean canPass(Node node, int acquireCount) {
        return canPass(node, acquireCount, false);
    }

    // 通常 acquireCount 为 1，这里不用关心参数 prioritized
    @Override
    public boolean canPass(Node node, int acquireCount, boolean prioritized) {
        // Pass when acquire count is less or equal than 0.
        if (acquireCount <= 0) {
            return true;
        }
        // 
        if (count <= 0) {
            return false;
        }

        long currentTime = TimeUtil.currentTimeMillis();
        // 计算每 2 个请求之间的间隔，比如 QPS 限制为 10，那么间隔就是 100ms
        long costTime = Math.round(1.0 * (acquireCount) / count * 1000);

        // Expected pass time of this request.
        long expectedTime = costTime + latestPassedTime.get();

        // 可以通过，设置 latestPassedTime 然后就返回 true 了
        if (expectedTime <= currentTime) {
            // Contention may exist here, but it's okay.
            latestPassedTime.set(currentTime);
            return true;
        } else {
            // 不可以通过，需要等待
            long waitTime = costTime + latestPassedTime.get() - TimeUtil.currentTimeMillis();
            // 等待时长大于最大值，返回 false
            if (waitTime > maxQueueingTimeMs) {
                return false;
            } else {
                // 将 latestPassedTime 往前推
                long oldTime = latestPassedTime.addAndGet(costTime);
                try {
                    // 需要 sleep 的时间
                    waitTime = oldTime - TimeUtil.currentTimeMillis();
                    if (waitTime > maxQueueingTimeMs) {
                        latestPassedTime.addAndGet(-costTime);
                        return false;
                    }
                    // in race condition waitTime may <= 0
                    if (waitTime > 0) {
                        Thread.sleep(waitTime);
                    }
                    return true;
                } catch (InterruptedException e) {
                }
            }
        }
        return false;
    }

}
```

这个源码非常简单，策略也非常简单，这里就不做过多讨论了。



### WarmUpController

WarmUpController 用来防止突发流量迅速上升，导致系统负载严重过高，本来系统在稳定状态下能处理的，但是由于许多资源没有预热，导致这个时候处理不了了。比如，数据库需要建立连接、需要连接到远程服务等，这就是为什么我们需要预热。

啰嗦一句，这里不仅仅指系统刚刚启动需要预热，对于长时间处于低负载的系统，突发流量也需要重新预热。

Guava 的 SmoothWarmingUp 是用来控制获取令牌的速率的，和这里的控制 QPS 还是有一点区别，但是中心思想是一样的。我们在看完源码以后再讨论它们的区别。

![sentinel-2](https://assets.javadoop.com/imgs/20510079/rate-limiter/sentinel-2.png)

为了帮助大家理解源码，我们这边先设定一个场景：QPS 设置为 100，预热时间设置为 10 秒。代码中使用 “【】” 代表根据这个场景计算出来的值。

接下来，大家请仔细看下面的这块源码：

```java
public class WarmUpController implements TrafficShapingController {

    // 阈值
    protected double count;
    // 3
    private int coldFactor;
    // 转折点的令牌数，和 Guava 的 thresholdPermits 一个意思
    // [500]
    protected int warningToken = 0;
    // 最大的令牌数，和 Guava 的 maxPermits 一个意思
    // [1000]
    private int maxToken;
    // 斜线斜率
    // [1/25000]
    protected double slope;

    // 累积的令牌数，和 Guava 的 storedPermits 一个意思
    protected AtomicLong storedTokens = new AtomicLong(0);
    // 最后更新令牌的时间
    protected AtomicLong lastFilledTime = new AtomicLong(0);

    public WarmUpController(double count, int warmUpPeriodInSec, int coldFactor) {
        construct(count, warmUpPeriodInSec, coldFactor);
    }

    public WarmUpController(double count, int warmUpPeriodInSec) {
        construct(count, warmUpPeriodInSec, 3);
    }

    // 下面的构造方法，和 Guava 中是差不多的，只不过 thresholdPermits 和 maxPermits 都换了个名字
    private void construct(double count, int warmUpPeriodInSec, int coldFactor) {

        if (coldFactor <= 1) {
            throw new IllegalArgumentException("Cold factor should be larger than 1");
        }

        this.count = count;

        this.coldFactor = coldFactor;

        // warningToken 和 thresholdPermits 是一样的意思，计算结果其实是一样的
        // thresholdPermits = 0.5 * warmupPeriod / stableInterval.
        // 【warningToken = (10*100)/(3-1) = 500】
        warningToken = (int)(warmUpPeriodInSec * count) / (coldFactor - 1);

        // maxToken 和 maxPermits 是一样的意思，计算结果其实是一样的
        // maxPermits = thresholdPermits + 2*warmupPeriod/(stableInterval+coldInterval)
        // 【maxToken = 500 + (2*10*100)/(1.0+3) = 1000】
        maxToken = warningToken + (int)(2 * warmUpPeriodInSec * count / (1.0 + coldFactor));

        // 斜率计算
        // slope
        // slope = (coldIntervalMicros-stableIntervalMicros)/(maxPermits-thresholdPermits);
        // 【slope = (3-1.0) / 100 / (1000-500) = 1/25000】
        slope = (coldFactor - 1.0) / count / (maxToken - warningToken);

    }

    @Override
    public boolean canPass(Node node, int acquireCount) {
        return canPass(node, acquireCount, false);
    }

    @Override
    public boolean canPass(Node node, int acquireCount, boolean prioritized) {

        // Sentinel 的 QPS 统计使用的是滑动窗口

        // 当前时间窗口的 QPS 
        long passQps = (long) node.passQps();

        // 这里是上一个时间窗口的 QPS，这里的一个窗口跨度是1秒钟
        long previousQps = (long) node.previousPassQps();

        // 同步。设置 storedTokens 和 lastFilledTime 到正确的值
        syncToken(previousQps);

        long restToken = storedTokens.get();
        // 令牌数超过 warningToken，进入梯形区域
        if (restToken >= warningToken) {

            // 这里简单说一句，因为当前的令牌数超过了 warningToken 这个阈值，系统处于需要预热的阶段
            // 通过计算当前获取一个令牌所需时间，计算其倒数即是当前系统的最大 QPS 容量

            long aboveToken = restToken - warningToken;

            // 这里计算警戒 QPS 值，就是当前状态下能达到的最高 QPS。
            // (aboveToken * slope + 1.0 / count) 其实就是在当前状态下获取一个令牌所需要的时间
            double warningQps = Math.nextUp(1.0 / (aboveToken * slope + 1.0 / count));
            // 如果不会超过，那么通过，否则不通过
            if (passQps + acquireCount <= warningQps) {
                return true;
            }
        } else {
            // count 是最高能达到的 QPS
            if (passQps + acquireCount <= count) {
                return true;
            }
        }

        return false;
    }

    protected void syncToken(long passQps) {
        // 下面几行代码，说明在第一次进入新的 1 秒钟的时候，做同步
        // 题外话：Sentinel 默认地，1 秒钟分为 2 个时间窗口，分别 500ms
        long currentTime = TimeUtil.currentTimeMillis();
        currentTime = currentTime - currentTime % 1000;
        long oldLastFillTime = lastFilledTime.get();
        if (currentTime <= oldLastFillTime) {
            return;
        }

        // 令牌数量的旧值
        long oldValue = storedTokens.get();
        // 计算新的令牌数量，往下看
        long newValue = coolDownTokens(currentTime, passQps);

        if (storedTokens.compareAndSet(oldValue, newValue)) {
            // 令牌数量上，减去上一分钟的 QPS，然后设置新值
            long currentValue = storedTokens.addAndGet(0 - passQps);
            if (currentValue < 0) {
                storedTokens.set(0L);
            }
            lastFilledTime.set(currentTime);
        }

    }

    // 更新令牌数
    private long coolDownTokens(long currentTime, long passQps) {
        long oldValue = storedTokens.get();
        long newValue = oldValue;

        // 当前令牌数小于 warningToken，添加令牌
        if (oldValue < warningToken) {
            newValue = (long)(oldValue + (currentTime - lastFilledTime.get()) * count / 1000);
        } else if (oldValue > warningToken) {
            // 当前令牌数量处于梯形阶段，
            // 如果当前通过的 QPS 大于 count/coldFactor，说明系统消耗令牌的速度，大于冷却速度
            //    那么不需要添加令牌，否则需要添加令牌
            if (passQps < (int)count / coldFactor) {
                newValue = (long)(oldValue + (currentTime - lastFilledTime.get()) * count / 1000);
            }
        }
        return Math.min(newValue, maxToken);
    }

}
```

> coolDownTokens 这个方法用来计算新的 token 数量，其实我也没有完全理解作者的设计：
>
> 第一、对于令牌的增加，在 Guava 中，使用 warmupPeriodMicros / maxPermits 作为增长率，因为它实现的是 storedPermits 从 0 到 maxPermits 花费的时间为 warmupPeriod。而这里是以设置的 QPS 作为增长率，为什么？
>
> 第二、else if 分支中的决定我没有理解，为什么用 passQps 和 count / coldFactor 进行对比来决定是否继续添加令牌？
>
> 我自己的理解是，count/coldFactor 就是指冷却速度，那么就是说得通的。欢迎大家一起探讨。

最后，我们再简单说说 Guava 的 SmoothWarmingUp 和 Sentinel 的 WarmupController 的区别。

Guava 在于控制获取令牌的速率，它关心的是，获取  permits 需要多少时间，包括从 storedPermits 中获取，以及获取 freshPermits，以此推进 nextFreeTicketMicros 到未来的某个时间点。

而 Sentinel 在于控制 QPS，它用令牌数来标识当前系统处于什么状态，根据时间推进一直增加令牌，根据通过的 QPS 一直减少令牌。如果 QPS 持续下降，根据推演，可以发现 storedTokens 越来越多，然后越过 warningTokens 这个阈值，之后只有当 QPS 下降到 count/3 以后，令牌才会继续往上增长，一直到 maxTokens。

> storedTokens 是以 “count 每秒”的增长率增长的，减少是以 前一分钟的 QPS 来减少的。其实这里我也有个疑问，为什么增加令牌的时候考虑了时间，而减少的时候却不考虑时间因素，提了 issue，不过还没有得到回答。



### WarmUpRateLimiterController

注意，这个类继承自刚刚介绍的 WarmUpController。它的代码其实就是前面介绍的 RateLimiterController 加上 WarmUpController。

```java
public class WarmUpRateLimiterController extends WarmUpController {

    private final int timeoutInMs;
    private final AtomicLong latestPassedTime = new AtomicLong(-1);

    public WarmUpRateLimiterController(double count, int warmUpPeriodSec, int timeOutMs, int coldFactor) {
        super(count, warmUpPeriodSec, coldFactor);
        this.timeoutInMs = timeOutMs;
    }

    @Override
    public boolean canPass(Node node, int acquireCount) {
        return canPass(node, acquireCount, false);
    }

    @Override
    public boolean canPass(Node node, int acquireCount, boolean prioritized) {
        long previousQps = (long) node.previousPassQps();
        syncToken(previousQps);

        long currentTime = TimeUtil.currentTimeMillis();

        long restToken = storedTokens.get();
        long costTime = 0;
        long expectedTime = 0;

        // 和 RateLimiterController 比较，区别主要就是这块代码，计算 costTime 上有区别

        if (restToken >= warningToken) {
            long aboveToken = restToken - warningToken;

            // current interval = restToken*slope+1/count
            double warmingQps = Math.nextUp(1.0 / (aboveToken * slope + 1.0 / count));
            costTime = Math.round(1.0 * (acquireCount) / warmingQps * 1000);
        } else {
            costTime = Math.round(1.0 * (acquireCount) / count * 1000);
        }
        expectedTime = costTime + latestPassedTime.get();

        if (expectedTime <= currentTime) {
            latestPassedTime.set(currentTime);
            return true;
        } else {
            long waitTime = costTime + latestPassedTime.get() - currentTime;
            if (waitTime > timeoutInMs) {
                return false;
            } else {
                long oldTime = latestPassedTime.addAndGet(costTime);
                try {
                    waitTime = oldTime - TimeUtil.currentTimeMillis();
                    if (waitTime > timeoutInMs) {
                        latestPassedTime.addAndGet(-costTime);
                        return false;
                    }
                    if (waitTime > 0) {
                        Thread.sleep(waitTime);
                    }
                    return true;
                } catch (InterruptedException e) {
                }
            }
        }
        return false;
    }
}
```

这个代码很简单，就是 RateLimiterController  中的代码，然后加入了预热的内容。

在 RateLimiterController 中，单个请求的 costTime 是固定的，就是 1/count，比如设置 100 qps，那么 costTime 就是 10ms。

但是这边，加入了 WarmUp 的内容，就是说，通过令牌数量，来判断当前系统的 QPS 应该是多少，如果当前令牌数超过 warningTokens，那么系统的最大 QPS 容量已经低于我们预设的 QPS，相应的，costTime 就会延长。