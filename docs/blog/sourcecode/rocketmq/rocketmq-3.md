# RocketMQ源码解析-Namesrv启动流程

- Namesrv启动流程
- Netty客户端请求处理
- 总结


## Namesrv启动流程

### org.apache.rocketmq.namesrv.NamesrvStartup

&nbsp; &nbsp; `Namesrv`作为`RocketMQ`的注册中心，其本身是一个服务端，其他的组件与之通信实现组件注册，组件路由，集群管理等功能，`org.apache.rocketmq.namesrv.NamesrvStartup`类作为其启动类，main方法的启动流程是根据启动命令，启动参数，创建`org.apache.rocketmq.namesrv.NamesrvController`对象，然后调用`start`方法，该方法的流程是先进行`controller.initialize(）`进行一些初始化操作，然后注册`NamesrvController`的`shutdown`方法的回调钩子，其作用是程序`Namesrv`退出时执行`NamesrvController`的`shutdown`方法，最后调用`controller.start(）`。下面看一下代码：

```
    public static void main(String[] args) {
        main0(args);
    }

    public static NamesrvController main0(String[] args) {

        try {
            // 创建NamesrvController对象
            NamesrvController controller = createNamesrvController(args);
            // 启动
            start(controller);
            String tip = "The Name Server boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();
            log.info(tip);
            System.out.printf("%s%n", tip);
            return controller;
        } catch (Throwable e) {
            e.printStackTrace();
            System.exit(-1);
        }

        return null;
    }
    public static NamesrvController createNamesrvController(String[] args) throws IOException, JoranException {
        System.setProperty(RemotingCommand.REMOTING_VERSION_KEY, Integer.toString(MQVersion.CURRENT_VERSION));
        //PackageConflictDetect.detectFastjson();
        // rocketmq启动，命令行参数解析
        Options options = ServerUtil.buildCommandlineOptions(new Options());
        // 获取命令行
        commandLine = ServerUtil.parseCmdLine("mqnamesrv", args, buildCommandlineOptions(options), new PosixParser());
        if (null == commandLine) {
            System.exit(-1);
            return null;
        }

        final NamesrvConfig namesrvConfig = new NamesrvConfig();
        final NettyServerConfig nettyServerConfig = new NettyServerConfig();
        // 默认监听端口
        nettyServerConfig.setListenPort(9876);
        if (commandLine.hasOption('c')) {
            // 从命令行参数获取配置文件
            String file = commandLine.getOptionValue('c');
            if (file != null) {
                // 读取配置文件
                InputStream in = new BufferedInputStream(new FileInputStream(file));
                properties = new Properties();
                // 将配置文件配置转成Properties
                properties.load(in);
                // 序列化配置到NamesrvConfig和NettyServerConfig
                MixAll.properties2Object(properties, namesrvConfig);
                MixAll.properties2Object(properties, nettyServerConfig);

                namesrvConfig.setConfigStorePath(file);

                System.out.printf("load config properties file OK, %s%n", file);
                in.close();
            }
        }

        if (commandLine.hasOption('p')) {
            InternalLogger console = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_CONSOLE_NAME);
            MixAll.printObjectProperties(console, namesrvConfig);
            MixAll.printObjectProperties(console, nettyServerConfig);
            System.exit(0);
        }

        MixAll.properties2Object(ServerUtil.commandLine2Properties(commandLine), namesrvConfig);

        if (null == namesrvConfig.getRocketmqHome()) {
            System.out.printf("Please set the %s variable in your environment to match the location of the RocketMQ installation%n", MixAll.ROCKETMQ_HOME_ENV);
            System.exit(-2);
        }

        LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();
        JoranConfigurator configurator = new JoranConfigurator();
        configurator.setContext(lc);
        lc.reset();
        // 日志配置文件
        configurator.doConfigure(namesrvConfig.getRocketmqHome() + "/conf/logback_namesrv.xml");

        log = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_LOGGER_NAME);

        MixAll.printObjectProperties(log, namesrvConfig);
        MixAll.printObjectProperties(log, nettyServerConfig);

        final NamesrvController controller = new NamesrvController(namesrvConfig, nettyServerConfig);

        // remember all configs to prevent discard
        controller.getConfiguration().registerConfig(properties);

        return controller;
    }
    public static NamesrvController start(final NamesrvController controller) throws Exception {

        if (null == controller) {
            throw new IllegalArgumentException("NamesrvController is null");
        }
        // 一些初始化操作，包括NettyServer配置，KV配置加载，NettyHandler的注册等
        boolean initResult = controller.initialize();
        if (!initResult) {
            controller.shutdown();
            System.exit(-3);
        }
        // 注册钩子
        Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, new Callable<Void>() {
            @Override
            public Void call() throws Exception {
                controller.shutdown();
                return null;
            }
        }));
        // 启动
        controller.start();

        return controller;
    }
    public static void shutdown(final NamesrvController controller) {
        controller.shutdown();
    }
```

### org.apache.rocketmq.namesrv.NamesrvController

&nbsp; &nbsp; `org.apache.rocketmq.namesrv.NamesrvController`的`initialize()`和`start()`方法会在`Namesrv`启动的时候被调用，`initialize()`方法会初始化NettyServer配置，KV配置加载，请求处理器的注册等，`start()`方法会启动服务

#### initialize()方法

&nbsp; &nbsp; `this.kvConfigManager.load();`会加载KV配置，下一篇会详细分析KV配置的加载；`this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);`会初始化NettyServer，rocketMq的网络通信是基于Netty做的，这部分的核心代码在其`remoting`模块，后面详细会看；以后的流程就是初始化一个Netty的工程线程池，初始化网络请求的处理器，创建一个每隔10s检测Broker路由列表中Broker的存活情况的定时线程池，初始化定时打印 KV配置的线程池，最后是TSL协议处理的东西，代码如下，可以参考注释

```
    public boolean initialize() {
        // 加载配置
        this.kvConfigManager.load();
        // NettyServer实例化
        this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);
        // 初始化Netty工作线程
        this.remotingExecutor =
            Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));
        // 初始化请求处理器
        this.registerProcessor();
        // 每隔10s检测Broker路由列表中Broker的存活情况
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                NamesrvController.this.routeInfoManager.scanNotActiveBroker();
            }
        }, 5, 10, TimeUnit.SECONDS);
        // 定时打印 KV配置
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                NamesrvController.this.kvConfigManager.printAllPeriodically();
            }
        }, 1, 10, TimeUnit.MINUTES);
        // TLS协议处理，会监听文件变化，如果文件有变动，会重新加载协议文件
        if (TlsSystemConfig.tlsMode != TlsMode.DISABLED) {
            // Register a listener to reload SslContext
            try {
                fileWatchService = new FileWatchService(
                    new String[] {
                        TlsSystemConfig.tlsServerCertPath,
                        TlsSystemConfig.tlsServerKeyPath,
                        TlsSystemConfig.tlsServerTrustCertPath
                    },
                    new FileWatchService.Listener() {
                        boolean certChanged, keyChanged = false;
                        @Override
                        public void onChanged(String path) {
                            if (path.equals(TlsSystemConfig.tlsServerTrustCertPath)) {
                                log.info("The trust certificate changed, reload the ssl context");
                                reloadServerSslContext();
                            }
                            if (path.equals(TlsSystemConfig.tlsServerCertPath)) {
                                certChanged = true;
                            }
                            if (path.equals(TlsSystemConfig.tlsServerKeyPath)) {
                                keyChanged = true;
                            }
                            if (certChanged && keyChanged) {
                                log.info("The certificate and private key changed, reload the ssl context");
                                certChanged = keyChanged = false;
                                reloadServerSslContext();
                            }
                        }
                        private void reloadServerSslContext() {
                            ((NettyRemotingServer) remotingServer).loadSslContext();
                        }
                    });
            } catch (Exception e) {
                log.warn("FileWatchService created error, can't load the certificate dynamically");
            }
        }

        return true;
    }
```

#### start()方法

&nbsp; &nbsp; 该方法没什么可说的，就是启动NettyServer，remoting模块的NettyServer创建细节以后看，在一个就是文件变动监控线程启动，两部分代码如下


```
    public void start() throws Exception {
        // NettyServer启动
        this.remotingServer.start();
        // 文件变动监控线程启动
        if (this.fileWatchService != null) {
            this.fileWatchService.start();
        }
    }
    // this.fileWatchService.start();
    public void start() {
        log.info("Try to start service thread:{} started:{} lastThread:{}", getServiceName(), started.get(), thread);
        if (!started.compareAndSet(false, true)) {
            return;
        }
        stopped = false;
        this.thread = new Thread(this, getServiceName());
        this.thread.setDaemon(isDaemon);
        this.thread.start();
    }
```

## Netty客户端请求处理

&nbsp; &nbsp; 上面一节分析了`Namesrv`的服务端启动流程，现在我看来具体看一下Namesrv作为服务端是怎样处理客户端的请求的，上面提到了在服务端启动时候会注册一个请求处理器`this.registerProcessor();`，下面看一下这段代码，可以看到不同的环境会为NettyServer实例化不同的处理器

```
    private void registerProcessor() {
        // 集群还是单边的配置
        if (namesrvConfig.isClusterTest()) {
            // 处理集群的处理器
            this.remotingServer.registerDefaultProcessor(new ClusterTestRequestProcessor(this, namesrvConfig.getProductEnvName()),
                this.remotingExecutor);
        } else {
            // 默认的处理器
            this.remotingServer.registerDefaultProcessor(new DefaultRequestProcessor(this), this.remotingExecutor);
        }
    }
```

&nbsp; &nbsp; 下面我们分析下`DefaultRequestProcessor`，它继承了`AsyncNettyRequestProcessor`，实现了`NettyRequestProcessor`，并重写了核心的`processRequest`方法，该方法就是处理请求，分发具体的某一个请求处理，代码如下，`DefaultRequestProcessor`提供了每一种请求参数解析与响应，KVManager和RouterInfoManager提供具体实现，我们在下一篇文章中继续分析。

```
public class DefaultRequestProcessor extends AsyncNettyRequestProcessor implements NettyRequestProcessor 
```
```
    @Override
    public RemotingCommand processRequest(ChannelHandlerContext ctx,
        RemotingCommand request) throws RemotingCommandException {

        if (ctx != null) {
            log.debug("receive request, {} {} {}",
                request.getCode(),
                RemotingHelper.parseChannelRemoteAddr(ctx.channel()),
                request);
        }

        // 根据request 的 code字段处理不同的请求
        switch (request.getCode()) {
            // 添加KV配置
            case RequestCode.PUT_KV_CONFIG:
                return this.putKVConfig(ctx, request);
            // 获取KV配置
            case RequestCode.GET_KV_CONFIG:
                return this.getKVConfig(ctx, request);
            // 删除KV配置
            case RequestCode.DELETE_KV_CONFIG:
                return this.deleteKVConfig(ctx, request);
            // 查询数据版本号
            case RequestCode.QUERY_DATA_VERSION:
                return queryBrokerTopicConfig(ctx, request);
            // 注册Broker
            case RequestCode.REGISTER_BROKER:
                Version brokerVersion = MQVersion.value2Version(request.getVersion());
                // 版本兼容处理
                if (brokerVersion.ordinal() >= MQVersion.Version.V3_0_11.ordinal()) {
                    return this.registerBrokerWithFilterServer(ctx, request);
                } else {
                    return this.registerBroker(ctx, request);
                }
            // 注销Broker
            case RequestCode.UNREGISTER_BROKER:
                return this.unregisterBroker(ctx, request);
            // 根据topic路由
            case RequestCode.GET_ROUTEINFO_BY_TOPIC:
                return this.getRouteInfoByTopic(ctx, request);
            // 获取Broker集群信息
            case RequestCode.GET_BROKER_CLUSTER_INFO:
                return this.getBrokerClusterInfo(ctx, request);
            case RequestCode.WIPE_WRITE_PERM_OF_BROKER:
                return this.wipeWritePermOfBroker(ctx, request);
            // 根据namespace获取所有的topic列表
            case RequestCode.GET_ALL_TOPIC_LIST_FROM_NAMESERVER:
                return getAllTopicListFromNameserver(ctx, request);
            // 删除topic
            case RequestCode.DELETE_TOPIC_IN_NAMESRV:
                return deleteTopicInNamesrv(ctx, request);
            // 根据namespace获取所有的KV列表
            case RequestCode.GET_KVLIST_BY_NAMESPACE:
                return this.getKVListByNamespace(ctx, request);
            //
            case RequestCode.GET_TOPICS_BY_CLUSTER:
                return this.getTopicsByCluster(ctx, request);
            case RequestCode.GET_SYSTEM_TOPIC_LIST_FROM_NS:
                return this.getSystemTopicListFromNs(ctx, request);
            case RequestCode.GET_UNIT_TOPIC_LIST:
                return this.getUnitTopicList(ctx, request);
            case RequestCode.GET_HAS_UNIT_SUB_TOPIC_LIST:
                return this.getHasUnitSubTopicList(ctx, request);
            case RequestCode.GET_HAS_UNIT_SUB_UNUNIT_TOPIC_LIST:
                return this.getHasUnitSubUnUnitTopicList(ctx, request);
            // 更新namesrv配置
            case RequestCode.UPDATE_NAMESRV_CONFIG:
                return this.updateConfig(ctx, request);
            // 根据namesrv配置
            case RequestCode.GET_NAMESRV_CONFIG:
                return this.getConfig(ctx, request);
            default:
                break;
        }
        return null;
    }
```

## 总结

&nbsp; &nbsp; `rockerMq`的`NameServer`本质是一个提供了KV配置管理和路由信息管理的Netty的服务端，在服务启动的时候会根据命令行参数以及配置文件进行服务端的初始化，然后注册请求处理工作线程，请求处理器，创建TLS协议文件变动监听线程，创建Broker探活线程等，最后启动Netty服务端；客户端的请求由`org.apache.rocketmq.namesrv.processor.DefaultRequestProcessor`的`processRequest`进行具体的分发处理及响应客户端，具体的处理细节在`KVConfigManager`和`RouteInfoManager`中，下一篇文档我们具体分析一下这两个类的核心。