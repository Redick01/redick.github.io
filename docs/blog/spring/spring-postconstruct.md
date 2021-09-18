# Spring框架@PostConstruct注解详解 <!-- {docsify-ignore-all} -->

- 前言


## 前言

&nbsp; &nbsp; 本文简单来看一下Spring框架@PostConstruct注解的原理。

## 业务背景


&nbsp; &nbsp; 在某些业务场景下我们需要程序在启动的时候就加载某些数据，比如，在程序启动的过程中需要从数据库中加载数据并缓存到程序的内存中。

### 通过依赖查找实现

&nbsp; &nbsp; 针对这个场景最直观的做法是我在容器的启动过程当中，通过`依赖查找`的方式获取到mapper，然后从数据库中获取数据并缓存到内存中。实现方式如下：

```java

@Slf4j
public class MainClass {

    public static ClassPathXmlApplicationContext context = null;

    private static CountDownLatch shutdownLatch = new CountDownLatch(1);

    public static void main(String[] args) throws Exception {
        // 加载spring上下文
        context = new ClassPathXmlApplicationContext(new String[]{"spring-config.xml"});
        context.start();
        // 从数据库获取数据并缓存到内存
        ItpcsConfigMapper itpcsConfigMapper = (ItpcsConfigMapper) context.getBean("itpcsConfigMapper");
        List<ItpcsConfig> RuleResultSet = itpcsConfigMapper.selectAll();
        RuleResultSet.forEach(itpcsConfig -> PropertyMap.add(itpcsConfig.getName(), itpcsConfig.getValue()));
        
        //注册Spring关闭钩子，在停止应用时关闭Spring上下文防止资源不及时关闭
        context.registerShutdownHook();
        log.info(LogUtil.marker(), "System already started.");
        shutdownLatch.await();
    }
}
```