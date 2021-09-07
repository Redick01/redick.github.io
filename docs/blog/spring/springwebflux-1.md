# SpringWebFlux集成 <!-- {docsify-ignore-all} -->

## 目标

- 改造公司SpringMVC工程，集成SpringWebFlux
- SpringWebFlux两种接口暴露方式
- SpringWebFlux操作MysqlSpringWebFlux操作Mysql



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

- 编写路由配置：`GET`代表发布的get请求接口，`accept(TEXT_PLAIN)`参数格式，`demoHandler::hello`处理请求的处理器和方法

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

- DemoHandler：请求处理器

```
@Component
public class DemoHandler {

    @LogMarker(businessDescription = "===>webflux get test", interfaceName = "hello")
    public Mono<ServerResponse> hello(ServerRequest serverRequest) {
        // 构建相应数据
        return ServerResponse.ok()
                // 数据歌手
                .contentType(MediaType.TEXT_PLAIN)
                // 数据体
                .body(BodyInserters.fromValue("Hello WebFulx"));
    }
}
```



#### 路由多个path和handler

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
    public RouterFunction<ServerResponse> route(DemoHandler demoHandler, DemoHandler1 demoHandler1) {
        return RouterFunctions.route(GET("/hello").and(accept(TEXT_PLAIN)), demoHandler::hello)
                .andRoute(POST("/helloPost").and(accept(APPLICATION_JSON)).and(contentType(APPLICATION_JSON)), demoHandler1::helloPost);
    }
}
```



## SpringWebFlux操作Mysql

​    `Spring WebFlux`是一种非阻塞的基于`Ractor`模型的框架，基于Netty实现，在非阻塞编程里面，基于响应式的编程，线程不会被阻塞，还可以处理其他请求。举一个简单例子：假设只有一个线程池，请求来的时候，线程池处理，需要读取数据库IO，这个IO是 NIO或AIO非阻塞IO，那么就将请求数据写入数据库连接，直接返回。之后数据库返回数据，这个链接的`Selector`会有`Read事件`准备就绪，这时候，再通过这个线程池去读取数据处理`（相当于回调）`，这时候用的线程和之前不一定是同一个线程。这样的话，线程就不用等待数据库返回，而是直接处理其他请求。这样情况下，即使某个业务SQL的执行时间长，也不会影响其他业务的执行。

​    但是实现这些的基础是，IO必须是非阻塞的，遗憾的是`JDBC`官方没有非阻塞IO的实现，都是BIO的实现，这就导致了无法让线程将请求写入链接之后直接返回，必须等待响应。其实也是有解决办法的，就是将`hold`请求的`业务线程与数据库操作的线程分开（Java Future和SpringWebFlux均可实现），比如业务线程池A专门负责接口请求并将JDBC的BIO请求交给另一个线程池B去处理，处理完数据之后，再交给A执行剩下的业务逻辑。这样A也不用阻塞，可以处理其他请求。这样做也有坏处，如果一个SQL执行时间长，还是会将线程池B队列打满并阻塞，这样线程池A接收的请求也会被阻塞，真正的想要解决还是需要JDBC提供非阻塞IO的实现，此外这种方式其实得不偿失，增加了一个线程池，意味着要多进行一次线程切换，这也是开销。

​    在开源社区也有非阻塞IO的数据库客户端的实现了，这里主要讨论java，`springboot`封装了`r2dbc`（开源社区的一个非阻塞IO的mysql客户端的实现，不建议用在核心生产上）；mysql的客户端还有`Jasync-sql`使用方式可以去参考github。

​    下面通过`spring-boot-starter-data-r2dbc`来进行mysql数据库集成。我springboot版本号是`2.3.4.RELEASE`

- pom

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
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-r2dbc</artifactId>
        </dependency>
```

- yml配置，也可以自己研究写java配置

```
spring:
  r2dbc:
    url: r2dbcs:mysql://127.0.0.1:3316/account
    username: root
    password: admin123
```

- pojo

```
@Data
@Table
public class Account implements Serializable {

    @Id
    private Long id;

    private String userId;

    private BigDecimal balance;

    private BigDecimal freezeAmount;

    private Timestamp createTime;

    private Timestamp updateTime;
}
```

- 持久层接口

```
public interface AccountReactiveRepository extends ReactiveSortingRepository<Account, Long> {
}

    默认提供以下数据库操作方法
    Flux<T> findAll(Sort var1);
    
    <S extends T> Mono<S> save(S var1);

    <S extends T> Flux<S> saveAll(Iterable<S> var1);

    <S extends T> Flux<S> saveAll(Publisher<S> var1);

    Mono<T> findById(ID var1);

    Mono<T> findById(Publisher<ID> var1);

    Mono<Boolean> existsById(ID var1);

    Mono<Boolean> existsById(Publisher<ID> var1);

    Flux<T> findAll();

    Flux<T> findAllById(Iterable<ID> var1);

    Flux<T> findAllById(Publisher<ID> var1);

    Mono<Long> count();

    Mono<Void> deleteById(ID var1);

    Mono<Void> deleteById(Publisher<ID> var1);

    Mono<Void> delete(T var1);

    Mono<Void> deleteAll(Iterable<? extends T> var1);

    Mono<Void> deleteAll(Publisher<? extends T> var1);

    Mono<Void> deleteAll();
```

- 测试代码及结果

```
@RestController
@RequestMapping(value = "/controllerPattern")
public class HelloControllerTest {

    private final AccountReactiveRepository accountReactiveRepository;

    @Autowired
    public HelloControllerTest(AccountReactiveRepository accountReactiveRepository) {
        this.accountReactiveRepository = accountReactiveRepository;
    }
    
    @GetMapping(value = "/getFromDB/{userId}")
    public Mono<Account> getFromDb(@PathVariable String userId) {
        return accountReactiveRepository.findById(Long.parseLong(userId)).log();
    }
}
{"@timestamp":"2021-02-20T17:13:24.913+08:00","@version":"4.1.1","message":"onSubscribe(MonoNext.NextSubscriber)","logger_name":"reactor.Mono.Next.1","thread_name":"reactor-http-nio-2","level":"INFO","level_value":20000}
{"@timestamp":"2021-02-20T17:13:24.914+08:00","@version":"4.1.1","message":"request(unbounded)","logger_name":"reactor.Mono.Next.1","thread_name":"reactor-http-nio-2","level":"INFO","level_value":20000}
{"@timestamp":"2021-02-20T17:13:25.446+08:00","@version":"4.1.1","message":"onNext(Account(id=1, userId=10000, balance=9982377, freezeAmount=0, createTime=2017-09-18 14:54:22.0, updateTime=null))","logger_name":"reactor.Mono.Next.1","thread_name":"reactor-tcp-nio-1","level":"INFO","level_value":20000}

浏览器结果：
{"id":1,"userId":"10000","balance":9982377,"freezeAmount":0,"createTime":"2017-09-18T06:54:22.000+00:00","updateTime":null}
```

