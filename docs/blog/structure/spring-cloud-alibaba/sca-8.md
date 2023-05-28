# Spring Cloud Alibaba-OpenFeign与RestTemplate <!-- {docsify-ignore-all} -->



## 前言

​    本篇将介绍`Spring Cloud Alibaba`使用`OpenFeign`和`RestTemplate`进行RPC调用，并且将介绍两种RPC工具如何集成`Sentinel`进行系统保护。



## OpenFeign

#### OpenFeign介绍

​    OpenFeign是**一种声明式、模板化的HTTP客户端**。 在Spring Cloud中使用OpenFeign，可以做到使用HTTP请求访问远程服务，就像调用本地方法一样的，开发者完全感知不到这是在调用远程方法，更感知不到在访问HTTP请求。

#### OpenFeign使用

​    编写两个服务，一个provider服务`account-svc`一个consumer服务`order-svc`。

##### Provider代码

- pom依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.redick.cloud</groupId>
        <artifactId>ruuby-account</artifactId>
        <version>${revision}</version>
    </parent>
    
    <artifactId>ruuby-account-svc</artifactId>

    <packaging>jar</packaging>
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
    </dependencies>
</project>
```

- application.yml配置

```yaml
server:
  port: 8088
spring:
  application:
    name: account-svc
  cloud:
    nacos:
      username: "nacos"
      password: "nacos"
      discovery:
        # 服务注册中心地址
        server-addr: 127.0.0.1:8848
```

- 服务端代码

```java
@SpringBootApplication
@EnableDiscoveryClient
public class AccountApplication {

    public static void main( String[] args ) {
        SpringApplication.run(AccountApplication.class, args);
    }
    
    @GetMapping(path = "/echo/{string}")
    public String echo(@PathVariable String string) {
        return "Hello Nacos Discovery " + string;
    }
}
```

##### Consumer代码

- pom依赖

​    对比之前的demo，增加了`artifactId`为`spring-cloud-starter-openfeign`的`OpenFeign`依赖。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>io.redick.cloud</groupId>
        <artifactId>ruuby-account</artifactId>
        <version>${revision}</version>
    </parent>
    
    <artifactId>ruuby-account-svc</artifactId>

    <packaging>jar</packaging>
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-loadbalancer</artifactId>
        </dependency>
      	<!-- OpenFeign -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
    </dependencies>
</project>
```

- application.yml配置

```yaml
server:
  port: 8089
spring:
  application:
    name: order-svc
  cloud:
    nacos:
      username: "nacos"
      password: "nacos"
      discovery:
        # 服务注册中心地址
        server-addr: 127.0.0.1:8848
```

- 服务启动代码

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients(basePackages = {"io.redick.cloud.account"})
@AllArgsConstructor
public class OrderApplication {

    private final AccountService accountService;

    public static void main(String[] args) {
        SpringApplication.run(OrderApplication.class, args);
    }
  
      @GetMapping("/echo")
    public String echo(){
        return accountService.echo(appName);
    }
}
```

​    在`io.redick.cloud.account`包下创建`AccountService`，`@FeignClient`注解标注这是一个`OpenFeign`客户端，可以看到`OpenFeign`屏蔽了底层Http调用细节，可以像调用一个本地方法一样去进行远程调用，并且参数的序列化，反序列化也屏蔽了，对于开发人员来说使用起来更方便。

> 注：*OpenFeign+@PathVariable* *需要指定**value**否则会报错*

```java
@FeignClient(name = "account-svc", path = "/account")
public interface AccountService {

    @GetMapping(path = "/echo/{string}")
    String echo(@PathVariable(value="string") String string);
}
```

- 测试

```shell
➜ curl -X GET http://127.0.0.1:8089/order/feignEcho
Hello Nacos Discovery order-svc
```

- 配置使用Nacos LoadBalancer

​    demo中没有配置LoadBalancer，因为引用了`spring-cloud-starter-loadbalancer`依赖，所以默认情况下使用`spring-cloud-starter-loadbalancer`的`RoundRobinLoadBalancer`负载均衡器。Nacos Discovery提供了`NacosLoadBalancer`，通过如下配置开启`NacosLoadBalancer`。

```yaml
spring:
  cloud:
    loadbalancer:
      nacos:
        enabled: true
```

##### OpenFeign集成Sentinel

​    Spring Cloud Alibaba集成Sentinel引入`spring-cloud-starter-alibaba-sentinel`的starter，并且添加一些配置即可实现。

- pom依赖

```xml
				<!-- SpringCloud Alibaba Sentinel -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
        </dependency>

        <!-- Sentinel Datasource Nacos -->
        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-datasource-nacos</artifactId>
        </dependency>
```

- application.yml配置

```yaml
spring:
  cloud:
    # sentinel支持
    sentinel:
      # 取消控制台懒加载
      eager: true
      transport:
        dashboard: 127.0.0.1:8080
      # 动态数据源支持
      datasource:
        ds1:
          nacos:
            server-addr: 127.0.0.1:8848
            dataId: ${spring.application.name}-sentinel-${spring.profiles.active}.json
            namespace: 3ef5e608-6ee8-4881-8e50-ed47a5a04af2
            groupId: DEFAULT_GROUP
            data-type: json
            # 具体配置参考com.alibaba.cloud.sentinel.datasource.RuleType
            rule-type: flow
# feign支持sentinel
feign:
  sentinel:
    enabled: true
```

​    Nacos配置中心增加dataId为`${spring.application.name}-sentinel-${spring.profiles.active}.json`的配置文件，并配置流控规则，配置如下：

```json
[
    {
        "resource": "GET:http://account-svc/account/echo/{string}",
        "count": 0,
        "grade": 0,
        "limitApp": "default"
    }
]
```

- OpenFeign自定义降级

```
public class AccountCallback implements AccountService {

    @Override
    public String echo(String string) {
        return "Sentinel circuit breaker!";
    }
}

public class FeignConfiguration {

    @Bean
    public AccountCallback accountCallback() {
        return new AccountCallback();
    }
}
```

AccountService @FeignClient标签增加自定义配置和自定义降级。

```java
@FeignClient(name = "account-svc", path = "/account", fallback = AccountCallback.class,
        configuration = FeignConfiguration.class)
public interface AccountService {

    /**
     * 注：OpenFeign+@PathVariable 需要指定value否则会报错
     *
     * @param string String
     * @return String
     */
    @GetMapping(path = "/echo/{string}")
    String echo(@PathVariable(value="string") String string);
}
```

- 将Nacos上Sentinel配置count改为0，测试结果如下：

```shell
➜ curl -X GET http://127.0.0.1:8089/order/feignEcho
Sentinel circuit breaker!
```



## RestTemplate

​    大部分代码参考上面的代码，下面是RestTemplate相关的配置代码，`@LoadBalanced`开启负载均衡，`@SentinelRestTemplate`是Sentinel注解，括号里的配置都是自定义降级，流控的配置。

#### 创建RestTemplate Bean

```java
@Configuration
public class RestTemplateConfiguration {

    @Bean
    @LoadBalanced
    @SentinelRestTemplate(blockHandler = "blockHandle", blockHandlerClass = SentinelHandleUtil.class,
            fallback = "fallback", fallbackClass = SentinelHandleUtil.class)
    public RestTemplate restTemplate() {
        RestTemplate restTemplate = new RestTemplate();
        // 链路追踪
        restTemplate.setInterceptors(Stream.of(new TraceIdRestTemplateInterceptor()).collect(Collectors.toList()));
        return restTemplate;
    }
}
```

#### 自定义流控代码SentinelHandleUtil

```java
public class SentinelHandleUtil {

    public static SentinelClientHttpResponse blockHandle(HttpRequest request, byte[] body, ClientHttpRequestExecution execution
            , BlockException exception) {
        return new SentinelClientHttpResponse(JSON.toJSONString(R.fail(null, Constants.FLOW, "Sentinel flow block!")));
    }

    public static SentinelClientHttpResponse fallback(HttpRequest request, byte[] body, ClientHttpRequestExecution execution
            , BlockException exception) {
        return new SentinelClientHttpResponse(JSON.toJSONString(R.fail(null, Constants.FLOW, "Sentinel flow block!")));
    }
}
```

#### application.yml配置开启Sentinel

```yaml
# RestTemplate支持sentinel
resttemplate:
  sentinel:
    enabled: true
```

#### Nacos配置中心Sentinel配置文件增加配置

```shell
    {
        "resource": "GET:http://account-svc:8088/account/echo/order-svc",
        "count": 0,
        "grade": 0,
        "limitApp": "default"
    }
```

#### 测试结果

```shell
➜ curl -X GET http://127.0.0.1:8089/order/echo
{"code":455,"msg":"Sentinel flow block!"}
```



## 总结

​    致此Spring Cloud Alibaba集成OpenFeign，RestTemplate作为RPC，Spring Cloud LoadBalancer，NacosLoadBalancer作为负载均衡，Sentinel作为系统保护介绍完毕，相关底层逻辑感兴趣的可以自行Debug一下代码。