# RocketMQ源码解析-Producer启动流程分析


## Demo

&nbsp; &nbsp; 参考简单的生产者生产消息示例`org.apache.rocketmq.example.batch.SimpleBatchProducer`，代码如下：

```
public class SimpleBatchProducer {

    public static void main(String[] args) throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer("BatchProducerGroupName");
        producer.start();

        //If you just send messages of no more than 1MiB at a time, it is easy to use batch
        //Messages of the same batch should have: same topic, same waitStoreMsgOK and no schedule support
        String topic = "BatchTest";
        List<Message> messages = new ArrayList<>();
        messages.add(new Message(topic, "Tag", "OrderID001", "Hello world 0".getBytes()));
        messages.add(new Message(topic, "Tag", "OrderID002", "Hello world 1".getBytes()));
        messages.add(new Message(topic, "Tag", "OrderID003", "Hello world 2".getBytes()));

        producer.send(messages);
    }
}
```

## 实例化Producer对象

&nbsp; &nbsp; 下面代码是实例化`Producer`对象，指定`Producer`的生产者组名称，实例化`org.apache.rocketmq.client.impl.producer.DefaultMQProducerImpl`，核心流程是初始化异步发送线程池队列，初始大小50000，初始化异步发送的线程池。
```
    public DefaultMQProducer(final String namespace, final String producerGroup, RPCHook rpcHook) {
        this.namespace = namespace;
        this.producerGroup = producerGroup;
        defaultMQProducerImpl = new DefaultMQProducerImpl(this, rpcHook);
    }

    public DefaultMQProducerImpl(final DefaultMQProducer defaultMQProducer, RPCHook rpcHook) {
        this.defaultMQProducer = defaultMQProducer;
        this.rpcHook = rpcHook;
        // 异步线程池队列
        this.asyncSenderThreadPoolQueue = new LinkedBlockingQueue<Runnable>(50000);
        // 异步发送消息线程池
        this.defaultAsyncSenderExecutor = new ThreadPoolExecutor(
            Runtime.getRuntime().availableProcessors(),
            Runtime.getRuntime().availableProcessors(),
            1000 * 60,
            TimeUnit.MILLISECONDS,
            this.asyncSenderThreadPoolQueue,
            new ThreadFactory() {
                private AtomicInteger threadIndex = new AtomicInteger(0);

                @Override
                public Thread newThread(Runnable r) {
                    return new Thread(r, "AsyncSenderExecutor_" + this.threadIndex.incrementAndGet());
                }
            });
    }
```

## Producer的start

&nbsp; &nbsp; `DefaultMQProducer`的`start`方法流程如下，设置生产者组，`DefaultMQProducerImpl`start，判断是否开启消息追踪，如果开启start消息追踪的实现，默认是`AsyncTraceDispatcher`。
```
    @Override
    public void start() throws MQClientException {
        // 设置生产者组
        this.setProducerGroup(withNamespace(this.producerGroup));
        this.defaultMQProducerImpl.start();
        // 是否开启消息追踪
        if (null != traceDispatcher) {
            try {
                // 启动消息追踪
                traceDispatcher.start(this.getNamesrvAddr(), this.getAccessChannel());
            } catch (MQClientException e) {
                log.warn("trace dispatcher start failed ", e);
            }
        }
    }
```

&nbsp; &nbsp; `DefaultMQProducerImpl`start流程如下：核心的流程是创建`MQClientInstance`实例，`MQClientInstance`封装了RocketMQ的网络处理的API，是Producer、Consumer与Namesrv、Broker进行网络交互的模块。

```
    public void start(final boolean startFactory) throws MQClientException {
        switch (this.serviceState) {
            case CREATE_JUST:
                // 初始化serviceState状态为ServiceState.START_FAILED
                this.serviceState = ServiceState.START_FAILED;
                // 检查参数
                this.checkConfig();
                // 生产者组不等于 CLIENT_INNER_PRODUCER时将InstanceName 转成PID
                if (!this.defaultMQProducer.getProducerGroup().equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {
                    this.defaultMQProducer.changeInstanceNameToPID();
                }
                // 创建生产者的MQClient实例，即MQClientInstance，
                this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQProducer, rpcHook);
                // 注册更新本地的生产者表，即producerTable，Namesrv端的生产者信息通过心跳等手段更新
                boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);
                // 存在旧生产者
                if (!registerOK) {
                    this.serviceState = ServiceState.CREATE_JUST;
                    throw new MQClientException("The producer group[" + this.defaultMQProducer.getProducerGroup()
                        + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                        null);
                }
                // 更新topic发布信息
                this.topicPublishInfoTable.put(this.defaultMQProducer.getCreateTopicKey(), new TopicPublishInfo());

                if (startFactory) {
                    mQClientFactory.start();
                }

                log.info("the producer [{}] start OK. sendMessageWithVIPChannel={}", this.defaultMQProducer.getProducerGroup(),
                    this.defaultMQProducer.isSendMessageWithVIPChannel());
                // producer启动成功，设置producer状态为运行中
                this.serviceState = ServiceState.RUNNING;
                break;
            case RUNNING:
            case START_FAILED:
            case SHUTDOWN_ALREADY:
                throw new MQClientException("The producer service state not OK, maybe started once, "
                    + this.serviceState
                    + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK),
                    null);
            default:
                break;
        }
        // 向所有Broker发送心跳
        this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
        // 定时remove timeout request
        this.timer.scheduleAtFixedRate(new TimerTask() {
            @Override
            public void run() {
                try {
                    RequestFutureTable.scanExpiredRequest();
                } catch (Throwable e) {
                    log.error("scan RequestFutureTable exception", e);
                }
            }
        }, 1000 * 3, 1000);
    }
```

&nbsp; &nbsp; `MQClientInstance`的`start`方法的核心功能为启动网络通信的客户端，即一个Netty的客户端；启动一些定时任务，如定时获取Namesrv的地址列表、定时从Namesrv获取Topic信息并更新、定时清理所有掉线的Broker并发送心跳、定时维持所有Consumer的消费进度；启动拉取消息服务；启动负载均衡服务。

```
    public void start() throws MQClientException {
        synchronized (this) {
            switch (this.serviceState) {
                case CREATE_JUST:
                    this.serviceState = ServiceState.START_FAILED;
                    // If not specified,looking address from name server
                    if (null == this.clientConfig.getNamesrvAddr()) {
                        this.mQClientAPIImpl.fetchNameServerAddr();
                    }
                    // Start request-response channel
                    this.mQClientAPIImpl.start();
                    // Start various schedule tasks
                    this.startScheduledTask();
                    // Start pull service
                    this.pullMessageService.start();
                    // Start rebalance service
                    this.rebalanceService.start();
                    // Start push service
                    this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
                    log.info("the client factory [{}] start OK", this.clientId);
                    this.serviceState = ServiceState.RUNNING;
                    break;
                case START_FAILED:
                    throw new MQClientException("The Factory object[" + this.getClientId() + "] has been created before, and failed.", null);
                default:
                    break;
            }
        }
    }
```

## 总结

&nbsp; &nbsp; 一个简单的RocketMQ的生产者的启动过程也相对很简单，首先就是要对`DefaultMQProducer`和`DefaultMQProducerImpl`进行初始化，然后就是生产者的启动，生产者的启动其核心就是创建`MQClientInstance`，然后调用其`start`方法，该方法的核心是创建并启动一个用于网络通信的Netty客户端；创建启动各种定时任务；启动拉取消息的线程；启动一个重新进行负载均衡的线程等，导致了一个生产者客户端的启动流程大致就看完了，感兴趣的可以自己捋一边代码，并且亲手debug一下，下一篇文章我们看一下啊发送消息的流程。