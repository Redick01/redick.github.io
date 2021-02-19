# SpringWebFlux集成

## 目标

- 改造公司SpringMVC工程，集成SpringWebFlux
- SpringWebFlux两种接口暴露方式



##改造公司SpringMVC工程，集成SpringWebFlux

​    基于`MISS.BasicPlatform_Rest`改造

### pom文件

```
        <!--Spring WebFlux support start-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-webflux</artifactId>
        </dependency>
        <dependency>
            <groupId>io.projectreactor</groupId>
            <artifactId>reactor-test</artifactId>
            <scope>test</scope>
        </dependency>
        <!--Spring WebFlux support end-->
```

**`注意`**：基于`MISS.BasicPlatform_Rest`改造时要去掉`pom`中的`spring-boot-starter-web`，由于`MISS.BasicPlatform_Rest`，中的个别依赖包、脚手架都有引`spring-boot-starter-web`，所以，要在pom中`exclusion`干净，如果不排除干净包，会导致程序启动后无法通过`SpringWebFulx`的`Router Configuration`的方式暴露接口，因为`SpringWebFlux`基于`netty`而`SpringMVC`是内置了`tomcat`，所以`spring-boot-starter-web`不排除干净的话配置会有冲突；以下是pom中目前发现要排除包的点

```
        <dependency>
            <groupId>com.github.chenhaiyangs</groupId>
            <artifactId>web-validation-spring-boot-starter</artifactId>
            <version>1.2.0</version>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-web</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>com.ruubypay.miss</groupId>
            <artifactId>ruubypay-log-spring-boot-starter</artifactId>
            <version>1.0-RELEASE</version>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-web</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
```



###  yml或properties配置

​    这里以yml举例

```
spring:
  main:
    allow-bean-definition-overriding: true
server:
  port: 8808
  address: 0.0.0.0

zk:
  address: 172.17.10.157:2181
  version: 1.0.0
  root: /config/MISS.Core.TransDisposeService
```



## SpringWebFlux两种接口暴露方式



###  `@RestController`方式

​    该方式与SpringMVC的Controller的使用当使用方式相似，

```
@RestController
@RequestMapping(value = "/controllerPattern")
public class HelloControllerTest {

    @PostMapping(value = "/postTest")
    @LogMarker(businessDescription = "===>/controllerPattern/postTest")
    public Mono<HelloResponseDTO> postTest(HelloDTO helloDTO) throws Exception {
        System.out.println(helloDTO.getMsg());
        HelloResponseDTO responseDTO = new HelloResponseDTO();
        responseDTO.setResp("22222");
        return Mono.just(responseDTO);
    }
}
```



### 配置`RouterFunction`方式



#### 路由单个path和handler

- 编写路由配置

```
@Configuration
@ComponentScan("com.ruubypay.miss.biz.handler")
public class WebRouterConfiguration {
    
    /**
     * 路由配置
     * @param demoHandler 请求处理器
     * @return 响应
     */
    @Bean
    public RouterFunction<ServerResponse> route(DemoHandler demoHandler) {
        return RouterFunctions.route(GET("/hello").and(accept(TEXT_PLAIN)), demoHandler::hello);
    }
}
```

- DemoHandler

```
@Component
public class DemoHandler {

    @LogMarker(businessDescription = "===>webflux get test", interfaceName = "hello")
    public Mono<ServerResponse> hello(ServerRequest serverRequest) {
        return ServerResponse.ok()
                .contentType(MediaType.TEXT_PLAIN)
                .body(BodyInserters.fromValue("Hello WebFulx"));
    }
}
```



