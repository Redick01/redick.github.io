# Dubbo服务启动-Dubbo Consumer引用服务 <!-- {docsify-ignore-all} -->



## Consumer消费者Demo示例

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans" xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">
    <context:property-placeholder/>

    <dubbo:application name="serialization-java-consumer">
        <dubbo:parameter key="qos.enable" value="true" />
        <dubbo:parameter key="qos.accept.foreign.ip" value="false" />
        <dubbo:parameter key="qos.port" value="33333" />
    </dubbo:application>

    <dubbo:registry address="zookeeper://${zookeeper.address:127.0.0.1}:2181"/>

    <dubbo:reference id="demoService" check="true" interface="org.apache.dubbo.samples.serialization.api.DemoService"/>

</beans>
```

&nbsp; &nbsp; 在之前的章节中已经知道，Dubbo基于Spring自定义标签规范实现了自定义标签，通过自定义标签完成了bean的加载，并且通过实现监听Spring容器刷新完毕事件启动dubbo客户端。启动客户端伴随着服务发布和服务的订阅。

```java
public class DubboConsumer {

    public static void main(String[] args) {
        ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("spring/dubbo-demo-consumer.xml");
        context.start();
        DemoService demoService = context.getBean("demoService", DemoService.class);
        String hello = demoService.sayHello("world");
        System.out.println(hello);
    }
}
```

&nbsp; &nbsp; dubbo通过`<dubbo:reference`标签引用服务，之后在程序中通过Spring的Context依赖查找(getBean)的方式获取引用的服务的代理实例。`<dubbo:reference`加载的Bean是ReferenceBean，它实现了FactoryBean接口，getBean时会调用ReferenceBean的getObject()方法，这是获取引用的入口。getBean方法会判断Reference对象是否是空的，如果是空的，调用init方法。代码如下：

```java
    @Override
    public Object getObject() {
        return get();
    }
```

## ReferenceConfig#getObject()获取应用Bean

&nbsp; &nbsp; `ReferenceBean`继承了`ReferenceConfig`，当调用ReferenceBean的getObject()方法会调用`ReferenceBean`的get()方法。

```java
    public synchronized T get() {
        if (destroyed) {
            throw new IllegalStateException("The invoker of ReferenceConfig(" + url + ") has already destroyed!");
        }
        // 代理引用如果是空的，调用init
        if (ref == null) {
            init();
        }
        return ref;
    }

    public synchronized void init() {
        if (initialized) {
            return;
        }

        if (bootstrap == null) {
            bootstrap = DubboBootstrap.getInstance();
            bootstrap.init();
        }
        // 1. 检查配置ConsumerConfig，有的话检查配置，没有就新建一个ConsumerConfig
        // 2. 反射创建调用的API
        // 3. 初始化ServiceMetadata
        // 4. 注册Consumer
        // 5. 检查ReferenceConfig，RegistryConfig，ConsumerConfig
        checkAndUpdateSubConfigs();

        checkStubAndLocal(interfaceClass);
        // 检查引用的接口是否mock
        ConfigValidationUtils.checkMock(interfaceClass, this);
        // consumer的信息
        Map<String, String> map = new HashMap<String, String>();
        map.put(SIDE_KEY, CONSUMER_SIDE);
        // 添加运行时参数到map，包括：dubbo，release，timestamp，pid
        ReferenceConfigBase.appendRuntimeParameters(map);
        // 是不是泛化，不是的话进入条件
        if (!ProtocolUtils.isGeneric(generic)) {
            String revision = Version.getVersion(interfaceClass, version);
            if (revision != null && revision.length() > 0) {
                map.put(REVISION_KEY, revision);
            }
            // 获取方法，生成包装类，使用javassist，将生成的类放到WRAPPER_MAP中，key是org.apache.dubbo.samples.serialization.api.DemoService类对象，value是包装类的实例
            String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
            if (methods.length == 0) {
                logger.warn("No method found in service interface " + interfaceClass.getName());
                map.put(METHODS_KEY, ANY_VALUE);
            } else {
                // method放到map，这里method是sayHello
                map.put(METHODS_KEY, StringUtils.join(new HashSet<String>(Arrays.asList(methods)), COMMA_SEPARATOR));
            }
        }
        // interface org.apache.dubbo.samples.serialization.api.DemoService
        map.put(INTERFACE_KEY, interfaceName);
        // 追加其他参数到map中
        AbstractConfig.appendParameters(map, getMetrics());
        AbstractConfig.appendParameters(map, getApplication());
        AbstractConfig.appendParameters(map, getModule());
        // remove 'default.' prefix for configs from ConsumerConfig
        // appendParameters(map, consumer, Constants.DEFAULT_KEY);
        AbstractConfig.appendParameters(map, consumer);
        AbstractConfig.appendParameters(map, this);
        MetadataReportConfig metadataReportConfig = getMetadataReportConfig();
        if (metadataReportConfig != null && metadataReportConfig.isValid()) {
            map.putIfAbsent(METADATA_KEY, REMOTE_METADATA_STORAGE_TYPE);
        }
        Map<String, AsyncMethodInfo> attributes = null;
        if (CollectionUtils.isNotEmpty(getMethods())) {
            attributes = new HashMap<>();
            for (MethodConfig methodConfig : getMethods()) {
                AbstractConfig.appendParameters(map, methodConfig, methodConfig.getName());
                String retryKey = methodConfig.getName() + ".retry";
                if (map.containsKey(retryKey)) {
                    String retryValue = map.remove(retryKey);
                    if ("false".equals(retryValue)) {
                        map.put(methodConfig.getName() + ".retries", "0");
                    }
                }
                AsyncMethodInfo asyncMethodInfo = AbstractConfig.convertMethodConfig2AsyncInfo(methodConfig);
                if (asyncMethodInfo != null) {
//                    consumerModel.getMethodModel(methodConfig.getName()).addAttribute(ASYNC_KEY, asyncMethodInfo);
                    attributes.put(methodConfig.getName(), asyncMethodInfo);
                }
            }
        }
        
        String hostToRegistry = ConfigUtils.getSystemProperty(DUBBO_IP_TO_REGISTRY);
        if (StringUtils.isEmpty(hostToRegistry)) {
            hostToRegistry = NetUtils.getLocalHost();
        } else if (isInvalidLocalHost(hostToRegistry)) {
            throw new IllegalArgumentException("Specified invalid registry ip from property:" + DUBBO_IP_TO_REGISTRY + ", value:" + hostToRegistry);
        }
        map.put(REGISTER_IP_KEY, hostToRegistry);
        // 所有数据在存到serviceMetadata的attachments中
        serviceMetadata.getAttachments().putAll(map);
        // 创建Service代理
        ref = createProxy(map);
        // 设置ServiceMetadata的Service代理引用
        serviceMetadata.setTarget(ref);
        serviceMetadata.addAttribute(PROXY_CLASS_REF, ref);
        ConsumerModel consumerModel = repository.lookupReferredService(serviceMetadata.getServiceKey());
        consumerModel.setProxyObject(ref);
        consumerModel.init(attributes);
        // 标记初始化完毕
        initialized = true;

        // dispatch a ReferenceConfigInitializedEvent since 2.7.4
        dispatch(new ReferenceConfigInitializedEvent(this, invoker));
    }
```

## ReferenceConfig#createProxy()创建服务代理

```java
    @SuppressWarnings({"unchecked", "rawtypes", "deprecation"})
    private T createProxy(Map<String, String> map) {
        // 是否是InJvm，协议是InJvm
        if (shouldJvmRefer(map)) {
            URL url = new URL(LOCAL_PROTOCOL, LOCALHOST_VALUE, 0, interfaceClass.getName()).addParameters(map);
            invoker = REF_PROTOCOL.refer(interfaceClass, url);
            if (logger.isInfoEnabled()) {
                logger.info("Using injvm service " + interfaceClass.getName());
            }
        } else {
            urls.clear();
            if (url != null && url.length() > 0) { // user specified URL, could be peer-to-peer address, or register center's address.
                String[] us = SEMICOLON_SPLIT_PATTERN.split(url);
                if (us != null && us.length > 0) {
                    for (String u : us) {
                        URL url = URL.valueOf(u);
                        if (StringUtils.isEmpty(url.getPath())) {
                            url = url.setPath(interfaceName);
                        }
                        if (UrlUtils.isRegistry(url)) {
                            urls.add(url.addParameterAndEncoded(REFER_KEY, StringUtils.toQueryString(map)));
                        } else {
                            urls.add(ClusterUtils.mergeUrl(url, map));
                        }
                    }
                }
            } else { // assemble URL from register center's configuration
                // 如果protocols不是injvm检查注册中心
                if (!LOCAL_PROTOCOL.equalsIgnoreCase(getProtocol())) {
                    checkRegistry();
                    // url列表，将zookeeper://改成registry://
                    List<URL> us = ConfigValidationUtils.loadRegistries(this, false);
                    if (CollectionUtils.isNotEmpty(us)) {
                        for (URL u : us) {
                            // 监控URL
                            URL monitorUrl = ConfigValidationUtils.loadMonitor(this, u);
                            // 如果有监控配置放到map中
                            if (monitorUrl != null) {
                                map.put(MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                            }
                            // 根据map中的信息生成url，这里生成的结果是
                            /*
                            registry://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService?application=serialization-java-consumer&dubbo=2.0.2&pid=66793&qos.accept.foreign.ip=false&qos.enable=true&qos.port=33333&refer=application%3Dserialization-java-consumer%26check%3Dtrue%26dubbo%3D2.0.2%26init%3Dfalse%26interface%3Dorg.apache.dubbo.samples.serialization.api.DemoService%26methods%3DsayHello%26pid%3D66793%26qos.accept.foreign.ip%3Dfalse%26qos.enable%3Dtrue%26qos.port%3D33333%26register.ip%3D192.168.58.45%26release%3D2.7.7%26side%3Dconsumer%26sticky%3Dfalse%26timestamp%3D1636532992568&registry=zookeeper&release=2.7.7&timestamp=1636533091883
                            */
                            urls.add(u.addParameterAndEncoded(REFER_KEY, StringUtils.toQueryString(map)));
                        }
                    }
                    if (urls.isEmpty()) {
                        throw new IllegalStateException("No such any registry to reference " + interfaceName + " on the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", please config <dubbo:registry address=\"...\" /> to your spring config.");
                    }
                }
            }
            // 如果引用的URL就1个直接通过refer引用服务，这里和export相似，通过调用链实现的
            // - ProtocolListenerWrapper
            // - - ProtocolFilterWrapper
            // - - - RegistryProtocol
            if (urls.size() == 1) {
                // 通过RegistryProtocol#refer引用服务
                invoker = REF_PROTOCOL.refer(interfaceClass, urls.get(0));
            } else {
                // 如果是引用的服务多个，循环处理
                List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
                URL registryURL = null;
                for (URL url : urls) {
                    invokers.add(REF_PROTOCOL.refer(interfaceClass, url));
                    if (UrlUtils.isRegistry(url)) {
                        registryURL = url; // use last registry url
                    }
                }
                if (registryURL != null) { // registry url is available
                    // for multi-subscription scenario, use 'zone-aware' policy by default
                    // 集群处理
                    URL u = registryURL.addParameterIfAbsent(CLUSTER_KEY, ZoneAwareCluster.NAME);
                    // The invoker wrap relation would be like: ZoneAwareClusterInvoker(StaticDirectory) -> FailoverClusterInvoker(RegistryDirectory, routing happens here) -> Invoker
                    // 加入到集群中，这里包含集中集群处理模式，分别是：
                    invoker = CLUSTER.join(new StaticDirectory(u, invokers));
                } else { // not a registry url, must be direct invoke.
                    invoker = CLUSTER.join(new StaticDirectory(invokers));
                }
            }
        }
        // invoer不可用处理
        if (shouldCheck() && !invoker.isAvailable()) {
            invoker.destroy();
            throw new IllegalStateException("Failed to check the status of the service "
                    + interfaceName
                    + ". No provider available for the service "
                    + (group == null ? "" : group + "/")
                    + interfaceName +
                    (version == null ? "" : ":" + version)
                    + " from the url "
                    + invoker.getUrl()
                    + " to the consumer "
                    + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion());
        }
        if (logger.isInfoEnabled()) {
            logger.info("Refer dubbo service " + interfaceClass.getName() + " from url " + invoker.getUrl());
        }
        /**
         * @since 2.7.0
         * ServiceData存储，SPI机制，支持内存和远程两种方式
         */
        String metadata = map.get(METADATA_KEY);
        WritableMetadataService metadataService = WritableMetadataService.getExtension(metadata == null ? DEFAULT_METADATA_STORAGE_TYPE : metadata);
        if (metadataService != null) {
            URL consumerURL = new URL(CONSUMER_PROTOCOL, map.remove(REGISTER_IP_KEY), 0, map.get(INTERFACE_KEY), map);
            metadataService.publishServiceDefinition(consumerURL);
        }
        // create service proxy
        return (T) PROXY_FACTORY.getProxy(invoker, ProtocolUtils.isGeneric(generic));
    }
```

&nbsp; &nbsp; 创建代理服务会创建Invoker，在引用服务过程中，会判断协议是否为injvm，会根据协议做不同的处理，不是injvm协议会根据构造的配置信息（map）生成url并将url协议有zookeeper://改成registry://，然后通过`Protocol`接口`refer`方法引用服务，与发布服务相似，引用服务的过程也会包装方法的调用链，如下：

```
- ProtocolListenerWrapper
- - ProtocolFilterWrapper
- - - RegistryProtocol
```

&nbsp; &nbsp; 在refer的过程中会对一个服务端的引用和一个服务多个服务端的服务进行区分处理，对于有多个服务端的服务会进行集群处理（cluster），会讲invoker列表加入到集群中，在调用过程中会根据集群策略来选择不同的策略进行调用，集群策略实现也实现了SPI机制，


## RegistryProtocol#refer引用服务

```java
    @Override
    @SuppressWarnings("unchecked")
    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        // 注册中心url 将 registry:// 转成 zookeeper://
        url = getRegistryUrl(url);
        // 获取注册中心
        Registry registry = registryFactory.getRegistry(url);
        if (RegistryService.class.equals(type)) {
            return proxyFactory.getInvoker((T) registry, type, url);
        }

        // 从url中解析出引用服务信息
        Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(REFER_KEY));
        // 获取分组
        String group = qs.get(GROUP_KEY);
        // 如果设置分组了，那么使用MergeableCluster策略
        if (group != null && group.length() > 0) {
            if ((COMMA_SPLIT_PATTERN.split(group)).length > 1 || "*".equals(group)) {
                return doRefer(getMergeableCluster(), registry, type, url);
            }
        }
        return doRefer(cluster, registry, type, url);
    }
```

## RegistryProtocol#doRefer引用服务

```java
    private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
        // 获取集群目录Directory，代表Invoker的集合
        RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
        directory.setRegistry(registry);
        directory.setProtocol(protocol);
        // all attributes of REFER_KEY
        Map<String, String> parameters = new HashMap<String, String>(directory.getConsumerUrl().getParameters());
        URL subscribeUrl = new URL(CONSUMER_PROTOCOL, parameters.remove(REGISTER_IP_KEY), 0, type.getName(), parameters);
        if (directory.isShouldRegister()) {
            directory.setRegisteredConsumerUrl(subscribeUrl);
            // 注册消费者url
            registry.register(directory.getRegisteredConsumerUrl());
        }
        directory.buildRouterChain(subscribeUrl);
        // 向ZK发起订阅服务，并设置监听，这里比较复杂，最终会在ZookeeperRegistry#doSubscribe做服务订阅
        directory.subscribe(toSubscribeUrl(subscribeUrl));
        // 加入集群
        Invoker<T> invoker = cluster.join(directory);
        List<RegistryProtocolListener> listeners = findRegistryProtocolListeners(url);
        if (CollectionUtils.isEmpty(listeners)) {
            return invoker;
        }

        RegistryInvokerWrapper<T> registryInvokerWrapper = new RegistryInvokerWrapper<>(directory, cluster, invoker, subscribeUrl);
        for (RegistryProtocolListener listener : listeners) {
            listener.onRefer(this, registryInvokerWrapper);
        }
        return registryInvokerWrapper;
    }
```

## RegistryDirectory#subscribe订阅服务

&nbsp; &nbsp; `RegistryProtocol#doRefer`方法，directory.subscribe会按照下面的调用链进行处理，最后调用`ZookeeperRegistry#doSubscribe`方法向zk注册数据订阅接口，并设置监听。

```
- RegistryDirectory
- - ListenerRegistryWrapper
- - - FailbackRegistry
- - - - ZookeeperRegistry
```

```java
    @Override
    public void doSubscribe(final URL url, final NotifyListener listener) {
        try {
            if (ANY_VALUE.equals(url.getServiceInterface())) {
                String root = toRootPath();
                // 初始化监听
                ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.computeIfAbsent(url, k -> new ConcurrentHashMap<>());
                ChildListener zkListener = listeners.computeIfAbsent(listener, k -> (parentPath, currentChilds) -> {
                    for (String child : currentChilds) {
                        child = URL.decode(child);
                        if (!anyServices.contains(child)) {
                            anyServices.add(child);
                            subscribe(url.setPath(child).addParameters(INTERFACE_KEY, child,
                                    Constants.CHECK_KEY, String.valueOf(false)), k);
                        }
                    }
                });
                // 创建zk节点
                zkClient.create(root, false);
                List<String> services = zkClient.addChildListener(root, zkListener);
                if (CollectionUtils.isNotEmpty(services)) {
                    for (String service : services) {
                        service = URL.decode(service);
                        anyServices.add(service);
                        subscribe(url.setPath(service).addParameters(INTERFACE_KEY, service,
                                Constants.CHECK_KEY, String.valueOf(false)), listener);
                    }
                }
            } else {
                List<URL> urls = new ArrayList<>();
                for (String path : toCategoriesPath(url)) {
                    ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.computeIfAbsent(url, k -> new ConcurrentHashMap<>());
                    ChildListener zkListener = listeners.computeIfAbsent(listener, k -> (parentPath, currentChilds) -> ZookeeperRegistry.this.notify(url, k, toUrlsWithEmpty(url, parentPath, currentChilds)));
                    zkClient.create(path, false);
                    List<String> children = zkClient.addChildListener(path, zkListener);
                    if (children != null) {
                        urls.addAll(toUrlsWithEmpty(url, path, children));
                    }
                }
                // 
                notify(url, listener, urls);
            }
        } catch (Throwable e) {
            throw new RpcException("Failed to subscribe " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }

```

## DubboProtocol#protocolBindingRefer创建Invoker

&nbsp; &nbsp; refer服务创建Invoker时会调用该方法，该方法会通过`getClients`创建网络客户端，创建客户端是会判断客户端是否为共享链接，根据`connections`创建客户端`ExchangeClient`，然后通过`initClient`初始化客户端，初始化过程中会判断是否为延迟的客户端`LazyConnectExchangeClient`，不是延迟客户端，就会通过`connect`连接服务提供者，与服务提供者连接，具体建立连接流程不在这里说明，会在网络通信介绍。

```java
    @Override
    public <T> Invoker<T> protocolBindingRefer(Class<T> serviceType, URL url) throws RpcException {
        optimizeSerialization(url);

        // 创建RPC Invoker，通过getClients(url)创建网络客户端
        DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
        // 将invoker添加到invokers中
        invokers.add(invoker);

        return invoker;
    }

    // 创建客户端数组
    private ExchangeClient[] getClients(URL url) {
        // whether to share connection
        // 是否共享链接
        boolean useShareConnect = false;

        int connections = url.getParameter(CONNECTIONS_KEY, 0);
        List<ReferenceCountExchangeClient> shareClients = null;
        // if not configured, connection is shared, otherwise, one connection for one service
        if (connections == 0) {
            useShareConnect = true;

            /*
             * The xml configuration should have a higher priority than properties.
             * xml配置优先级高于properties配置
             */
            String shareConnectionsStr = url.getParameter(SHARE_CONNECTIONS_KEY, (String) null);
            connections = Integer.parseInt(StringUtils.isBlank(shareConnectionsStr) ? ConfigUtils.getProperty(SHARE_CONNECTIONS_KEY,
                    DEFAULT_SHARE_CONNECTIONS) : shareConnectionsStr);
            // 共享链接客户端
            shareClients = getSharedClient(url, connections);
        }
        // 创建ExchangeClient
        ExchangeClient[] clients = new ExchangeClient[connections];
        for (int i = 0; i < clients.length; i++) {
            if (useShareConnect) {
                // 从共享客户端获取
                clients[i] = shareClients.get(i);

            } else {
                // 初始化客户端
                clients[i] = initClient(url);
            }
        }

        return clients;
    }

    // 初始化客户端
    private ExchangeClient initClient(URL url) {

        // client type setting.
        String str = url.getParameter(CLIENT_KEY, url.getParameter(SERVER_KEY, DEFAULT_REMOTING_CLIENT));

        url = url.addParameter(CODEC_KEY, DubboCodec.NAME);
        // enable heartbeat by default
        url = url.addParameterIfAbsent(HEARTBEAT_KEY, String.valueOf(DEFAULT_HEARTBEAT));

        // BIO is not allowed since it has severe performance issue.
        if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
            throw new RpcException("Unsupported client type: " + str + "," +
                    " supported client type is " + StringUtils.join(ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions(), " "));
        }

        ExchangeClient client;
        try {
            // connection should be lazy
            if (url.getParameter(LAZY_CONNECT_KEY, false)) {
                // 延迟加载客户端
                client = new LazyConnectExchangeClient(url, requestHandler);

            } else {
                // 通过NettyTransporter connect创建客户端 与provider建立连接
                client = Exchangers.connect(url, requestHandler);
            }

        } catch (RemotingException e) {
            throw new RpcException("Fail to create remoting client for service(" + url + "): " + e.getMessage(), e);
        }

        return client;
    }
```

## 总结

&nbsp; &nbsp; Dubbo Consumer整个的启动，引用服务的流程大致就看完了，`ReferenceBean`实现了Spring的`FactoryBean`接口，在使用Spring上下文getBean时就会调用到`ReferenceBean`的getObject方法，这时就会通过创建代理（createProxy），然后通过`Protocol`的`refer`方法引用服务；

&nbsp; &nbsp; 引用服务的流程大致可以理解为，通过`DubboProtocol`的`refer`方法创建`DubboInvoker`调用`getClients`方法创建`ExchangeClient`，然后通过`initClient`方法初始化网络客户端，初始化客户端过程中会通过`Exchangers`的`connect`与服务提供者建立连接，后续就是网络客户端与服务端建立连接这里会和服务提供者相似，通过`doOpen`初始化网络客户端，然后调用`doConnect`建立连接。