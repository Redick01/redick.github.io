# Dubbo服务启动 <!-- {docsify-ignore-all} -->

- Dubbo Bean加载
- Dubbo消费者
- Dubbo生产者


### Dubbo Bean加载

&nbsp; &nbsp; 我们使用`dubbo`都是和`Spring`进行结合使用，并且常用的方式就是通过在`xml`中配置dubbo来实现的，我们结合这种方式来看一下，spring是如何加载dubbo的bean的，下面我们来看一下一个最基本的dubbo的配置，结合配置来分析：以下示例来自于官方例子

```xml
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xmlns="http://www.springframework.org/schema/beans" xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://dubbo.apache.org/schema/dubbo http://dubbo.apache.org/schema/dubbo/dubbo.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">
    <context:property-placeholder/>

    <dubbo:application name="demo-provider"/>

    <dubbo:registry address="zookeeper://${zookeeper.address:127.0.0.1}:2181"/>

    <bean id="demoService" class="org.apache.dubbo.samples.serialization.impl.DemoServiceImpl"/>

    <dubbo:service interface="org.apache.dubbo.samples.serialization.api.DemoService" ref="demoService"
                   serialization="java"/>

</beans>
```

&nbsp; &nbsp; 在配置文件中配置了注册中心`<dubbo:registry`，服务的生产者`<dubbo:service`等信息，使用的的标签是dubbo的自定义标签，要继续了解dubbo自定义标签要先了解一下开发一个自定义标签的流程，下面是开发流程：

- 编写一个Java Bean；
- 编写XSD文件；
- 编写标签解析器，实现BeanDefinitionParser接口；
- 编写注册标签解析器的NamespaceHandlerSupport，继承NamespaceHandlerSupport，重写init方法，注册自定义的标签解析器；
- 编写spring.handlers和spring.schemas文件；

&nbsp; &nbsp; 开发完自定义的标签后就可以在spring配置文件中使用了，通过`dubbo`的`spring.handlers`文件找到注册标签解析器的类`com.alibaba.dubbo.config.spring.schema.DubboNamespaceHandler`。通过下面代码可以看到，dubbo注册了10个自定义标签，除了`annotation`标签，其他的标签均使用`DubboBeanDefinitionParser`解析器进行解析，解析dubbo自定义标签后就是Spring的创建对象和属性赋值。

```java
public class DubboNamespaceHandler extends NamespaceHandlerSupport {

    static {
        Version.checkDuplicate(DubboNamespaceHandler.class);
    }

    @Override
    public void init() {
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
    }

}
```

## Dubbo消费者

&nbsp; &nbsp; 在上面的dubbo bean加载章节中我们知道dubbo通过Spring自定义标签，在程序启动时创建了实例并进行了实例赋值，接下来我们看一下Dubbo消费者是如何启动并进行远程调用的

### Dubbo消费者启动

#### Dubbo客户端启动监听DubboBootstrapApplicationListener

> 上一章节说到了通过spring自定义标签已经创建了comsumer的bean，consumer的bean创建成功了，我们来分析下消费者启动的过程，dubbo实现了Spring的监听接口`ApplicationListener`接口，实现监听接口是为了当Spring容器refresh完成后能够接到容器刷新完成的事件。当Spring容器刷新完后通过时间监听执行dubbo客户端启动，代码如下：

```java

public class DubboBootstrapApplicationListener extends OneTimeExecutionApplicationContextEventListener
        implements Ordered {

    /**
     * The bean name of {@link DubboBootstrapApplicationListener}
     *
     * @since 2.7.6
     */
    public static final String BEAN_NAME = "dubboBootstrapApplicationListener";

    private final DubboBootstrap dubboBootstrap;

    public DubboBootstrapApplicationListener() {
        // 获取DubboBootstrap对象，会通过SPI加载Environment，ConfigManager
        this.dubboBootstrap = DubboBootstrap.getInstance();
    }

    @Override
    public void onApplicationContextEvent(ApplicationContextEvent event) {
        // 处理ContextRefreshedEvent时间，该事件是Spring容器刷新完毕时间
        if (event instanceof ContextRefreshedEvent) {
            onContextRefreshedEvent((ContextRefreshedEvent) event);
        } else if (event instanceof ContextClosedEvent) {
            // 处理ContextClosedEvent时间，spring上下文关闭事件
            onContextClosedEvent((ContextClosedEvent) event);
        }
    }

    // spring上下文刷新完毕事件
    private void onContextRefreshedEvent(ContextRefreshedEvent event) {
        dubboBootstrap.start();
    }

    // Spring上下文关闭事件
    private void onContextClosedEvent(ContextClosedEvent event) {
        dubboBootstrap.stop();
    }

    @Override
    public int getOrder() {
        return LOWEST_PRECEDENCE;
    }
}

abstract class OneTimeExecutionApplicationContextEventListener implements ApplicationListener, ApplicationContextAware {

    private ApplicationContext applicationContext;

    public final void onApplicationEvent(ApplicationEvent event) {
        if (isOriginalEventSource(event) && event instanceof ApplicationContextEvent) {
            onApplicationContextEvent((ApplicationContextEvent) event);
        }
    }

    /**
     * The subclass overrides this method to handle {@link ApplicationContextEvent}
     *
     * @param event {@link ApplicationContextEvent}
     */
    protected abstract void onApplicationContextEvent(ApplicationContextEvent event);

    /**
     * Is original {@link ApplicationContext} as the event source
     *
     * @param event {@link ApplicationEvent}
     * @return
     */
    private boolean isOriginalEventSource(ApplicationEvent event) {
        return (applicationContext == null) // Current ApplicationListener is not a Spring Bean, just was added
                // into Spring's ConfigurableApplicationContext
                || Objects.equals(applicationContext, event.getSource());
    }

    @Override
    public final void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    public ApplicationContext getApplicationContext() {
        return applicationContext;
    }
}
```

> Spring容器刷新完毕发布事件代码如下：

```java

	/**
	 * Finish the refresh of this context, invoking the LifecycleProcessor's
	 * onRefresh() method and publishing the
	 * {@link org.springframework.context.event.ContextRefreshedEvent}.
	 */
	protected void finishRefresh() {
		// Initialize lifecycle processor for this context.
		initLifecycleProcessor();

		// Propagate refresh to lifecycle processor first.
		getLifecycleProcessor().onRefresh();

		// Publish the final event.发布ContextRefreshedEvent事件
		publishEvent(new ContextRefreshedEvent(this));

		// Participate in LiveBeansView MBean, if active.
		LiveBeansView.registerApplicationContext(this);
	}
```

#### DubboBootstrap#start()dubbo启动

```java
    public DubboBootstrap start() {
        // 是否启动过
        if (started.compareAndSet(false, true)) {
            ready.set(false);
            // 初始化
            initialize();
            if (logger.isInfoEnabled()) {
                logger.info(NAME + " is starting...");
            }
            // 1. 发布dubbo服务
            exportServices();

            // Not only provider register，原数据发布一次
            if (!isOnlyRegisterProvider() || hasExportedServices()) {
                // 2. 发布元数据
                exportMetadataService();
                //3. Register the local ServiceInstance if required
                registerServiceInstance();
            }

            referServices();
            if (asyncExportingFutures.size() > 0) {
                new Thread(() -> {
                    try {
                        this.awaitFinish();
                    } catch (Exception e) {
                        logger.warn(NAME + " exportAsync occurred an exception.");
                    }
                    ready.set(true);
                    if (logger.isInfoEnabled()) {
                        logger.info(NAME + " is ready.");
                    }
                }).start();
            } else {
                ready.set(true);
                if (logger.isInfoEnabled()) {
                    logger.info(NAME + " is ready.");
                }
            }
            if (logger.isInfoEnabled()) {
                logger.info(NAME + " has started.");
            }
        }
        return this;
    }
```


#### 初始化initialize();

```java
    private void initialize() {
        // 1. 校验是否初始化过
        if (!initialized.compareAndSet(false, true)) {
            return;
        }
        // 2. 初始化扩展框架
        ApplicationModel.initFrameworkExts();
        // 3. 启动配置中心 
        startConfigCenter();
        // 4. 是否需要注册中心作为配置中心
        useRegistryAsConfigCenterIfNecessary();
        // 5. 加载远程配置
        loadRemoteConfigs();
        // 6. 检查配置
        checkGlobalConfigs();
        // 7. 初始化MetaService
        initMetadataService();
        // 8. 时间监听初始化
        initEventListener();

        if (logger.isInfoEnabled()) {
            logger.info(NAME + " has been initialized!");
        }
    }
```

##### ApplicationModel.initFrameworkExts();

> dubbo通过SPI机制加载三个类，分别是：Environment，ConfigManager，ServiceRepostitory；Environment是Dubbo的环境信息类，会加载配置信息，从配置文件系统参数，外部配置中心等加载配置信息。

```java
    public static void initFrameworkExts() {
        Set<FrameworkExt> exts = ExtensionLoader.getExtensionLoader(FrameworkExt.class).getSupportedExtensionInstances();
        for (FrameworkExt ext : exts) {
            ext.initialize();
        }
    }
```

> Environment会加载五种配置，PropertiesConfiguration从properties中加载配置，SystemConfiguration系统配置，EnvironmentConfiguration加载系统环境配置信息，externalConfiguration和appExternalConfiguration这两个成员变量的对象是同一种类InmemoryConfiguration，initialize()方法会获取所有配置中心的配置，将配置信息放到externalConfiguration和appExternalConfiguration，initialize()方法会在加载SPI后调用，就是上一段代码的ext.initialize();

```java
    private final PropertiesConfiguration propertiesConfiguration;
    private final SystemConfiguration systemConfiguration;
    private final EnvironmentConfiguration environmentConfiguration;
    private final InmemoryConfiguration externalConfiguration;
    private final InmemoryConfiguration appExternalConfiguration;

    
    private Map<String, String> externalConfigurationMap = new HashMap<>();
    private Map<String, String> appExternalConfigurationMap = new HashMap<>();

    

    public Environment() {
        this.propertiesConfiguration = new PropertiesConfiguration();
        this.systemConfiguration = new SystemConfiguration();
        this.environmentConfiguration = new EnvironmentConfiguration();
        this.externalConfiguration = new InmemoryConfiguration();
        this.appExternalConfiguration = new InmemoryConfiguration();
    }

    @Override
    public void initialize() throws IllegalStateException {
        ConfigManager configManager = ApplicationModel.getConfigManager();
        // 获取所有配置
        Optional<Collection<ConfigCenterConfig>> defaultConfigs = configManager.getDefaultConfigCenter();
        defaultConfigs.ifPresent(configs -> {
            // 将配置信息放到externalConfiguration和appExternalConfiguration
            for (ConfigCenterConfig config : configs) {
                this.setExternalConfigMap(config.getExternalConfiguration());
                this.setAppExternalConfigMap(config.getAppExternalConfiguration());
            }
        });

        this.externalConfiguration.setProperties(externalConfigurationMap);
        this.appExternalConfiguration.setProperties(appExternalConfigurationMap);
    }
```

> Environment会根据优先级将配置加载到CompositeConfiguration中，代码如下

```java
    public synchronized CompositeConfiguration getPrefixedConfiguration(AbstractConfig config) {
        CompositeConfiguration prefixedConfiguration = new CompositeConfiguration(config.getPrefix(), config.getId());
        Configuration configuration = new ConfigConfigurationAdapter(config);
        if (this.isConfigCenterFirst()) {
            // The sequence would be: SystemConfiguration -> AppExternalConfiguration -> ExternalConfiguration -> AbstractConfig -> PropertiesConfiguration
            // Config center has the highest priority
            prefixedConfiguration.addConfiguration(systemConfiguration);
            prefixedConfiguration.addConfiguration(environmentConfiguration);
            prefixedConfiguration.addConfiguration(appExternalConfiguration);
            prefixedConfiguration.addConfiguration(externalConfiguration);
            prefixedConfiguration.addConfiguration(configuration);
            prefixedConfiguration.addConfiguration(propertiesConfiguration);
        } else {
            // The sequence would be: SystemConfiguration -> AbstractConfig -> AppExternalConfiguration -> ExternalConfiguration -> PropertiesConfiguration
            // Config center has the highest priority
            prefixedConfiguration.addConfiguration(systemConfiguration);
            prefixedConfiguration.addConfiguration(environmentConfiguration);
            prefixedConfiguration.addConfiguration(configuration);
            prefixedConfiguration.addConfiguration(appExternalConfiguration);
            prefixedConfiguration.addConfiguration(externalConfiguration);
            prefixedConfiguration.addConfiguration(propertiesConfiguration);
        }
        return prefixedConfiguration;
    }
```

##### startConfigCenter();

> startConfigCenter();启动配置中心

如果不单独配置配置中心不会加载配置中心

```java
    private void startConfigCenter() {
        // 配置中心配置
        Collection<ConfigCenterConfig> configCenters = configManager.getConfigCenters();

        // configCenters是空的
        if (CollectionUtils.isEmpty(configCenters)) {
            ConfigCenterConfig configCenterConfig = new ConfigCenterConfig();
            // 从Environment刷新配置
            configCenterConfig.refresh();
            // 检查配置，配置中的address和protocol不能是空的并且address要包含"//"
            if (configCenterConfig.isValid()) {
                configManager.addConfigCenter(configCenterConfig);
                configCenters = configManager.getConfigCenters();
            }
        } else {
            for (ConfigCenterConfig configCenterConfig : configCenters) {
                configCenterConfig.refresh();
                ConfigValidationUtils.validateConfigCenterConfig(configCenterConfig);
            }
        }
        // 配置中心配置不为空，DynamicConfig处理，支持Apollo，Etcd，zookeeper，consul，nacos等动态配置
        if (CollectionUtils.isNotEmpty(configCenters)) {
            CompositeDynamicConfiguration compositeDynamicConfiguration = new CompositeDynamicConfiguration();
            for (ConfigCenterConfig configCenter : configCenters) {
                // 根据配置中心配置，创建动态配置，prepareEnvironment负责配置中心准备，比如zk就会创建基于zk的动态配置，会创建zk客户端和配置监听
                compositeDynamicConfiguration.addConfiguration(prepareEnvironment(configCenter));
            }
            environment.setDynamicConfiguration(compositeDynamicConfiguration);
        }
        // 刷新所有配置，ApplicationConfig，MonitorConfig，ModuleConfig，ProtocolConfig，RegistryConfig，ProviderConfig，ConsumerConfig
        configManager.refreshAll();
    }
```

##### useRegistryAsConfigCenterIfNecessary();是否需要注册中心作为配置中心

```java
    private void useRegistryAsConfigCenterIfNecessary() {
        // we use the loading status of DynamicConfiguration to decide whether ConfigCenter has been initiated.
        // DynamicConfiguration加载过，ConfigCenter已经初始化了，不用将注册中心作为配置中心了
        if (environment.getDynamicConfiguration().isPresent()) {
            return;
        }
        // 已经加载过配置中心了，就不用将注册中心作为配置中心了
        if (CollectionUtils.isNotEmpty(configManager.getConfigCenters())) {
            return;
        }
        // 将注册中心配置加到配置中心配置中
        configManager.getDefaultRegistries().stream()
                .filter(registryConfig -> registryConfig.getUseAsConfigCenter() == null || registryConfig.getUseAsConfigCenter())
                .forEach(registryConfig -> {
                    String protocol = registryConfig.getProtocol();
                    String id = "config-center-" + protocol + "-" + registryConfig.getPort();
                    ConfigCenterConfig cc = new ConfigCenterConfig();
                    cc.setId(id);
                    if (cc.getParameters() == null) {
                        cc.setParameters(new HashMap<>());
                    }
                    if (registryConfig.getParameters() != null) {
                        cc.getParameters().putAll(registryConfig.getParameters());
                    }
                    cc.getParameters().put(CLIENT_KEY, registryConfig.getClient());
                    cc.setProtocol(registryConfig.getProtocol());
                    cc.setPort(registryConfig.getPort());
                    cc.setAddress(registryConfig.getAddress());
                    cc.setNamespace(registryConfig.getGroup());
                    cc.setUsername(registryConfig.getUsername());
                    cc.setPassword(registryConfig.getPassword());
                    if (registryConfig.getTimeout() != null) {
                        cc.setTimeout(registryConfig.getTimeout().longValue());
                    }
                    cc.setHighestPriority(false);
                    configManager.addConfigCenter(cc);
                });
        // 再次startConfigCenter
        startConfigCenter();
    }
```

##### checkGlobalConfigs();检查全置

&nbsp; &nbsp; 校验各种配置，不展开说了

```java
    private void checkGlobalConfigs() {
        // check Application
        ConfigValidationUtils.validateApplicationConfig(getApplication());

        // 检查原数据配置
        Collection<MetadataReportConfig> metadatas = configManager.getMetadataConfigs();
        if (CollectionUtils.isEmpty(metadatas)) {
            MetadataReportConfig metadataReportConfig = new MetadataReportConfig();
            metadataReportConfig.refresh();
            if (metadataReportConfig.isValid()) {
                configManager.addMetadataReport(metadataReportConfig);
                metadatas = configManager.getMetadataConfigs();
            }
        }
        if (CollectionUtils.isNotEmpty(metadatas)) {
            for (MetadataReportConfig metadataReportConfig : metadatas) {
                metadataReportConfig.refresh();
                ConfigValidationUtils.validateMetadataConfig(metadataReportConfig);
            }
        }

        // check Provider
        Collection<ProviderConfig> providers = configManager.getProviders();
        if (CollectionUtils.isEmpty(providers)) {
            configManager.getDefaultProvider().orElseGet(() -> {
                ProviderConfig providerConfig = new ProviderConfig();
                configManager.addProvider(providerConfig);
                providerConfig.refresh();
                return providerConfig;
            });
        }
        for (ProviderConfig providerConfig : configManager.getProviders()) {
            ConfigValidationUtils.validateProviderConfig(providerConfig);
        }
        // check Consumer
        Collection<ConsumerConfig> consumers = configManager.getConsumers();
        if (CollectionUtils.isEmpty(consumers)) {
            configManager.getDefaultConsumer().orElseGet(() -> {
                ConsumerConfig consumerConfig = new ConsumerConfig();
                configManager.addConsumer(consumerConfig);
                consumerConfig.refresh();
                return consumerConfig;
            });
        }
        for (ConsumerConfig consumerConfig : configManager.getConsumers()) {
            ConfigValidationUtils.validateConsumerConfig(consumerConfig);
        }

        // check Monitor
        ConfigValidationUtils.validateMonitorConfig(getMonitor());
        // check Metrics
        ConfigValidationUtils.validateMetricsConfig(getMetrics());
        // check Module
        ConfigValidationUtils.validateModuleConfig(getModule());
        // check Ssl
        ConfigValidationUtils.validateSslConfig(getSsl());
    }
```