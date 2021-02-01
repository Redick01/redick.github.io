# soul网关源码分析之spingwebflux-subscribeOn设计细节分析

## 目标

- soul网关应用spingwebflux
- soul网关使用subscribeOn巧妙设计分析
- 总结

## 起因

​    起因是群里一个小伙伴问到soul网关在`webHandler`中配置`Scheduler`是为啥，我想这位小伙伴一定是深有研究后才发起的话题，顺着这个话题我也打算一探究竟。

## soul网关应用spingwebflux

​    经过了这段时间对soul网关的研究，我们已经知道了，soul网关是集成了`spirng webflux`用来接收请求的，核心的代码是`SoulWebHandler`，该类实现了`spirng webflux`的`WebHandler`接口并且重写了`handle`方法，`handle`方法会`hold`住请求并且执行网关的`插件链`进行请求的处理的，下面是`SoulWebHandler`构造函数，我们可以看到构造函数中初始化了一个`Scheduler`，这个东西在我第一眼看到的时候我就认定他是一个和处理任务有关的类，如下图中代码，`插件链`就不说了，首先会根据获取配置`soul.scheduler.type`	的类型是否是`fixed`，如果是的话那就计算一个数值，后面会根据这个数值创建线程数为`threads`名称为`soul-work-threads`的线程池，如果类型不是`fixed`，那就创建一个`elastic()`的线程池，这是一个没有边界的弹性线程池。我通过`debug`soul网关发现默认这里配置的就是`fixed`，所以是创建了固定线程数据的线程池，也就是`Schedulers.newParallel("soul-work-threads", threads);`这段代码。

```
    private final List<SoulPlugin> plugins;

    private final Scheduler scheduler;

    /**
     * Instantiates a new Soul web handler.
     *
     * @param plugins the plugins
     */
    public SoulWebHandler(final List<SoulPlugin> plugins) {
        this.plugins = plugins;
        String schedulerType = System.getProperty("soul.scheduler.type", "fixed");
        if (Objects.equals(schedulerType, "fixed")) {
            int threads = Integer.parseInt(System.getProperty(
                    "soul.work.threads", "" + Math.max((Runtime.getRuntime().availableProcessors() << 1) + 1, 16)));
            scheduler = Schedulers.newParallel("soul-work-threads", threads);
        } else {
            scheduler = Schedulers.elastic();
        }
    }
```



## soul网关使用subscribeOn巧妙设计分析

​    网关启动，已经初始化了这个指定线程数的线程池`scheduler`，下面重点看的其实就是在那里使用了，并且他的作用是什么，顺着`SoulWebHandler`代码，我们看到了处理请求的`handle`方法中使用了`scheduler`，代码如下，这里可以看到在执行插件链后用了一个`subscribeOn`，重点来了，这个东西，其实就是应用创建好的线程池的重点，接下来分析`subscribeOn`以及应用它的设计技巧

```
    @Override
    public Mono<Void> handle(@NonNull final ServerWebExchange exchange) {
        MetricsTrackerFacade.getInstance().counterInc(MetricsLabelEnum.REQUEST_TOTAL.getName());
        Optional<HistogramMetricsTrackerDelegate> startTimer = MetricsTrackerFacade.getInstance().histogramStartTimer(MetricsLabelEnum.REQUEST_LATENCY.getName());
        return new DefaultSoulPluginChain(plugins).execute(exchange).subscribeOn(scheduler)
                .doOnSuccess(t -> startTimer.ifPresent(time -> MetricsTrackerFacade.getInstance().histogramObserveDuration(time)));
    }
```

### subscribeOn和publishOn

​    首先我们来了解一下`subscribeOn`，经过查阅资料知道了`subscribeOn`是用来切换线程上下文的，并且从名字上看，这个东西也是与订阅有关的，这就不得不说反应式编程基于`观察者模式`的发布-订阅了，再资料中我们还发现了一个与`subscribeOn`类似功能的方法`publishOn`，他俩的不同点就在于什么时候进行线程上下文的切换，`subscribeOn`只要是有调用就进行切换，`publishOn`则是在调用之后才会对调用后的流式操作进行线程上下文的切换，其实看到这里我还是没理解上，不着急，遇到这种问题，我们验证一下，下面是我的验证代码以及验证结果。

```
    @Test
    public void usePublishOnAndSubscribeOn() throws InterruptedException {
        Scheduler s = Schedulers.newParallel("parallel-scheduler", 4);
        // 创建Flux，先做一个map操作然后subscribeOn又进行一个map操作
        final Flux<String> flux = Flux
                .range(1, 2)
                .map(i -> i + 10 + ":" + Thread.currentThread().getName())
                .subscribeOn(s)
                .map(i -> i + 10 + ":" + Thread.currentThread().getName());
        new Thread(() -> flux.subscribe(System.out::println), "ThreadA").start();
        Thread.sleep(100);
        System.out.println("------------------------------");
        // 创建Flux，先做一个map操作然后publishOn又进行一个map操作
        final Flux<String> flux2 = Flux
                .range(1, 2)
                .map(i -> i + 10 + ":" + Thread.currentThread().getName())
                .publishOn(s)
                .map(i -> i + 10 + ":" + Thread.currentThread().getName());
        new Thread(() -> flux2.subscribe(System.out::println), "ThreadB").start();
        Thread.sleep(100);
    }
    
测试结果：  
11:parallel-scheduler-110:parallel-scheduler-1
12:parallel-scheduler-110:parallel-scheduler-1
------------------------------
11:ThreadB10:parallel-scheduler-2
12:ThreadB10:parallel-scheduler-2
```

- 测试结果分析

1. subscribeOn：可以看到不管是`subscribeOn`之前还是之后都用的是切换过的`parallel-scheduler`线程
2. publishOn：在`publishOn`之前，map操作使用的是线程`ThreadB`，`publishOn`之后，map操作使用的是切换过的线`parallel-scheduler`
3. 在soul网关中使用的`Mono`我测试代码用的是`Flux`，他们俩的区别是`Mono`操作的是0或者1个元素的异步序列，`Flux`则是操作的0或者N个元素的异步序列，我用`Flux`的目的其实就是在操作多个元素序列能够使测试的`publishOn`和`subscribeOn`前后的结果更加直观。

###  根据测试结果做总结

​    根据我的测试结果，我分析soul网关使用`subscribeOn`的目的是类似于`netty`的异步线程处理模型设计思路，就是说，`webHanlder`在`hold`住请求后通过`subscribeOn`切换线程异步的处理插件链，就相当于`netty`的`boss`线程拿到请求发送到`work`线程`EventLoopGroup`，然后`work`线程在异步的处理请求一样，分析到这里我以为我的思路大致就是soul网关作者的设计思路了

### 实际soul网关使用`subscribeOn`的设计思路

​    在讨论中，群里的小伙伴给出了soul网关使用`subscribeOn`的设计思路，其实是这样的，soul网关在接收到请求后会执行插件链，但是在执行插件链过程中因为都是异步的操作，组成一个反应式流的代码有快有慢，可能有的插件执行的快，有的插件执行的慢，如果单独的一个业务线程去处理的话，就会导致执行的快的还要等待执行的慢的插件，所以使用`subscribeOn`多线程的进行处理将其隔离处理，听到这里我也是恍然大悟不得不说这里设计的的确是很精妙。

## 总结
​
​    通过一个小设计的讨论，理解了soul网关在处理插件链的时候使用`subscribeOn`的设计思路，收获蛮大的，好的设计思路要借鉴如果遇到合适的应用场景也能够使用。