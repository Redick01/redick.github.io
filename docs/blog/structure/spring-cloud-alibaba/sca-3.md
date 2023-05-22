## Spring Cloud Gateway-基础搭建 <!-- {docsify-ignore-all} -->



## 项目目录

```shell
ruuby-gateway
```



## POM依赖

​    Spring Boot，Spring Cloud，Discovery，Config等基础依赖在父pom中已经配置，参考[依赖版本管理及项目结构](../../../blog/structure/spring-cloud-alibaba/sca-2.md)

```xml
		<dependencies>
        <!-- SpringCloud Gateway -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>
        
        <!-- SpringCloud Loadbalancer -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-loadbalancer</artifactId>
        </dependency>

        <dependency>
            <groupId>io.redick.cloud</groupId>
            <artifactId>ruuby-common</artifactId>
            <version>${revision}</version>
        </dependency>
    </dependencies>
```

## 配置文件

- 本地bootstrap.yml配置文件

​    本地的配置文件指定网关服务的配置中心为nacos，指定共享配置文件`${spring.application.name}-${spring.profiles.active}.${spring.cloud.nacos.config.file-extension}`，其规则是服务名-active profile-spring.cloud.nacos.config.file-extension，所以此服务配置中心生效的配置文件为`ruuby-gateway-dev.yml`并且允许配置热更新`refresh-enabled: true`。

```yaml
spring:
  application:
    name: ruuby-gateway
  profiles:
    active: dev
  cloud:
    nacos:
      username: "nacos"
      password: "nacos"
      config:
        server-addr: 127.0.0.1:8848
        # namespace id
        namespace: 3ef5e608-6ee8-4881-8e50-ed47a5a04af2
        # 阿里云平台ak，sk
        # access-key:
        # secret-key:
        # 配置文件格式
        file-extension: yml
        shared-configs:
          - ${spring.application.name}-${spring.profiles.active}.${spring.cloud.nacos.config.file-extension}
        refresh-enabled: true
```

- Nacos配置中心配置文件ruuby-gateway-dev.yml

​    `spring.cloud.nacos.dicovery`配置服务发现注册中心；

​    `spring.cloud.gateway`配置网关路由信息，`predicates`配置根据`Path`进行路由，`id`进行服务发现，`uri: lb://account-svc`通过负载均衡请求后端。

```yaml
server:
  port: 8081
spring:
  application:
    name: ruuby-gateway
  profiles:
    active: dev
  cloud:
    nacos:
      username: "nacos"
      password: "nacos"
      discovery:
        # 服务注册中心地址
        server-addr: 127.0.0.1:8848
        # 阿里云平台ak，sk
        # access-key:
        # secret-key:
        namespace: 3ef5e608-6ee8-4881-8e50-ed47a5a04af2
      config:
        server-addr: 127.0.0.1:8848
        # 阿里云平台ak，sk
        # access-key:
        # secret-key:
        # 配置文件格式
        file-extension: yml
        shared-configs:
          - ${spring.application.name}-${spring.profiles.active}.${spring.cloud.nacos.config.file-extension}
        namespace: 3ef5e608-6ee8-4881-8e50-ed47a5a04af2
        group: DEFAULT-GROUP
    gateway:
      routes:
        - id: account-svc
          uri: lb://account-svc
          predicates:
            - Path=/gateway/account/**
          filters:
            - StripPrefix=1
            # url黑名单 这是一个局部过滤器
            # - name: BlackListUrlFilter
            #   args: 
            #     blackListUrl:
            #       - /account/list
            # 鉴权过滤器
            # - name: AuthFilter
            #   args: 
            #     type: jwt
            #     publicKey: 11111
        - id: order-svc
          uri: lb://order-svc
          predicates:
            - Path=/gateway/order/**
          filters:
            - StripPrefix=1
# 监控指标收集endpoint
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

## 网关启动



```java
@SpringBootApplication
@EnableDiscoveryClient
public class GateWayApplication {

    public static void main(String[] args) {
        SpringApplication.run(GateWayApplication.class, args);
    }
}
```

​    出现下面的日志，启动成功，这里日志的格式是经过日志插件格式化的，后面会单独写一篇日志插件的文档。

```shell
{"@timestamp":"2023-05-22T17:21:27.100+08:00","@version":"0.0.1","message":"Spring Cloud LoadBalancer is currently working with the default cache. While this cache implementation is useful for development and tests, it's recommended to use Caffeine cache in production.You can switch to using Caffeine cache, by adding it and org.springframework.cache.caffeine.CaffeineCacheManager to the classpath.","logger_name":"org.springframework.cloud.loadbalancer.config.LoadBalancerCacheAutoConfiguration$LoadBalancerCaffeineWarnLogger","thread_name":"main","level":"WARN","level_value":30000}
{"@timestamp":"2023-05-22T17:21:27.767+08:00","@version":"0.0.1","message":"Started GateWayApplication in 7.757 seconds (JVM running for 8.699)","logger_name":"io.redick.cloud.account.GateWayApplication","thread_name":"main","level":"INFO","level_value":20000}
```

### 网关转发请求测试

​    接口请求地址为`http://127.0.0.1:8081/gateway/order/accountLis`，是通过网关转发的，匹配了`/gateway/order/*`规则，将请求转发到`order-svc`服务。

```shell
➜  ~ curl -X GET http://127.0.0.1:8081/gateway/order/accountList
{"code":200,"msg":null,"data":[{"pageIndex":0,"pageSize":10,"orderByColumn":"","asc":"asc","productId":"123","totalCount":200,"productName":"华为P100","beginTime":null,"endTime":null,"productDesc":"华为手机牛逼"},{"pageIndex":0,"pageSize":10,"orderByColumn":"","asc":"asc","productId":"200","totalCount":100,"productName":"华为Mate100","beginTime":null,"endTime":null,"productDesc":"华为手机真牛逼"}]}
```

​    下面是不通过网关转发访问该接口，通过对比可以看到网关实现了请求转发和负载均衡，并且网关通过应用名进行服务发现。

```shell
➜  ~ curl -X GET http://127.0.0.1:8089/order/accountList
{"code":200,"msg":null,"data":[{"pageIndex":0,"pageSize":10,"orderByColumn":"","asc":"asc","productId":"123","totalCount":200,"productName":"华为P100","beginTime":null,"endTime":null,"productDesc":"华为手机牛逼"},{"pageIndex":0,"pageSize":10,"orderByColumn":"","asc":"asc","productId":"200","totalCount":100,"productName":"华为Mate100","beginTime":null,"endTime":null,"productDesc":"华为手机真牛逼"}]}
```

