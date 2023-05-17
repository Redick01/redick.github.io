# 使用TransmittableThreadLocal实现异步场景日志链路追踪 <!-- {docsify-ignore-all} -->

- 背景
- 解决方案

## 背景

&nbsp; &nbsp; 在生产环境排查问题往往都是通过日志，但对于巨大的日志量，如何针对某一个操作进行一整个日志链路的追踪就显得尤为重要，在Java语言第三方的日志工具都提了日志链路追踪的方案，比如logback的MDC，MDC的使用也很简单，就是在业务的开始put一个key-value，这个key-value就能贯穿整个线程的执行流程，使用代码如下：
```java
MDC.put("traceId", UUID.randomUUID().toString());
```

&nbsp; &nbsp; MDC虽然提供了一个现成的整个执行流程的日志追踪的方案，但是也只是一个线程，假如一个线程中又启动了另一个线程呢，这时MDC就无法完成完整的链路追踪工作了，因为MDC是基于ThreadLocal实现的，所以当一个线程中启动另一个线程的时候两个线程的`TraceId`就隔离开了，也就无法做到日志链路追踪。

## 解决方案

### 线程间参数传递技术选型

- **InheritableThreadLocal**

&nbsp; &nbsp; `InheritableThreadLocal`是JDK实现的一种线程传递解决方案，由当前线程创建的线程，将会继承当前线程里`ThreadLocal`保存的值，但由于`InheritableThreadLocal`是在创建线程是解决`ThreadLocal`的传值问题，但是线程不可能一直创建，在工程代码中往往都是使用线程池，但是，递交异步任务使相应的`ThreadLocal`的值就无法传递过去了。

- **TransmittableThreadLocal真正的解决方案**

&nbsp; &nbsp; `TransmittableThreadLocal`是阿里巴巴开源的，用于解决**在使用线程池等会缓存线程的组件情况下传递ThreadLocal**问题的`InheritableThreadLocal`扩展，具体的实现以后有机会深入研究，该解决方案常用于以下几个场景：

  >分布式跟踪系统
  >应用容器或上层框架跨应用代码给下层SDK传递信息
  >日志收集记录系统上下文

### 重写MDCAdapter

&nbsp; &nbsp; 由于MDC是基于ThreadLocal实现的，所以我们现在需要做的就是重写MDCAdapter，使系统再使用MDC时实际上是使用的我们自己实现的MDCAdapter，自定义的MDCAdapter时要注意包名应该与logback的MDCAdapter一致，因为我们要在程序启动的时候替换MDC中的MDCAdapter，MDC的MDCAdapter是包级私有，所以自定义MDCAdapter的包名一定要哥logback的MDCAdapter一致，自定义MDCAdapter代码如下：

```java
package org.slf4j;


import com.alibaba.ttl.TransmittableThreadLocal;
import org.slf4j.spi.MDCAdapter;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;
import java.util.Set;

/**
 * 重写logback的LogbackMDCAdapter，注意MDC是MDCAdapter是包级私有，所以重写的报名应该是org.slf4j
 * 用TransmittableThreadLocal替换ThreadLocal，解决多线程，异步情况x-glabal-sessionId无法传递问题
 * @author liupenghui
 * @date 2021/6/30 2:38 下午
 */
public class TtlMDCAdapter implements MDCAdapter {

    private final ThreadLocal<Map<String, String>> copyOnInheritThreadLocal = new TransmittableThreadLocal<>();

    private static final int WRITE_OPERATION = 1;
    private static final int MAP_COPY_OPERATION = 2;

    static TtlMDCAdapter mtcMDCAdapter;

    static {
        mtcMDCAdapter = new TtlMDCAdapter();
        // 替换MDC的MDCAdapter
        MDC.mdcAdapter = mtcMDCAdapter;
    }

    public static MDCAdapter getInstance() {
        return mtcMDCAdapter;
    }

    final ThreadLocal<Integer> lastOperation = new ThreadLocal<Integer>();

    private Integer getAndSetLastOperation(int op) {
        Integer lastOp = lastOperation.get();
        lastOperation.set(op);
        return lastOp;
    }

    private boolean wasLastOpReadOrNull(Integer lastOp) {
        return lastOp == null || lastOp.intValue() == MAP_COPY_OPERATION;
    }

    private Map<String, String> duplicateAndInsertNewMap(Map<String, String> oldMap) {
        Map<String, String> newMap = Collections.synchronizedMap(new HashMap<String, String>());
        if (oldMap != null) {
            // we don't want the parent thread modifying oldMap while we are
            // iterating over it
            synchronized (oldMap) {
                newMap.putAll(oldMap);
            }
        }

        copyOnInheritThreadLocal.set(newMap);
        return newMap;
    }

    /**
     * Put a context value (the <code>val</code> parameter) as identified with the
     * <code>key</code> parameter into the current thread's context map. Note that
     * contrary to log4j, the <code>val</code> parameter can be null.
     * <p/>
     * <p/>
     * If the current thread does not have a context map it is created as a side
     * effect of this call.
     *
     * @throws IllegalArgumentException in case the "key" parameter is null
     */
    @Override
    public void put(String key, String val) throws IllegalArgumentException {
        if (key == null) {
            throw new IllegalArgumentException("key cannot be null");
        }

        Map<String, String> oldMap = copyOnInheritThreadLocal.get();
        Integer lastOp = getAndSetLastOperation(WRITE_OPERATION);

        if (wasLastOpReadOrNull(lastOp) || oldMap == null) {
            Map<String, String> newMap = duplicateAndInsertNewMap(oldMap);
            newMap.put(key, val);
        } else {
            oldMap.put(key, val);
        }
    }

    /**
     * Remove the the context identified by the <code>key</code> parameter.
     * <p/>
     */
    @Override
    public void remove(String key) {
        if (key == null) {
            return;
        }
        Map<String, String> oldMap = copyOnInheritThreadLocal.get();
        if (oldMap == null)
            return;

        Integer lastOp = getAndSetLastOperation(WRITE_OPERATION);

        if (wasLastOpReadOrNull(lastOp)) {
            Map<String, String> newMap = duplicateAndInsertNewMap(oldMap);
            newMap.remove(key);
        } else {
            oldMap.remove(key);
        }
    }

    /**
     * Clear all entries in the MDC.
     */
    @Override
    public void clear() {
        lastOperation.set(WRITE_OPERATION);
        copyOnInheritThreadLocal.remove();
    }

    /**
     * Get the context identified by the <code>key</code> parameter.
     * <p/>
     */
    @Override
    public String get(String key) {
        final Map<String, String> map = copyOnInheritThreadLocal.get();
        if ((map != null) && (key != null)) {
            return map.get(key);
        } else {
            return null;
        }
    }

    /**
     * Get the current thread's MDC as a map. This method is intended to be used
     * internally.
     */
    public Map<String, String> getPropertyMap() {
        lastOperation.set(MAP_COPY_OPERATION);
        return copyOnInheritThreadLocal.get();
    }

    /**
     * Returns the keys in the MDC as a {@link Set}. The returned value can be
     * null.
     */
    public Set<String> getKeys() {
        Map<String, String> map = getPropertyMap();

        if (map != null) {
            return map.keySet();
        } else {
            return null;
        }
    }

    /**
     * Return a copy of the current thread's context map. Returned value may be
     * null.
     */
    @Override
    public Map<String, String> getCopyOfContextMap() {
        Map<String, String> hashMap = copyOnInheritThreadLocal.get();
        if (hashMap == null) {
            return null;
        } else {
            return new HashMap<String, String>(hashMap);
        }
    }

    @Override
    public void setContextMap(Map<String, String> contextMap) {
        lastOperation.set(WRITE_OPERATION);

        Map<String, String> newMap = Collections.synchronizedMap(new HashMap<String, String>());
        newMap.putAll(contextMap);

        // the newMap replaces the old one for serialisation's sake
        copyOnInheritThreadLocal.set(newMap);
    }
}
```

### TtlMDCAdapter生效

- **Spring程序**

程序启动时设置MDCAdapter，代码如下：

```java
TtlMDCAdapter.getInstance();
```

- **SpringBoot程序**

实现`ApplicationContextInitializer`接口，重写`initialize`方法，代码如下：
```java
public class TtlMDCAdapterInitializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {

    @Override
    public void initialize(ConfigurableApplicationContext configurableApplicationContext) {
        // 加载自定义的MDCAdapter
        TtlMDCAdapter.getInstance();
    }
}
```

**TtlMDCAdapterInitializer**生效的三种方式：

> 在META-INF目录下创建spring.factories，内容如下：

```factories
# SpringBoot程序添加自定义ApplicationContextInitializer，用以支持异步程序日志链路追踪
org.springframework.context.ApplicationContextInitializer=com.ruubypay.log.TtlMDCAdapterInitializer
```

> application.properties添加配置方式：

```properties
context.initializer.classes=com.ruubypay.log.TtlMDCAdapterInitializer
```

> 程序代码addInitializers

```java
@SpringBootApplication
public class Application {
 
	public static void main(String[] args) {
		SpringApplication springApplication = new SpringApplication(Application.class);
        springApplication.addInitializers(new Demo01ApplicationContextInitializer());
        springApplication.run(args);
	}
}
```

### 增强线程池JUC的ThreadPoolExecutor和Spring的ThreadPoolTaskExecutor

&nbsp; &nbsp; 使用线程池的异步任务场景必须要对线程池进行一定的修饰，如果不对线程池进行增强，是无法做到traceId的正确性的，会造成traceId混乱，复用等情况的发生，导致这个问题的原因这里不做暂不做解释，下面代码分别是对JUC的ThreadPoolExecutor和Spring的ThreadPoolTaskExecutor的修饰代码，也就是说我们在业务代码使用线程池的时候就不要直接使用原生的线程池了，直接使用我们经过Ttl修饰过的线程池。

**修饰ThreadPoolExecutor**
```java
public class TtlThreadPoolExecutor extends ThreadPoolExecutor {

    public TtlThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue);
    }

    public TtlThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory);
    }

    public TtlThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, RejectedExecutionHandler handler) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, handler);
    }

    public TtlThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory, RejectedExecutionHandler handler) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, handler);
    }

    @Override
    public void execute(Runnable command) {
        Runnable runnable = TtlRunnable.get(command);
        super.execute(runnable);
    }

    @Override
    public <T> Future<T> submit(Runnable task, T result) {
        Runnable runnable = TtlRunnable.get(task);
        return super.submit(runnable, result);
    }

    @Override
    public Future<?> submit(Runnable task) {
        Runnable runnable = TtlRunnable.get(task);
        return super.submit(runnable);
    }

    @Override
    public <T> Future<T> submit(Callable<T> task) {
        Callable command = TtlCallable.get(task);
        return super.submit(command);
    }

    @Override
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        Runnable command = TtlRunnable.get(runnable);
        return super.newTaskFor(command, value);
    }

    @Override
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        Callable command = TtlCallable.get(callable);
        return super.newTaskFor(command);
    }

    @Override
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks) throws InterruptedException, ExecutionException {
        Collection<? extends Callable<T>> callables = TtlCallable.gets(tasks);
        return super.invokeAny(callables);
    }

    @Override
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException {
        Collection<? extends Callable<T>> callables = TtlCallable.gets(tasks);
        return super.invokeAny(callables, timeout, unit);
    }

    @Override
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) throws InterruptedException {
        Collection<? extends Callable<T>> callables = TtlCallable.gets(tasks);
        return super.invokeAll(callables);
    }

    @Override
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException {
        Collection<? extends Callable<T>> callables = TtlCallable.gets(tasks);
        return super.invokeAll(callables, timeout, unit);
    }
}
```

**修饰ThreadPoolTaskExecutor**

```java
public class TtlThreadPoolTaskExecutor extends ThreadPoolTaskExecutor {

    @Override
    public void execute(Runnable command) {
        Runnable ttlRunnable = TtlRunnable.get(command);
        super.execute(ttlRunnable);
    }

    @Override
    public <T> Future<T> submit(Callable<T> task) {
        Callable ttCallable = TtlCallable.get(task);
        return super.submit(ttCallable);
    }

    @Override
    public Future<?> submit(Runnable task) {
        Runnable ttlRunnable = TtlRunnable.get(task);
        return super.submit(ttlRunnable);
    }

    @Override
    public ListenableFuture<?> submitListenable(Runnable task) {
        Runnable ttlRunnable = TtlRunnable.get(task);
        return super.submitListenable(ttlRunnable);
    }

    @Override
    public <T> ListenableFuture<T> submitListenable(Callable<T> task) {
        Callable ttlCallable = TtlCallable.get(task);
        return super.submitListenable(ttlCallable);
    }
}
```

**使用示例**

```java
    public static void main(String[] args) {
        TtlThreadPoolTaskExecutor paymentThreadPool = new TtlThreadPoolTaskExecutor();
        paymentThreadPool.setCorePoolSize(5);
        paymentThreadPool.setMaxPoolSize(10);
        paymentThreadPool.setKeepAliveSeconds(60);
        paymentThreadPool.setQueueCapacity(1000);
        paymentThreadPool.setThreadFactory(new ThreadFactoryBuilder().setNameFormat("trans-dispose-%d").build());
        paymentThreadPool.setRejectedExecutionHandler(new ThreadPoolExecutor.AbortPolicy());
        paymentThreadPool.initialize();
        paymentThreadPool.execute(() -> {
            System.out.println("异步线程执行");
        });
    }
```