# Dubbo服务启动-Dubbo Consumer引用服务 <!-- {docsify-ignore-all} -->



## Consumer消费者Demo示例

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

&nbsp; &nbsp; 在之前的章节中已经知道，Dubbo基于Spring自定义标签规范实现了自定义标签，通过自定义标签完成了bean的加载，并且通过实现监听Spring容器刷新完毕事件启动dubbo客户端。