# Dubbo服务启动-Dubbo Provider发布服务 <!-- {docsify-ignore-all} -->

&nbsp; &nbsp; Dubbo `ServiceBean`加载完成后，会发布`dubbo`服务，发布服务的入口是`DubboBootstrap#exportServices()`方法，接下来我们看一下Provider发布服务的流程

## DubboBootstrap#exportServices();发布dubbo服务

```java
    private void exportServices() {
        configManager.getServices().forEach(sc -> {
            // TODO, compatible with ServiceConfig.export()
            ServiceConfig serviceConfig = (ServiceConfig) sc;
            serviceConfig.setBootstrap(this);
            // 异步发布
            if (exportAsync) {
                ExecutorService executor = executorRepository.getServiceExporterExecutor();
                Future<?> future = executor.submit(() -> {
                    sc.export();
                    exportedServices.add(sc);
                });
                asyncExportingFutures.add(future);
            } else {
                // 发布服务
                sc.export();
                // 将发布服务放到已发布服务列表里
                exportedServices.add(sc);
            }
        });
    }
```

### ServiceConfig#export发布服务

&nbsp; &nbsp; ServiceConfig#export是发布接口的方法，方法流程是检查bootstrap初始化情况和原数据初始化然后执行`doExport();`发布

```java
    public synchronized void export() {
        if (!shouldExport()) {
            return;
        }
        // bootstrap没有初始化，则初始化bootstrap
        if (bootstrap == null) {
            bootstrap = DubboBootstrap.getInstance();
            bootstrap.init();
        }

        checkAndUpdateSubConfigs();

        // 初始化原数据
        serviceMetadata.setVersion(version);
        serviceMetadata.setGroup(group);
        serviceMetadata.setDefaultGroup(group);
        serviceMetadata.setServiceType(getInterfaceClass());
        serviceMetadata.setServiceInterfaceName(getInterface());
        serviceMetadata.setTarget(getRef());

        if (shouldDelay()) {
            DELAY_EXPORT_EXECUTOR.schedule(this::doExport, getDelay(), TimeUnit.MILLISECONDS);
        } else {
            doExport();
        }

        exported();
    }
```

### ServiceConfig#doExport();


```java
    protected synchronized void doExport() {
        if (unexported) {
            throw new IllegalStateException("The service " + interfaceClass.getName() + " has already unexported!");
        }
        if (exported) {
            return;
        }
        // 设置exported状态
        exported = true;

        if (StringUtils.isEmpty(path)) {
            path = interfaceName;
        }
        // 发布服务
        doExportUrls();
    }
```

### ServiceConfig#doExportUrls();

```java
    @SuppressWarnings({"unchecked", "rawtypes"})
    private void doExportUrls() {
        ServiceRepository repository = ApplicationModel.getServiceRepository();
        ServiceDescriptor serviceDescriptor = repository.registerService(getInterfaceClass());
        // 注册provider到ServiceRepository
        repository.registerProvider(
                getUniqueServiceName(),
                ref,
                serviceDescriptor,
                this,
                serviceMetadata
        );
        // 注册中心
        List<URL> registryURLs = ConfigValidationUtils.loadRegistries(this, true);

        for (ProtocolConfig protocolConfig : protocols) {
            String pathKey = URL.buildKey(getContextPath(protocolConfig)
                    .map(p -> p + "/" + path)
                    .orElse(path), group, version);
            // 注册用户指定的路径
            repository.registerService(pathKey, interfaceClass);
            // TODO, uncomment this line once service key is unified
            serviceMetadata.setServiceKey(pathKey);
            doExportUrlsFor1Protocol(protocolConfig, registryURLs);
        }
    }
```


### ServiceConfig#doExportUrlsFor1Protocol发布服务

&nbsp; &nbsp; doExportUrlsFor1Protocol主要发布服务到本地和远程

```java
    private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
        String name = protocolConfig.getName();
        // 如果name是空默认是dubbo
        if (StringUtils.isEmpty(name)) {
            name = DUBBO;
        }

        Map<String, String> map = new HashMap<String, String>();
        map.put(SIDE_KEY, PROVIDER_SIDE);
        // 追加配置信息
        ServiceConfig.appendRuntimeParameters(map);
        AbstractConfig.appendParameters(map, getMetrics());
        AbstractConfig.appendParameters(map, getApplication());
        AbstractConfig.appendParameters(map, getModule());
        // remove 'default.' prefix for configs from ProviderConfig
        // appendParameters(map, provider, Constants.DEFAULT_KEY);
        AbstractConfig.appendParameters(map, provider);
        AbstractConfig.appendParameters(map, protocolConfig);
        AbstractConfig.appendParameters(map, this);
        ...省略部分代码，省略了方法配置的处理
        // 初始化服务原数据的attachments
        serviceMetadata.getAttachments().putAll(map);

        // 发布服务
        // 服务ip
        String host = findConfigedHosts(protocolConfig, registryURLs, map);
        // 端口
        Integer port = findConfigedPorts(protocolConfig, name, map);
        // 发布的URL
        URL url = new URL(name, host, port, getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), map);

        // 通过SPI加载自定义的附加参数，简单demo中不会走到
        if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                .hasExtension(url.getProtocol())) {
            url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                    .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
        }

        String scope = url.getParameter(SCOPE_KEY);
        // 当scope配置的是none，不进行发布服务
        if (!SCOPE_NONE.equalsIgnoreCase(scope)) {

            // 发布到本地
            if (!SCOPE_REMOTE.equalsIgnoreCase(scope)) {
                exportLocal(url);
            }
            // 发不到远程
            if (!SCOPE_LOCAL.equalsIgnoreCase(scope)) {
                // 注册中心存在
                if (CollectionUtils.isNotEmpty(registryURLs)) {
                    for (URL registryURL : registryURLs) {
                        // 协议如果是injvm，不发布到远程了
                        if (LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
                            continue;
                        }
                        // 服务url
                        url = url.addParameterIfAbsent(DYNAMIC_KEY, registryURL.getParameter(DYNAMIC_KEY));
                        // 监控URL，demo中没有，是null
                        URL monitorUrl = ConfigValidationUtils.loadMonitor(this, registryURL);
                        if (monitorUrl != null) {
                            // 如果监控URL不是空添加参数到服务URL中
                            url = url.addParameterAndEncoded(MONITOR_KEY, monitorUrl.toFullString());
                        }
                        if (logger.isInfoEnabled()) {
                            if (url.getParameter(REGISTER_KEY, true)) {
                                logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
                            } else {
                                logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
                            }
                        }

                        // For providers, 自定义代理
                        String proxy = url.getParameter(PROXY_KEY);
                        if (StringUtils.isNotEmpty(proxy)) {
                            registryURL = registryURL.addParameter(PROXY_KEY, proxy);
                        }
                        // 通过SPI机制加载实现动态代理方式，默认是javassist操作字节码，通过Wrapper类使用javassist进行字节码增强，动态生成类
                        // ProxyFactory获取Invoker
                        Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(EXPORT_KEY, url.toFullString()));
                        // 包装Invoker
                        DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
                        // 发布服务，通过SPI机制加载Protocol，ProtocolListenerWrapper#expot发布服务 
                        // 1、QosProtocolWrapper启动Qos
                        // 2、注册中心RegistryProtocol发布服务
                        // 3、DubboProtocol#export发布服务 openServer网络服务创建，createServer创建网络Server（DubboProtocolServer）
                        Exporter<?> exporter = PROTOCOL.export(wrapperInvoker);
                        exporters.add(exporter);
                    }
                } else {
                    if (logger.isInfoEnabled()) {
                        logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
                    }
                    Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, url);
                    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                    Exporter<?> exporter = PROTOCOL.export(wrapperInvoker);
                    exporters.add(exporter);
                }
                /**
                 * @since 2.7.0
                 * ServiceData Store
                 */
                 // 元数据存储，SPI机制获取元数据存储服务，默认是local，这里获取到的是local的InMemoryWritableMetadataService
                WritableMetadataService metadataService = WritableMetadataService.getExtension(url.getParameter(METADATA_KEY, DEFAULT_METADATA_STORAGE_TYPE));
                if (metadataService != null) {
                    // 元数据存到map中
                    metadataService.publishServiceDefinition(url);
                }
            }
        }
        this.urls.add(url);
    }
```

## 生成Invoker

&nbsp; &nbsp; Invoker是一个调用器，主要的功能是客户端请求服务时，服务端会查找发布服务时生成的Invoker方法，根据Invocation对象指定的方法和参数，执行其doInvoke方法，并将结果包装为Result对象返回给客户端，从而实现远程调用。

&nbsp; &nbsp; Dubbo中生成Invoker的代码在ServiceConfig中，如下：

&nbsp; &nbsp; dubbo使用proxyFactory的getInvoker生成Invoker实例。我们先来讨论一下proxyFactory，根据ExtensionLoader的实现，proxyFactory获取Adaptive类时，首先找@Adaptive注解的类，如果没有会由dubbo创建一个新的类ProxyFactory$Adaptive，其内部会根据url的proxy值获取factory，默认取proxyFactor接口上的注解@SPI("javassist")指定的值，getExtension()时会根据传入的参数获取对应的ProxyFactory实现类，还会查找ProxyFactory实现类中将proxyFactory作为唯一参数的构造函数的实现类作为Wrapper类进行包装。

```java
// 默认javassist
private static final ProxyFactory PROXY_FACTORY = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();

Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(EXPORT_KEY, url.toFullString()));
```

对ProxyFactory接口来说，最终生成的类是StubProxyFactoryWrapper，内部的proxyFactory为JavassistProxyFactory。proxyFactory变量的对象链如下：

```
-| StubProxyFactoryWrapper
	-| JavassistProxyFactory
```

- **StubProxyFactoryWrapper的getInvoker方法会调用内部JavassistProxyFactory的getInvoker**

&nbsp; &nbsp; 首先通过`Wrapper.getWrapper`将指定的ref类包装为Wrapper类，Wrapper类内部包含`invokeMethod(Object instance, String mn, Class<?>[] types, Object[] args)`可以调用指定实例的某个方法。最后将Wrapper包装为一个AbstractProxyInvoker类。`Wrapper类动态生成`，内部会直接调用实现类的方法，而不是使用反射调用，下面是invokeMethod的实现。

&nbsp; &nbsp; 经过这一系列的过程，完成了Invoker对象的创建，下面是内部调用结构：

```
|-AbstractProxyInvoker:invoker()
	|-Wrapper:invokeMethod()
    		|-DemoService : 指定的方法
```

```java
    @Override
    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        // TODO Wrapper类不能正确处理带$的类名，会调用makeWrapper生成Wrapper
        final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName,
                                      Class<?>[] parameterTypes,
                                      Object[] arguments) throws Throwable {
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
    }
```

## 发布Invoker

### ServiceConfig#exportLocal发布服务到本地


&nbsp; &nbsp; injvm协议服务发布到本地，ProxyFactory获取Invoker，通过SPI机制获取Protocol实现，调用Protocol实现的export发布服务

```java
    /**
     * 发布协议为injvm的服务
     */
    private void exportLocal(URL url) {
        URL local = URLBuilder.from(url)
                .setProtocol(LOCAL_PROTOCOL)
                .setHost(LOCALHOST_VALUE)
                .setPort(0)
                .build();
        // ProxyFactory获取Invoker，通过SPI机制获取Protocol实现，调用Protocol实现的export发布服务
        Exporter<?> exporter = PROTOCOL.export(
                PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, local));
        exporters.add(exporter);
        logger.info("Export dubbo service " + interfaceClass.getName() + " to local registry url : " + local);
    }
```

&nbsp; &nbsp; 在引用服务流程中，我们已经分析了获取Proxy对象的过程，每个协议获取Invoker的方式不同，对于injvm协议来说，protocol被包装为：

```
|- ProtocolFilterWrapper
	|- ProtocolListenerWrapper
		|- InjvmProtocol 
```

- **InjvmProtocol#export**

&nbsp; &nbsp; 创建`InjvmExporter`实例，并且缓存`InjvmExporter`实例到`exporterMap`中，缓存内容是服务url和`InjvmExporter实例`的映射。

```java
    @Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        return new InjvmExporter<T>(invoker, invoker.getUrl().getServiceKey(), exporterMap);
    }
```

&nbsp; &nbsp; InjvmExporter构造函数，将服务key和InjvmExporter映射关系存到exporterMap中。

```java
    InjvmExporter(Invoker<T> invoker, String key, Map<String, Exporter<?>> exporterMap) {
        super(invoker);
        this.key = key;
        this.exporterMap = exporterMap;
        exporterMap.put(key, this);
    }
```

### 发布到远程

&nbsp; &nbsp; 由于不同协议发布的方式不同，因此根据dubbo的理念，会根据url加载不同的protocol实现类来发布AbstractProxyInvoker的实例。与ProxyFactory类似，ExtensionLoader加载Protocol后的protocol实例为：

```
|- ProtocolFilterWrapper
	|- ProtocolListenerWrapper 
		|- RegistryProtocol 
```

- **RegistryProtocol#export发布注册服务和订阅服务**

&nbsp; &nbsp; 在发布流程中，我们已经知道了服务要发布为dubbo协议时，不同点在发布Invoker的不同。非injvm协议都使用了RegistryProtocol的export()来发布服务，RegistryProtocol的内部变量bounds中保存了<服务，协议>对应的Exporter，每次发布后会保存到这个map中。代码如下：

```java
    private final ConcurrentMap<String, ExporterChangeableWrapper<?>> bounds = new ConcurrentHashMap<>();

    @Override
    public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
        // 注册中心url
        URL registryUrl = getRegistryUrl(originInvoker);
        // provider url
        URL providerUrl = getProviderUrl(originInvoker);

        // Subscribe the override data
        // FIXME When the provider subscribes, it will affect the scene : a certain JVM exposes the service and call
        //  the same service. Because the subscribed is cached key with the name of the service, it causes the
        //  获取订阅服务URL
        final URL overrideSubscribeUrl = getSubscribedOverrideUrl(providerUrl);
        final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
        overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);

        providerUrl = overrideUrlWithConfig(providerUrl, overrideSubscribeListener);
        // 使用dubbo协议发布
        final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker, providerUrl);

        // 获取注册中心
        final Registry registry = getRegistry(originInvoker);
        // 获取注册provider url
        final URL registeredProviderUrl = getUrlToRegistry(providerUrl, registryUrl);

        // decide if we need to delay publish
        boolean register = providerUrl.getParameter(REGISTER_KEY, true);
        if (register) {
            // 注册到配置中心，demo中是zk
            register(registryUrl, registeredProviderUrl);
        }

        // register stated url on provider model
        registerStatedUrl(registryUrl, registeredProviderUrl, register);

        // 向服务中心订阅服务，并添加监听类
        registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);

        exporter.setRegisterUrl(registeredProviderUrl);
        exporter.setSubscribeUrl(overrideSubscribeUrl);

        notifyExport(exporter);
        //Ensure that a new exporter instance is returned every time export
        return new DestroyableExporter<>(exporter);
    }   
```

&nbsp; &nbsp; dubbo协议发布服务时，会根据发布时生成的Invoker，构建InvokerFilterChain，并添加监听事件，最后，打开协议指定的服务器，等待客户端连接后处理调用。
doLocalExport(originInvoker);中首先根据服务名在bounds之后查找对应的Exporter，如果找到，说明已经发不过了；如果没有找到则使用DubboProtocol协议发布Invoker。在发布之前，会将发布之前生成的Invoker包装为InvokerDelegete对象，这是因为originInvoker的url是注册中心协议的urlregistry://xxxx/xxx?xx;而dubboProtocol发布时需要改为dubbo://xxx/xx?xxx

```java
private <T> ExporterChangeableWrapper<T> doLocalExport(final Invoker<T> originInvoker, URL providerUrl) {
        String key = getCacheKey(originInvoker);

        return (ExporterChangeableWrapper<T>) bounds.computeIfAbsent(key, s -> {
            Invoker<?> invokerDelegate = new InvokerDelegate<>(originInvoker, providerUrl);
            return new ExporterChangeableWrapper<>((Exporter<T>) protocol.export(invokerDelegate), originInvoker);
        });
    }
```

- **DubboProtocol#export发布dubbo服务**

&nbsp; &nbsp; 继续分析dubboProtocol的export，protocol变量与之前相同，会根据协议名称获取协议链

```java
|- ProtocolFilterWrapper
	|- ProtocolListenerWrapper
    	|- DubboProtocol
```

&nbsp; &nbsp; `ProtocolFilterWrapper#buildInvokerChain`构造Invoker过滤器链


&nbsp; &nbsp; 用来生成调用链，内部的buildInvokerChain方法会查找Filter的实现类，查找group为provider的，并根据order排序，将这些Filter连接成一个调用链 InvokerFilterChain

```
EchoFilter -> ClassloaderFilter -> GenericFilter -> 
ContextFilter -> TraceFilter -> TimeoutFilter ->
MonitorFilter -> ExceptionFilter -> InvokerDelegete
```

```java
    private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
        Invoker<T> last = invoker;
        // SPI机制获取所有过滤器Filter
        List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);

        if (!filters.isEmpty()) {
            for (int i = filters.size() - 1; i >= 0; i--) {
                final Filter filter = filters.get(i);
                final Invoker<T> next = last;
                last = new Invoker<T>() {

                    ...省略
                };
            }
        }

        return last;
    }
```

- **DubboProtocol#export**

```java
    @Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        URL url = invoker.getUrl();

        // export service.
        String key = serviceKey(url);
        DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
        exporterMap.put(key, exporter);

        //export an stub service for dispatching event
        Boolean isStubSupportEvent = url.getParameter(STUB_EVENT_KEY, DEFAULT_STUB_EVENT);
        Boolean isCallbackservice = url.getParameter(IS_CALLBACK_SERVICE, false);
        if (isStubSupportEvent && !isCallbackservice) {
            String stubServiceMethods = url.getParameter(STUB_EVENT_METHODS_KEY);
            if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
                if (logger.isWarnEnabled()) {
                    logger.warn(new IllegalStateException("consumer [" + url.getParameter(INTERFACE_KEY) +
                            "], has set stubproxy support event ,but no stub methods founded."));
                }

            }
        }
        // 打开Server，会创建Server
        openServer(url);
        // 指定的序列化
        optimizeSerialization(url);

        return exporter;
    }
```

- **DubboProtocol打开服务器，底层通信实现**

&nbsp; &nbsp; 创建Server，创建`ExchangeServer`，默认使用`netty`作为Server的实现服务器

```java
    private void openServer(URL url) {
        // find server.
        String key = url.getAddress();
        //client can export a service which's only for server to invoke
        boolean isServer = url.getParameter(IS_SERVER_KEY, true);
        if (isServer) {
            ProtocolServer server = serverMap.get(key);
            // 从Server缓存中获取不到就创建一个Server放到map中
            if (server == null) {
                synchronized (this) {
                    server = serverMap.get(key);
                    if (server == null) {
                        // 创建Server 并放到map中
                        serverMap.put(key, createServer(url));
                    }
                }
            } else {
                // server supports reset, use together with override
                server.reset(url);
            }
        }
    }

    private ProtocolServer createServer(URL url) {
        url = URLBuilder.from(url)
                // send readonly event when server closes, it's enabled by default
                .addParameterIfAbsent(CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString())
                // enable heartbeat by default
                .addParameterIfAbsent(HEARTBEAT_KEY, String.valueOf(DEFAULT_HEARTBEAT))
                .addParameter(CODEC_KEY, DubboCodec.NAME)
                .build();
        // 默认netty
        String str = url.getParameter(SERVER_KEY, DEFAULT_REMOTING_SERVER);

        if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
            throw new RpcException("Unsupported server type: " + str + ", url: " + url);
        }

        ExchangeServer server;
        try {
            // Dubbo服务器创建
            server = Exchangers.bind(url, requestHandler);
        } catch (RemotingException e) {
            throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
        }

        str = url.getParameter(CLIENT_KEY);
        if (str != null && str.length() > 0) {
            Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions();
            if (!supportedTypes.contains(str)) {
                throw new RpcException("Unsupported client type: " + str);
            }
        }

        return new DubboProtocolServer(server);
    }
```

## 总结

&nbsp; &nbsp; Dubbo发布服务主要是通过`ServiceConfig`的`export`方法，发布过服务过程中会通过`SPI`技术生成代理创建Invoker，Dubbo默认使用`javassist技术`实现动态代理；Dubbo通过协议调用链通过`DubboProtocol#export发布dubbo服务`，发布的过程会创建服务器，并且绑定服务器。绑定服务器的接口为`Exchangers`，Dubbo提供了`netty`，`Mina`等实现，下一篇我们一起看一下Dubbo底层的网络通信详细了解一下这部分。