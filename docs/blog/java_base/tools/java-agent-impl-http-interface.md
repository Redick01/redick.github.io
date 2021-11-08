# 基于JavaAgent实现发布http接口


## 需求

&nbsp; &nbsp; 公司运维系统想要监控服务是否正常启动，这些服务是k8s部署的，运维人员的要求业务服务提供一个http接口用于监控服务健康监测，要求所有的接口请求的URL，参数等都是相同的，这么做的目的是不需要通过规范来约束开发人员去开一个服务健康监测的接口。

&nbsp; &nbsp; 使用服务接口来检测服务我觉得相比较监控进程启动，端口监听等方式更准确一些。所以，为了满足运维同学的要求，起初想到的方案是提供一个jar，专门集成到项目中用于发布监控接口，但是想了一下，这么做需要涉及到服务的改造，我理想的方式是对应用无侵入的方式实现。


## 初步方案

&nbsp; &nbsp; 说到对应用无入侵，首先想到的就是`javaagent`技术，此前使用该技术实现了无入侵增强程序日志的工具，所以对使用`javaagent`已经没有问题，此时需要考虑的是如何发布接口了。

> 基础技术 JavaAgent

> 支持的技术 SpringBoot和DubboX发布的rest服务

&nbsp; &nbsp; 公司服务大致分为两类，一个是使用`springboot`发布的Spring MVC rest接口，另一种是基于`DubboX`发布的rest接口，因为公司在向服务网格转，所以按要求是去dubbo化的，没办法还是有其他小组由于一些其他原因没有或者说短期内不想进行服务改造的项目，这些项目比较老，不是springboot的，是使用spring+DubboX发布的rest服务。所以这个agent要至少能支持这两种技术。

> 支持SpringBoot

&nbsp; &nbsp; 想要支持SpringBoot很简单，因为SpringBoot支持自动装配，所以，我要写一个`spring.factories`来进行自动装配。

> 支持DubboX

&nbsp; &nbsp; 业务系统是传统spring+DubboX实现的，并不支持自动装配，这是个问题点，还有个问题点就是如何也发布一个DubboX的rest接口，这两个问题实际上就需要对SpringBean生命周期和Dubbo接口发布的流程有一定的了解了，这个一会儿再说。


## 技术实现

### pom文件依赖

```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <version>2.3.6.RELEASE</version>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-autoconfigure</artifactId>
            <version>2.3.6.RELEASE</version>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <version>2.3.6.RELEASE</version>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.70</version>
        </dependency>
        <dependency>
            <groupId>org.apache.zookeeper</groupId>
            <artifactId>zookeeper</artifactId>
            <version>3.4.6</version>
            <exclusions>
                <exclusion>
                    <groupId>log4j</groupId>
                    <artifactId>log4j</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>com.101tec</groupId>
            <artifactId>zkclient</artifactId>
            <version>0.7</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>dubbo</artifactId>
            <version>2.8.4</version>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework</groupId>
                    <artifactId>spring</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>javax.ws.rs</groupId>
            <artifactId>javax.ws.rs-api</artifactId>
            <version>2.0.1</version>
        </dependency>
    </dependencies>
```

### 实现一个JavaAgent

&nbsp; &nbsp; 实现一个JavaAgent很容易，以下三步就可以了，这里不细说了。

- **定义JavaAgent入口**

```java
public class PreAgent {

    public static void premain(String args, Instrumentation inst) {
        System.out.println("输入参数：" + args);
        // 通过参数控制，发布的接口是DubboX还是SpringMVC
        Args.EXPORT_DUBBOX = args;
    }
}
```

- **Maven打包配置**

```xml
        <plugins>
            <plugin>
                <artifactId>maven-deploy-plugin</artifactId>
                <configuration>
                    <skip>true</skip>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>1.4</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <keepDependenciesWithProvidedScope>true</keepDependenciesWithProvidedScope>
                            <promoteTransitiveDependencies>false</promoteTransitiveDependencies>
                            <createDependencyReducedPom>true</createDependencyReducedPom>
                            <minimizeJar>false</minimizeJar>
                            <createSourcesJar>true</createSourcesJar>

                            <transformers>
                                <transformer
                                        implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <manifestEntries>
                                        <Premain-Class>com.ruubypay.agent.PreAgent</Premain-Class>
                                    </manifestEntries>
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
```

- **MANIFEST.MF编写**

注：该文件在`resource/META-INF/`目录下

```
Manifest-Version: 1.0
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Premain-Class: com.ruubypay.agent.PreAgent
```

### 支持SpringBoot发布的Http接口

- **编写Controller**

&nbsp; &nbsp; 接口很简单就发布一个`get`接口，响应`pong`即可。

```java
@RestController
public class PingServiceController {

    @GetMapping(value = "/agentServer/ping")
    public String ping() {
        return "pong";
    }
}
```

- **创建spring.factories**

&nbsp; &nbsp; 通过这个配置文件可以实现SpringBoot自动装配，这里不细说SpringBoot自动装配的原理了，该文件的配置内容就是要自动装配的`Bean`的全路径，代码如下：

```factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.ruubypay.config.WebConfiguration
```

- **WebConfiguration配置类**

&nbsp; &nbsp; 这个配置配置类很简单，`@Configuration`声明这是个配置类，`@ComponentScan`扫描包。

```java
@Configuration
@ComponentScan(value = "com.ruubypay")
public class WebConfiguration {
}
```

### 支持DubboX发布的rest接口


- **定义API**

&nbsp; &nbsp; 使用的是`DubboX`发布`rest接口`需要`javax.ws.rs`包的注解，`@Produces({ContentType.APPLICATION_JSON_UTF_8})`声明序列化方式，`@Path`rest接口的路径，`@GET`声明为get接口。

```java
@Produces({ContentType.APPLICATION_JSON_UTF_8})
@Path("/agentServer")
public interface IPingService {

    /**
     * ping接口
     * @return
     */
    @GET
    @Path("/ping")
    String ping();
}
```

- **编写API实现类**

```java
@Component("IPingService")
public class IPingServiceImpl implements IPingService {

    @Override
    public String ping() {
        return "pong";
    }
}
```

- **实现发布Dubbo接口**

&nbsp; &nbsp; 如何实现发布接口是实现的难点；首先程序并不支持自动装配了，我们就要考虑如何获取到`Spring`上下文，如果能够注册`Bean`到`Spring容器`中，如何触发发布`Dubbo接口`等问题。

> Spring上下文获取及注册Bean到Spring容器中

&nbsp; &nbsp; 触发Bean注册，获取Spring上下文我们通过Spring的Aware接口可以实现，我这里使用的是`ApplicationContextAware`；注册`Bean`到`Spring容器`中可以使用`BeanDefinition`先创建Bean然后使用`DefaultListableBeanFactory`的`registerBeanDefinition`将`BeanDefinition`注册到Spring上下文中。

```java
@Component
public class AgentAware implements ApplicationContextAware {

    private static final String DUBBOX = "1";


    private ApplicationContext applicationContext;



    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
        // 如果不是DubboX，不用发布接口
        if (DUBBOX.equals(Args.EXPORT_DUBBOX)) {
            // 注册配置Bean WebConfiguration
            webConfiguration();
            // 发布DubboX接口
            exportDubboxService();
        }
    }

    public void webConfiguration() {
        System.out.println("创建WebConfiguration的bean");
        ConfigurableApplicationContext configurableApplicationContext = (ConfigurableApplicationContext) applicationContext;
        DefaultListableBeanFactory listableBeanFactory = (DefaultListableBeanFactory) configurableApplicationContext.getAutowireCapableBeanFactory();
        // 创建WebConfiguration的bean
        BeanDefinition webConfigurationBeanDefinition = new RootBeanDefinition(WebConfiguration.class);
        // 注册到集合beanFactory中
        System.out.println("注册到集合beanFactory中");
        listableBeanFactory.registerBeanDefinition(WebConfiguration.class.getName(), webConfigurationBeanDefinition);
    }

}
```

> 发布Dubbo接口

&nbsp; &nbsp; 通过`ApplicationContextAware`我们已经能够获取Spring上下文了，也就是说应用程序的Dubbo注册中心，发布接口协议，Dubbo Application等配置都已经存在Spring容器中了，我们只要拿过来使用即可，拿过来使用没问题，我们接下来就需要考虑，如何发布接口，这需要对Dubbo服务发布的流程有一定的了解，这里我不细说了，感兴趣的可以自己了解下，或者看我以前发布的文章；

&nbsp; &nbsp; 首先Dubbo接口的Provider端的核心Bean是`com.alibaba.dubbo.config.spring.ServiceBean`，使用Spring配置文件中的标签`<dubbo:service`标签生成的Bean就是`ServiceBean`，所以，这里我们只需要创建`ServiceBean`对象并且初始化对象中的必要数据，然后调用`ServiceBean#export()`方法就可以发布Dubbo服务了。

&nbsp; &nbsp; 这里需要的对象直接通过依赖查找的方式从Spring容器获取就可以了 `ApplicationConfig`,`ProtocolConfig`,`RegistryConfig`,`IPingService`。


```java
    public void exportDubboxService() {
        try {
            System.out.println("开始发布dubbo接口");
            // 获取ApplicationConfig
            ApplicationConfig applicationConfig = applicationContext.getBean(ApplicationConfig.class);
            // 获取ProtocolConfig
            ProtocolConfig protocolConfig = applicationContext.getBean(ProtocolConfig.class);
            // 获取RegistryConfig
            RegistryConfig registryConfig = applicationContext.getBean(RegistryConfig.class);
            // 获取IPingService接口
            IPingService iPingService = applicationContext.getBean(IPingService.class);
            // 创建ServiceBean
            ServiceBean<IPingService> serviceBean = new ServiceBean<>();
            serviceBean.setApplicationContext(applicationContext);
            serviceBean.setInterface("com.ruubypay.api.IPingService");
            serviceBean.setApplication(applicationConfig);
            serviceBean.setProtocol(protocolConfig);
            serviceBean.setRegistry(registryConfig);
            serviceBean.setRef(iPingService);
            serviceBean.setTimeout(12000);
            serviceBean.setVersion("1.0.0");
            serviceBean.setOwner("rubby");
            // 发布dubbo接口
            serviceBean.export();
            System.out.println("dubbo接口发布完毕");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```


### 使用方式

1. DubboX： java -javaagent:ruubypay-ping-agent.jar=1 -jar 服务jar包
2. springboot的http接口：java -javaagent:ruubypay-ping-agent.jar -jar 服务jar包


## 总结

&nbsp; &nbsp; 这个工具实现起来不复杂，总也就六个类和一个接口，但其实实现其能力所涉及的支持还是比较考验对框架的理解的，比如Spring生命周期，DubboX发布接口的流程以及实现一个最简单的JavaAgent的方式。

&nbsp; &nbsp; 另外也欢迎指正错误，如果有更好的实现方式也欢迎交流。
