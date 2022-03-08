# 异常导致ScheduledThreadPoolExecutor不执行原因分析 <!-- {docsify-ignore-all} -->


## 前言

&nbsp; &nbsp; 最近在调试一个监控应用指标的时候发现定时器在服务启动执行一次之后就不执行了，这里用的定时器是Java的调度线程池`ScheduledThreadPoolExecutor`，后来经过排查发现`ScheduledThreadPoolExecutor`线程池处理任务如果抛出异常，会导致线程池不调度；下面就通过一个例子简单分析下为什么异常会导致`ScheduledThreadPoolExecutor`不执行。


## `ScheduledThreadPoolExecutor`不调度分析

#### 示例程序

&nbsp; &nbsp; 在示例程序可以看到当计数器中的计数达到5的时候就会主动抛出一个异常，抛出异常后`ScheduledThreadPoolExecutor`就不调度了。

```java
public class ScheduledTask {

    private static final AtomicInteger count = new AtomicInteger(0);

    private static final ScheduledThreadPoolExecutor SCHEDULED_TASK = new ScheduledThreadPoolExecutor(
            1, new ThreadFactory() {
        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(Thread.currentThread().getThreadGroup(), r, "sc-task");
            t.setDaemon(true);
            return t;
        }
    });

    public static void main(String[] args) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(1);
        SCHEDULED_TASK.scheduleWithFixedDelay(() -> {
            System.out.println(111);
            if (count.get() == 5) {
                throw new IllegalArgumentException("my exception");
            }
            count.incrementAndGet();
        }, 0, 5, TimeUnit.SECONDS);
        latch.await();
    }
}
```

#### 源码分析

- **ScheduledThreadPoolExecutor#run**

&nbsp; &nbsp; run方法内部首先判断任务是不是周期性的任务，如果不是周期性任务通过`ScheduledFutureTask.super.run();`执行任务；如果状态是运行中或shutdown，取消任务执行；如果是周期性的任务，通过`ScheduledFutureTask.super.runAndReset()`执行任务并且重新设置状态，成功了就会执行`setNextRunTime`设置下次调度的时间，问题就是出现在`ScheduledFutureTask.super.runAndReset()`，这里执行任务出现了异常，导致结果为false，就不进行下次调度时间设置等

```java
        public void run() {
            boolean periodic = isPeriodic();
            if (!canRunInCurrentRunState(periodic))
                cancel(false);
            else if (!periodic)
                ScheduledFutureTask.super.run();
            else if (ScheduledFutureTask.super.runAndReset()) {
                setNextRunTime();
                reExecutePeriodic(outerTask);
            }
        }
```

- ***FutureTask#runAndReset**

&nbsp; &nbsp; 在线程任务执行过程中抛出异常，然后`catch`到了异常，最终导致这个方法返回false，然后`ScheduledThreadPoolExecutor#run`就不设置下次执行时间了，代码`c.call(); `抛出异常，跳过`ran = true;`代码，最终`runAndReset`返回false。

```java
    protected boolean runAndReset() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return false;
        boolean ran = false;
        int s = state;
        try {
            Callable<V> c = callable;
            if (c != null && s == NEW) {
                try {
                    c.call(); // don't set result
                    ran = true;
                } catch (Throwable ex) {
                    setException(ex);
                }
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
        return ran && s == NEW;
    }
```

## 总结

&nbsp; &nbsp; `Java`的`ScheduledThreadPoolExecutor`定时任务线程池所调度的任务中如果抛出了异常，并且异常没有捕获直接抛到框架中，会导致`ScheduledThreadPoolExecutor`定时任务不调度了，具体是因为当异常抛到`ScheduledThreadPoolExecutor`框架中时不进行下次调度时间的设置，从而导致`ScheduledThreadPoolExecutor`定时任务不调度。