# Soul网关使用

## 基于springboot搭建自己soul网关

- pom

```
    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
        <soul-version>2.2.1</soul-version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-webflux</artifactId>
            <version>2.2.2.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
            <version>2.2.2.RELEASE</version>
        </dependency>
        <!--soul gateway start-->
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-gateway</artifactId>
            <version>${soul-version}</version>
        </dependency>
        <!--soul data sync start use websocket-->
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-sync-data-websocket</artifactId>
            <version>${soul-version}</version>
        </dependency>
        <!--if you use http proxy start this-->
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-plugin-divide</artifactId>
            <version>${soul-version}</version>
        </dependency>
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-plugin-httpclient</artifactId>
            <version>${soul-version}</version>
        </dependency>
    </dependencies>
```

- 启动类

```
@SpringBootApplication
public class SoulBootstrapApplication {

    public static void main(String[] args) {
        SpringApplication.run(SoulBootstrapApplication.class, args);
    }
}
```

- 启动soul-admin和自己的soul网关

## Http请求接入网关

- 基于SpingBoot搭建spirngMvc项目

- pom依赖

```
        <!-- 启动springbootstarter支持 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <!-- springboot web启动 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <scope>compile</scope>
        </dependency>
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-client-springmvc</artifactId>
            <version>${soul-version}</version>
        </dependency>
```

- yml配置

```
server:
  port: 8085
soul:
  http:
    adminUrl: http://localhost:9095
    port: 8085
    contextPath: /soul
    appName: http
    full: false
```

- Controller编写

/test/** 代表当前类所有接口都会被代理
```
@RestController
@RequestMapping("/test")
@SoulSpringMvcClient(path = "/test/**")
public class SoulTestController {

    @PostMapping("/hello")
    public String hello(String req) {

        return req;
    }
}
```

- 启动访问

- - 直连访问URL：http://localhost:8085/test/hello
- - soul代理URL：http://localhost:8080/soul/test/hello
