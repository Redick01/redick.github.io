#  soul网关源码分析之sofa插件

## 目标

- 集成使用sofa插件
- soul网关完成sofa调用源码分析
- 总结



##网关集成 sofa插件

- **pom**

```
	   <dependency>
           <groupId>com.alipay.sofa</groupId>
           <artifactId>sofa-rpc-all</artifactId>
           <version>5.7.6</version>
       </dependency>
       <dependency>
           <groupId>org.apache.curator</groupId>
           <artifactId>curator-client</artifactId>
           <version>4.0.1</version>
       </dependency>
       <dependency>
           <groupId>org.apache.curator</groupId>
           <artifactId>curator-framework</artifactId>
           <version>4.0.1</version>
       </dependency>
       <dependency>
           <groupId>org.apache.curator</groupId>
           <artifactId>curator-recipes</artifactId>
           <version>4.0.1</version>
       </dependency>
       <dependency>
           <groupId>org.dromara</groupId>
           <artifactId>soul-spring-boot-starter-plugin-sofa</artifactId>
           <version>${soul-version}</version>
       </dependency>
```



## 业务服务接入soul网关

- **pom**

```
	   <dependency>
           <groupId>org.dromara</groupId>
           <artifactId>soul-spring-boot-starter-client-sofa</artifactId>
           <version>${soul-version}</version>
       </dependency>
```

- **application.yml**

```
soul:
  sofa:
    adminUrl: http://localhost:9095
    contextPath: /sofa
    appName: sofa
```

- **接口注册到网关**

- - 你sofa服务实现类的，方法上加上 @SoulSofaClient 注解，表示该接口方法注册到网关。
- - 启动你的提供者，输出日志 `sofa client register success` 大功告成，你的sofa接口已经发布到 soul网关.如果还有不懂的，可以参考 `soul-test-sofa`项目。

```
@Service("sofaTestService")
public class SofaTestServiceImpl implements DubboTestService {

    @Override
    @SoulSofaClient(path = "/findById", desc = "Find by Id")
    public DubboTest findById(final String id) {
        DubboTest dubboTest = new DubboTest();
        dubboTest.setId(id);
        dubboTest.setName("hello world Soul Sofa, findById");
        return dubboTest;
    }

    @Override
    @SoulSofaClient(path = "/findAll", desc = "Get all data")
    public DubboTest findAll() {
        DubboTest dubboTest = new DubboTest();
        dubboTest.setName("hello world Soul Sofa , findAll");
        dubboTest.setId(String.valueOf(new Random().nextInt()));
        return dubboTest;
    }

    @Override
    @SoulSofaClient(path = "/insert", desc = "Insert a row of data")
    public DubboTest insert(final DubboTest dubboTest) {
        dubboTest.setName("hello world Soul Sofa: " + dubboTest.getName());
        return dubboTest;
    }
}
```

![](/Users/penghuiliu/docsify-doc/docs/_media/image/source_code/soul/soul12/sofademo.jpg)

- **soul-admin注册接口结果**

![](/Users/penghuiliu/docsify-doc/docs/_media/image/source_code/soul/soul12/sofaselector.jpg)

## soul-admin支持sofa插件

- 首先在 `soul-admin` 插件管理中，把`sofa` 插件设置为开启。
- 其次在 `sofa` 插件中配置你的注册地址或者其他注册中心的地址.

![](/Users/penghuiliu/docsify-doc/docs/_media/image/source_code/soul/soul12/1611762495830.jpg)

## soul网关完成sofa调用源码分析

- **SofaPluginConfiguration**

```
@Configuration
@ConditionalOnClass(SofaPlugin.class)
public class SofaPluginConfiguration {
    
    /**
     * sofa插件初始化
     */
    @Bean
    public SoulPlugin sofaPlugin(final ObjectProvider<SofaParamResolveService> sofaParamResolveService) {
    		// SofaProxyService sofa服务代理初始化
        return new SofaPlugin(new SofaProxyService(sofaParamResolveService.getIfAvailable()));
    }
    
    /**
     * sofa参数体插件 初始化
     */
    @Bean
    public SoulPlugin bodyParamPlugin() {
        return new BodyParamPlugin();
    }
    
    /**
     * sofa请求响应处理插件初始化
     */
    @Bean
    public SoulPlugin sofaResponsePlugin() {
        return new SofaResponsePlugin();
    }
    
    /**
     * sofa数据处理器插件初始化，主要是初始化rpc注册数据
     */
    @Bean
    public PluginDataHandler sofaPluginDataHandler() {
        return new SofaPluginDataHandler();
    }
    
    /**
     * sofa元数据变更订阅初始化
     */
    @Bean
    public MetaDataSubscriber sofaMetaDataSubscriber() {
        return new SofaMetaDataSubscriber();
    }
}
```

- **SofaPlugin**

​    以下是sofa请求处理的核心代码

```
	@Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        // 从请求参数重获取数据
        String body = exchange.getAttribute(Constants.SOFA_PARAMS);
        // 获取soul网关上下文
        SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        // 获取元数据
        MetaData metaData = exchange.getAttribute(Constants.META_DATA);
        // 检查元数据
        if (!checkMetaData(metaData)) {
            assert metaData != null;
            log.error(" path is :{}, meta data have error.... {}", soulContext.getPath(), metaData.toString());
            exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
            Object error = SoulResultWrap.error(SoulResultEnum.META_DATA_ERROR.getCode(), SoulResultEnum.META_DATA_ERROR.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        if (StringUtils.isNoneBlank(metaData.getParameterTypes()) && StringUtils.isBlank(body)) {
            exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
            Object error = SoulResultWrap.error(SoulResultEnum.SOFA_HAVE_BODY_PARAM.getCode(), SoulResultEnum.SOFA_HAVE_BODY_PARAM.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        // 通过代理调用sofa接口
        final Mono<Object> result = sofaProxyService.genericInvoker(body, metaData, exchange);
        return result.then(chain.execute(exchange));
    }
```

- **SofaProxyService**

​    以下是进行sofa远程调用的核心处理，可以看到类似于dubbo，也是进行了泛化调用

```
		public Mono<Object> genericInvoker(final String body, final MetaData metaData, final ServerWebExchange exchange) throws SoulException {
		    // 获取泛化代理引用
        ConsumerConfig<GenericService> reference = ApplicationConfigCache.getInstance().get(metaData.getPath());
        if (Objects.isNull(reference) || StringUtils.isEmpty(reference.getInterfaceId())) {
            ApplicationConfigCache.getInstance().invalidate(metaData.getServiceName());
            reference = ApplicationConfigCache.getInstance().initRef(metaData);
        }
        GenericService genericService = reference.refer();
        Pair<String[], Object[]> pair;
        if (null == body || "".equals(body) || "{}".equals(body) || "null".equals(body)) {
            pair = new ImmutablePair<>(new String[]{}, new Object[]{});
        } else {
            pair = sofaParamResolveService.buildParameter(body, metaData.getParameterTypes());
        }
        CompletableFuture<Object> future = new CompletableFuture<>();
        // 响应回调
        RpcInvokeContext.getContext().setResponseCallback(new SofaResponseCallback<Object>() {
            @Override
            public void onAppResponse(final Object o, final String s, final RequestBase requestBase) {
                future.complete(o);
            }

            @Override
            public void onAppException(final Throwable throwable, final String s, final RequestBase requestBase) {
                future.completeExceptionally(throwable);
            }

            @Override
            public void onSofaException(final SofaRpcException e, final String s, final RequestBase requestBase) {
                future.completeExceptionally(e);
            }
        });
        // 泛化调用
        genericService.$invoke(metaData.getMethodName(), pair.getLeft(), pair.getRight());
        // 根据回调，处理返回数据
        return Mono.fromFuture(future.thenApply(ret -> {
            if (Objects.isNull(ret)) {
                ret = Constants.SOFA_RPC_RESULT_EMPTY;
            }
            exchange.getAttributes().put(Constants.SOFA_RPC_RESULT, ret);
            exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.SUCCESS.getName());
            return ret;
        })).onErrorMap(SoulException::new);
    }
```

- **SofaResponsePlugin**

​    以下是处理sofa调用响应数据的核心代码

```
	@Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
    		// 责任链处理
        return chain.execute(exchange).then(Mono.defer(() -> {
            final Object result = exchange.getAttribute(Constants.SOFA_RPC_RESULT);
            if (Objects.isNull(result)) {
                Object error = SoulResultWrap.error(SoulResultEnum.SERVICE_RESULT_ERROR.getCode(), SoulResultEnum.SERVICE_RESULT_ERROR.getMsg(), null);
                return WebFluxResultUtils.result(exchange, error);
            }
            Object success = SoulResultWrap.success(SoulResultEnum.SUCCESS.getCode(), SoulResultEnum.SUCCESS.getMsg(), JsonUtils.removeClass(result));
            // 处理返回数据
            return WebFluxResultUtils.result(exchange, success);
        }));
    }
```

- **回顾**

   至此，soul网关支持sofa调用的源码处理流程大概就分析完成，具体处理细节有待进一步完善



## 总结

​    soul网关支持sofa协议调用在应用时与dubbo非常相似，首先都需要集成sofa插件，并且要在soul-admin中开启sofa插件，并且要在soul-admin的sofa插件中配置注册中心；在源码分析阶段可以看到sofa的处理流程是沿用了责任链模式处理插件链，在rpc调用的时候与dubbo也是相似的。

