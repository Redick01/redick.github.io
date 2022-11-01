# JAVA - 通过gRPC拦截器实现分布式日志链路追踪 <!-- {docsify-ignore-all} -->

&nbsp; &nbsp; 之前开源过一个分布式日志链路追踪的工具，其作用是规范日志格式，实现分布式日志层面的链路追踪，并且工具支持SpringMVC，Dubbo，OpenFeign，HttpClient，OkHttp等网络工具或RPC框架，基于此，为了扩展日志链路追踪使用场景，同时最近又在学习JAVA+gRPC，所以将该日志工具的链路追踪能力扩展了到gRPC场景。


## 跨进程链路追踪原理

&nbsp; &nbsp; 想要实现跨进程间的分布式链路追踪，就要在发起远程调用的时候通过请求头或者公共的自定义域将链路参数放进去，然后服务端收到请求后将链路参数从请求头或者自定义域中或取出来，就这样一层一层的将链路参数传递下去直至调用结束。

&nbsp; &nbsp; JAVA的gRPC库io.grpc提供了在RPC调用中客户端和服务端的拦截器（Interceptor），通过客户端拦截器我们可以将链路追踪的参数放到gRPC调用的`Metadata`中，通过服务端拦截器能够从`Metadata`中获取到链路追踪所传递的参数；io.grpc提供的客户端拦截器和服务端拦截器分别是`io.grpc.ClientInterceptor`和`io.grpc.ServerInterceptor`。

## 代码实现

- maven依赖

```xml
    <dependencies>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-all</artifactId>
            <version>${grpc.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>net.devh</groupId>
            <artifactId>grpc-server-spring-boot-starter</artifactId>
            <version>${grpc.starter.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>net.devh</groupId>
            <artifactId>grpc-client-spring-boot-starter</artifactId>
            <version>${grpc.starter.version}</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>io.github.redick01</groupId>
            <artifactId>log-helper-spring-boot-starter-common</artifactId>
            <version>1.0.3-RELEASE</version>
        </dependency>
    </dependencies>
```

- 拦截器实现

```java
@Slf4j
@GrpcGlobalClientInterceptor
@GrpcGlobalServerInterceptor
public class GrpcInterceptor extends AbstractInterceptor implements ServerInterceptor, ClientInterceptor {
    
    // 链路追踪参数traceId
    private static final Metadata.Key<String> TRACE = Metadata.Key.of("traceId", Metadata.ASCII_STRING_MARSHALLER);

    // 链路追踪参数spanId
    private static final Metadata.Key<String> SPAN = Metadata.Key.of("spanId", Metadata.ASCII_STRING_MARSHALLER);

    // 链路追踪参数parentId
    private static final Metadata.Key<String> PARENT = Metadata.Key.of("parentId", Metadata.ASCII_STRING_MARSHALLER);

    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(
            MethodDescriptor<ReqT, RespT> methodDescriptor, CallOptions callOptions,
            Channel channel) {
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        try {
            return new ForwardingClientCall.SimpleForwardingClientCall<ReqT, RespT>(channel.newCall(methodDescriptor, callOptions)) {
                @Override
                public void start(Listener<RespT> responseListener, Metadata headers) {
                    // 客户端传递链路追中数据，将数据放到headers中
                    String traceId = traceId();
                    if (StringUtils.isNotBlank(traceId)) {
                        headers.put(TRACE, traceId);
                        headers.put(SPAN, spanId());
                        headers.put(PARENT, parentId());
                    }
                    // 继续下一步
                    super.start(new ForwardingClientCallListener.SimpleForwardingClientCallListener<RespT>(responseListener) {
                        @Override
                        public void onHeaders(Metadata headers) {
                            // 服务端传递回来的header
                            super.onHeaders(headers);
                        }
                    }, headers);
                }
            };
        } finally {
            stopWatch.stop();
            log.info(LogUtil.marker(stopWatch.getTime()), "GRPC调用耗时");
        }
    }

    @Override
    public <ReqT, RespT> Listener<ReqT> interceptCall(ServerCall<ReqT, RespT> serverCall,
            Metadata headers, ServerCallHandler<ReqT, RespT> serverCallHandler) {
        // 服务端从headers中获取到链路追踪参数
        String traceId = headers.get(TRACE);
        String spanId = headers.get(SPAN);
        String parentId = headers.get(PARENT);
        // 构建当前进程的链路追踪数据并体现在日志中
        Tracer.trace(traceId, spanId, parentId);
        log.info(LogUtil.marker(), "开始处理");
        return serverCallHandler.startCall(new ForwardingServerCall.SimpleForwardingServerCall<ReqT, RespT>(serverCall) {

            @Override
            public void sendHeaders(Metadata responseHeaders) {
                super.sendHeaders(responseHeaders);
            }

            @Override
            public void close(Status status, Metadata trailers) {
                super.close(status, trailers);
            }
        }, headers);
    }
}
```

- 客户端使用

客户端使用代码如下，该使用示例是在我开源的日志工具中的例子，我这里通过springboot自动装配将`GrpcInterceptor`交由spring容器管理。所以可以直接通过自动注入的方式使用。

```java
@RestController
public class TestController {

    @GrpcClient("userClient")
    private UserServiceGrpc.UserServiceBlockingStub userService;

    @Autowired
    private GrpcInterceptor grpcInterceptor;

    //@LogMarker(businessDescription = "获取用户名")
    @GetMapping("/getUser")
    public String getUser()     {
        User user = User.newBuilder()
                .setUserId(100)
                .putHobbys("pingpong", "play pingpong")
                .setCode(200)
                .build();
        Channel channel = ClientInterceptors.intercept(userService.getChannel(), grpcInterceptor);
        userService = UserServiceGrpc.newBlockingStub(channel);
        User u = userService.getUser(user);
        return u.getName();
    }
}
```

## 总结

&nbsp; &nbsp; Java使用gRPC完成的服务间的调用可以通过`io.grpc.ClientInterceptor`和`io.grpc.ServerInterceptor`定义客户端和服务端的拦截器实现分布式链路追踪。
&nbsp; &nbsp; 本文涉及的代码可以在我之前开源的分布式链路追踪的日志工具中找到，项目地址：https://github.com/Redick01/log-helper