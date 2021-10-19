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

#### DubboBootstrap#start()

