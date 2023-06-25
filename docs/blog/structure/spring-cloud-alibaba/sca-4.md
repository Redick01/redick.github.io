## Spring Cloud Gateway-日志格式化及链路追踪插件集成 <!-- {docsify-ignore-all} -->



## 日志格式化目的

​    为了配合日志分析系统ELK(ElasticSearch，Logstash，Kibana)方便解析日志文件，需要对日志文件的输出格式进行JSON格式化，我这里使用的日志工具是logback（幸运的躲过了log4j的漏洞）+logstash-encoder包进行的封装的一个日志插件，该插件实现了日志JSON格式化，适配了多种中间件的链路追踪，支持了中间件，RPC的远程调用的耗时计算等，并且完美支持`spring xml`接入和`spring boot`接入两种方式。总的来说我这里接入这个日志插件有三方面原因，1.日志格式化、2.日志链路追踪、3.中间过程耗时分析。

​    日志插件的快速接入方式这里不做过多介绍了，可以参考我的这个插件的开源项目，项目里提供了插件接入文档已经使用示例，[项目地址是](https://github.com/Redick01/log-helper)，也可以参考我的[博客文章](https://blog.csdn.net/qq_31279701/article/details/123900584?spm=1001.2014.3001.5501)。



## Spring Cloud Gateway适配日志插件

​    `log-helper`插件开始并不支持`Spring Cloud Gateway`的链路追踪及转发下游接口的耗时统计，所以在`Spring Cloud Alibaba`开发脚手架搭建的过程中意识到了这一点，并且对其进行了适配，下面就简单说一下适配日志链路追踪的方式。

​    `log-helper`插件是基于`MDC`实现的链路追踪，各个中间件适配也是基于中间件的`Filter`或`Interceptor`实现的，所以`Spring Cloud Gateway`也是沿用这个思路。

​     `Spring Cloud Gateway`提供了过滤器功能，这里简单介绍一下， `Spring Cloud Gateway`的过滤器， `Spring Cloud Gateway`提供了两个过滤器，一个全局过滤器，一个局部过滤器，下面简单介绍一下他们之间的区别。

- 全局过滤器：`GlobalFilter`，这是一个接口，实现了该接口并实现`filter`方法，所有的请求都会经过该过滤器。

- 局部过滤器：`AbstractGatewayFilterFactory`，它是一个抽象类，继承该抽象类，并重写`apply`方法后需要在网关的配置文件中配置过滤器，如果不配置是不会流经该过滤器的，配置项是`spring.cloud.gateway.routes.filters[].name`中配置，具体配置示例如下：

  ```yaml
  spring:
    
    cloud:
      gateway:
        routes:
          - id: account-svc
            uri: lb://account-svc
            predicates:
              - Path=/gateway/account/**
            filters:
              - StripPrefix=1
              # url黑名单 这是一个局部过滤器
              - name: BlackListUrlFilter
                args: 
                  blackListUrl:
                    - /account/list
  
  ```

  `BlackListUrlFilter`对应的就是继承了`AbstractGatewayFilterFactory`的过滤器类，该实现要是`Spring`管理的。

​    通过上面对`Spring Cloud Gateway`过滤器技术的介绍，下面我们来实现我们日志插件的过滤器，这里选用的就应该是全局过滤器，因为对于网关来说，所有的请求都应该经过该过滤器，并且这里需要两个过滤器，一个是链路追踪的过滤器，一个是转发接口响应时间计算的过滤器，下面具体介绍两个过滤器实现。



### 链路追踪过滤器-TracerFilter

​    实现该过滤器后，就是要从Http请求头中获取链路信息，这其实也是一种规范，如果整个系统都要实现链路追踪，那么我们就一定要有链路追踪数据的规范，包括不限于 数据传输 规范，数据生成规范等，更高级的链路追踪规范可以参考`OpenTracing`协议，该协议是`APM`软件的协议，我这里也对该协议进行了一些参考，并实现了自己的日志级别的`Tracer`。

​    简单说链路追踪过滤器的逻辑就是从Http请求头中获取链路数据，如果有链路数据就直接用获取到的数据组装`Tracer`数据，如果没有链路数据那么就直接生成`Tracer`数据，并将`Tracer`数据传递到转发的接口上，传递也是将`Tracer`数据通过Http请求头进行传递。具体实现代码如下：

​    `AbstractInterceptor`是`log-helper`插件的抽象类，作用是从`MDC`中获取链路数据和调用其他链路前后做一些时间计算和日志打标签的功能，`Tracer`是`log-helper`提供的`OpenTracing`的参考实现。

​    值得注意的是这里实现了`Ordered`接口并且实现了`getOrder`接口，该接口是`Spring`提供的顺序管理的接口，目的是告诉整个过滤器调用链，该过滤器的优先级是最高的，也就是说要最先执行（`Ordered.HIGHEST_PRECEDENCE`）。

​    `filter`方法在处理完链路数据后调用了`executeBefore(SCG_INVOKE_START + request.getURI().getPath());`方法，该方法是给日志打了一个网关开始处理的标签，`AbstractInterceptor`的代码可以参考`log-helper`。

```java
@Slf4j
public class TracerFilter extends AbstractInterceptor implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        Mono<Void> voidMono = chain.filter(exchange.mutate()
                .request(builder -> builder.headers(httpHeaders -> {
                    List<String> traceList = httpHeaders.get(Tracer.TRACE_ID);
                    String traceId = null;
                    if (traceList != null && traceList.size() > 0) {
                        traceId = traceList.get(0);
                    }
                    List<String> spanList = httpHeaders.get(Tracer.SPAN_ID);
                    String spanId = null;
                    if (spanList != null && spanList.size() > 0) {
                        spanId = spanList.get(0);
                    }
                    List<String> parentList = httpHeaders.get(Tracer.PARENT_ID);
                    String parentId = null;
                    if (parentList != null && parentList.size() > 0) {
                        parentId = parentList.get(0);
                    }
                    Tracer.trace(traceId, spanId, parentId);
                    httpHeaders.set(Tracer.TRACE_ID, traceId());
                    httpHeaders.set(Tracer.PARENT_ID, parentId());
                    httpHeaders.set(Tracer.SPAN_ID, spanId());
                })).build());
        executeBefore(SCG_INVOKE_START + request.getURI().getPath());
        return voidMono;
    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;
    }
}
```



### 转发接口响应耗时计算过滤器-RtFilter

​    该接口的目的就是计算出网关处理完一个请求的耗时，具体代码如下：

```java
@Slf4j
public class RtFilter extends AbstractInterceptor implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {

        return chain.filter(exchange).transformDeferred(request -> {
            final long start = System.currentTimeMillis();
            return request.doFinally(s -> {
               MDC.put(START_TIME, String.valueOf(start));
               executeAfter(SCG_INVOKE_END);
            });
        });
    }

    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE - 1;
    }
}
```



### Spring Boot Autoconfigure实现

​    两个过滤器实现完，我们要将其在项目启动时加载到`Spring`中，下面就是通过`Spring Boot`自动装配实现；关于自动装配知识这里不做过多介绍。

- spring.factories

​    在`resources/META-INF/spring.factories`中配置如下内容

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.redick.starter.configure.ScgTraceAutoConfiguration
```

- Config类编写

```java
@Configuration
public class ScgTraceAutoConfiguration {
  
    @Bean
    public TracerFilter tracerFilter() {
        return new TracerFilter();
    }

    @Bean
    public RtFilter rtFilter() {
        return new RtFilter();
    }
}

```

### Spring Cloud Gateway日志链路追踪发测试

- 准备两个服务

1. ruuby-gateway：网关
2. ruuby-account-svc：库存服务

- 验证

​    启动两个服务并通过网关请求接口`curl -X GET http://127.0.0.1:8081/gateway/account/list`，日志结果如下：

> ruuby-gateway日志：

​    网关层的日志记录了链路追踪数据`traceId`,`parentId`,`spanId`和日志标签`trace_tag`，并且在第三条日志的`duration`中记录了此次请求处理耗时490ms，可以看到通过这些链路数据可以在`ELK`平台上就可以进行日志分析和问题排查了。

```json
{"@timestamp":"2023-05-22T19:26:08.742+08:00","@version":"0.0.1","message":"scg_invoke_start/gateway/account/list","logger_name":"com.redick.support.AbstractInterceptor","thread_name":"reactor-http-nio-2","level":"INFO","level_value":20000,"traceId":"44a806f6-ea41-4e3b-85a5-477052d77474","spanId":"1","parentId":"0","trace_tag":"scg_invoke_start/gateway/account/list"}
{"@timestamp":"2023-05-22T19:26:08.829+08:00","@version":"0.0.1","message":"LoadBalancerCacheManager not available, returning delegate without caching.","logger_name":"org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplierBuilder","thread_name":"reactor-http-nio-2","level":"WARN","level_value":30000,"traceId":"44a806f6-ea41-4e3b-85a5-477052d77474","spanId":"1","parentId":"0"}
{"@timestamp":"2023-05-22T19:26:09.364+08:00","@version":"0.0.1","message":"scg_invoke_end","logger_name":"com.redick.support.AbstractInterceptor","thread_name":"reactor-http-nio-4","level":"INFO","level_value":20000,"traceId":"44a806f6-ea41-4e3b-85a5-477052d77474","spanId":"1","parentId":"0","trace_tag":"scg_invoke_end","duration":490}
```

> ruuby-account-svc日志：

​    ruuby-account-svc接口的日志如下，通过网关的链路数据`traceId`就能够查到ruuby-account-svc服务的日志，致此就实现了日志层面的链路追踪。

```json
{"@timestamp":"2023-05-22T19:26:09.149+08:00","@version":"0.0.1","message":"开始处理","logger_name":"io.redick.cloud.account.controller.AccountController","thread_name":"http-nio-8088-exec-1","level":"INFO","level_value":20000,"traceId":"44a806f6-ea41-4e3b-85a5-477052d77474","spanId":"2","request_type":"account#list","parentId":"1","log_pos":"开始处理","data":[]}
{"@timestamp":"2023-05-22T19:26:09.205+08:00","@version":"0.0.1","message":"slave2数据库","logger_name":"io.redick.cloud.datasource.context.DBContextHolder","thread_name":"http-nio-8088-exec-1","level":"INFO","level_value":20000,"traceId":"44a806f6-ea41-4e3b-85a5-477052d77474","spanId":"2","request_type":"account#list","parentId":"1","log_pos":"过程"}
{"@timestamp":"2023-05-22T19:26:09.247+08:00","@version":"0.0.1","message":"开始执行sql","logger_name":"com.redick.support.mysql.Mysql5StatementInterceptor","thread_name":"http-nio-8088-exec-1","level":"INFO","level_value":20000,"traceId":"44a806f6-ea41-4e3b-85a5-477052d77474","spanId":"2","request_type":"account#list","parentId":"1","trace_tag":"sql_exec_before"}
{"@timestamp":"2023-05-22T19:26:09.250+08:00","@version":"0.0.1","message":"结束执行sql","logger_name":"com.redick.support.mysql.Mysql5StatementInterceptor","thread_name":"http-nio-8088-exec-1","level":"INFO","level_value":20000,"traceId":"44a806f6-ea41-4e3b-85a5-477052d77474","spanId":"2","request_type":"account#list","parentId":"1","trace_tag":"sql_exec_after","duration":3}
{"@timestamp":"2023-05-22T19:26:09.277+08:00","@version":"0.0.1","message":"开始执行sql","logger_name":"com.redick.support.mysql.Mysql5StatementInterceptor","thread_name":"http-nio-8088-exec-1","level":"INFO","level_value":20000,"traceId":"44a806f6-ea41-4e3b-85a5-477052d77474","spanId":"2","request_type":"account#list","parentId":"1","trace_tag":"sql_exec_before"}
{"@timestamp":"2023-05-22T19:26:09.285+08:00","@version":"0.0.1","message":"结束执行sql","logger_name":"com.redick.support.mysql.Mysql5StatementInterceptor","thread_name":"http-nio-8088-exec-1","level":"INFO","level_value":20000,"traceId":"44a806f6-ea41-4e3b-85a5-477052d77474","spanId":"2","request_type":"account#list","parentId":"1","trace_tag":"sql_exec_after","duration":8}
{"@timestamp":"2023-05-22T19:26:09.322+08:00","@version":"0.0.1","message":"处理完毕","logger_name":"io.redick.cloud.account.controller.AccountController","thread_name":"http-nio-8088-exec-1","level":"INFO","level_value":20000,"traceId":"44a806f6-ea41-4e3b-85a5-477052d77474","spanId":"2","request_type":"account#list","parentId":"1","log_pos":"处理完毕","data":{"msg":null,"serialVersionUID":1,"SUCCESS":200,"code":200,"data":[{"pageIndex":0,"pageSize":10,"orderByColumn":"","asc":"asc","productId":"123","totalCount":200,"productName":"华为P100","beginTime":null,"endTime":null,"productDesc":"华为手机牛逼"},{"pageIndex":0,"pageSize":10,"orderByColumn":"","asc":"asc","productId":"200","totalCount":100,"productName":"华为Mate100","beginTime":null,"endTime":null,"productDesc":"华为手机真牛逼"}],"FAIL":500},"duration":172,"trace_tag":"endpoint_done","result":"成功"}
```

