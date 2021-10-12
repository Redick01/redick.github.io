# 使用javaagent实现代码无入侵增强logback <!-- {docsify-ignore-all} -->

- 前言
- 增强logback
- Web拦截器配置
- 编写javaagent程序
- 制作javaagent包
- 测试


## 前言

&nbsp; &nbsp; 程序运行日志对于系统问题排查，业务监控等都是十分重要的，Java记录日志大多通过logback，log4j等框架实现，我之前根据公司的日志规范封装了一个日志插件包，系统需要集成工具包并按照日志打印规范进行日志打印，运维系统使用filebeat收集日志到ES，开发通过ELK（ES，logstah，kibana）查看日志。但是某些很老旧且庞大的系统并没有集成日志工具包，并且因为某些原因一直没有进行改造，这就产生一些问题，比如，数据处理失败在查询日志上非常困难，因为日志中没有处理过程上下文的traceId，这就意味着，打印的各个之日都是断掉的，这根本没办法完成链路追踪。每次排查问题简直不要太痛苦；此外，对于某些接口想从日志中直观的体现出接口执行耗时也是没有的，当然我们可以使用Arthas等工具来做到记录接口耗时，但是每次想查看接口执行耗时都要是用Arthas显然也不合适。痛定思痛，有没有一种办法，在不改造系统对业务代码零入侵的情况下给系统日志中加上traceId和耗时呢？当然有，本篇文章就介绍一下通过javaagent技术+字节码增强logback（我公司用的logback，log4j也可以增强）`Logger`来实现业务系统代码无入侵增强日志。


## 增强logback

&nbsp; &nbsp; 首先定义traceId的存储，实现在日志内容中增加`traceId`的能力，我们定义一个类，我这里起名`TraceContext`就将它定义为链路的上下文，该类中只实现了traceId的能力，如果有特殊需求可以对其进行丰富。下面是`TraceContext`的代码，使用`ThreadLocal`存储`traceId`，提供的两个方法注释中已经说明，至于为什么使用`ThreadLocal`存储traceId是因为要做到线程隔离具体不解释了。

```java
/**
 * @author liupenghui
 * @date 2021/10/8 5:30 下午
 */
public class TraceContext {

    /**
     * 存储线程的traceId
     */
    private static final ThreadLocal<String> TRANSMITTABLE_THREAD_LOCAL = new ThreadLocal<>();

    /**
     * 如果 ThreadLocal中有traceId就是用否则就生成一个新的
     * @return traceId
     */
    public static String getTraceId() {
        String traceId = TRANSMITTABLE_THREAD_LOCAL.get();
        if (StringUtils.isNotBlank(traceId)) {
            return traceId;
        } else {
            // 简单使用UUID，可以使用雪花算法，我这里从简，大致演示个意思，并非工业级程序
            traceId = UUID.randomUUID().toString();
            TRANSMITTABLE_THREAD_LOCAL.set(traceId);
            return traceId;
        }
    }

    /**
     * 清理 ThreadLocal 中的 traceId
     */
    public static void remove() {
        TRANSMITTABLE_THREAD_LOCAL.remove();
    }
}
```

&nbsp; &nbsp; 然后增强logback，我这里借鉴了开源工具TLog，使用javassit进行字节码增强，代码如下：

```java
public class AspectLogEnhance {

    public static void enhance() {
        try {
            CtClass cc = null;
            try {
                ClassPool pool = ClassPool.getDefault();
                // 导入自定义logback增强类 LogbackBytesEnhance
                pool.importPackage("org.agent.enhance.bytes.logback.LogbackBytesEnhance");
                // 增强的类，ch.qos.logback.classic.Logger
                cc = pool.get("ch.qos.logback.classic.Logger");
                if (cc != null) {
                    // 增强的方法
                    CtMethod ctMethod = cc.getDeclaredMethod("buildLoggingEventAndAppend");
                    // 增强后的方法体，LogbackBytesEnhance.enhance自己实现的
                    ctMethod.setBody("{return LogbackBytesEnhance.enhance($1, $2, $3, $4, $5, $6, this);}");
                    cc.toClass();
                    System.out.println("locakback日志增强成功");
                }
            } catch (Exception e) {
                e.printStackTrace();
                System.out.println("locakback日志增强失败");
            }
        } catch (Throwable t) {
            t.printStackTrace();
        }
    }
}
```

**org.agent.enhance.bytes.logback.LogbackBytesEnhance**代码如下：

```java
public class LogbackBytesEnhance {


    /**
     * 增强ch.qos.logback.classic.Logger的buildLoggingEventAndAppend方法
     * @param fqcn
     * @param marker
     * @param level
     * @param msg
     * @param params
     * @param t
     * @param logger
     */
    public static void enhance(final String fqcn, final Marker marker, final Level level, final String msg, final Object[] params,
                               final Throwable t, Logger logger) {
        // 日志内容
        String resultLog;
        // 生成traceId
        if (StringUtils.isNotBlank(TraceContext.getTraceId())) {
            // 将traceId和日志内容拼接，这里是重点，trceId正是在这里生成并拼接到日志内容中
            resultLog = StrUtil.format("{} {}", TraceContext.getTraceId(), msg);
        } else {
            resultLog = msg;
        }
        // 增强buildLoggingEventAndAppend
        LoggingEvent loggingEvent = new LoggingEvent(fqcn, logger, level, resultLog, t, params);
        loggingEvent.setMarker(marker);
        logger.callAppenders(loggingEvent);
    }
}
```

## Web拦截器配置

&nbsp; &nbsp; 我这里的程序是基于SpringMVC的，所以我要配置Http请求的拦截器，如果接口是dubbo，SpringCloud等等其他的RPC框架也可以通过其提供的Filter或者拦截器进行处理，我这里就先只实现SpringMVC的拦截器配置，拦截器配置代码如下：

**SpringWebInterceptorConfiguration**

```java
/**
 * @author liupenghui
 * @date 2021/10/9 5:45 下午
 */
@Configuration
//有WebMvcConfigurer才加载这个配置
@ConditionalOnClass(name = {"org.springframework.web.servlet.config.annotation.WebMvcConfigurer"})
public class SpringWebInterceptorConfiguration {

    private static final Logger log = LoggerFactory.getLogger(SpringWebInterceptorConfiguration.class);

    @Bean
    public WebMvcConfiguration addWebInterceptor() {
        log.info("定义Web拦截器");
        return new WebMvcConfiguration();
    }
}
```

**WebMvcConfiguration添加拦截器**

&nbsp; &nbsp; 下面的代码实现了`WebMvcConfigurer`接口并重写了`addInterceptors`方法添加拦截器，这里添加了两个拦截器，一个负责清理`TraceContext`中的traceId，另一个负责记录接口的执行时间。

```java
/**
 * @author liupenghui
 * @date 2021/10/11 9:43 上午
 */
public class WebMvcConfiguration implements WebMvcConfigurer {

    private static final Logger log = LoggerFactory.getLogger(WebMvcConfiguration.class);

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        InterceptorRegistration interceptorRegistration;
        log.info("添加拦截器");
        // 拦截的请求路径
        interceptorRegistration = registry.addInterceptor(new WebInterceptor()).addPathPatterns("/*");
        //这里是为了兼容springboot 1.5.X，1.5.x没有order这个方法
        try{
            Method method = ReflectUtil.getMethod(InterceptorRegistration.class, "order", Integer.class);
            if (ObjectUtil.isNotNull(method)){
                method.invoke(interceptorRegistration, Ordered.HIGHEST_PRECEDENCE);
            }
        } catch (Exception e){
            e.printStackTrace();
            log.info("traceId拦截器加载失败");
        }
        interceptorRegistration = registry.addInterceptor(new TimeInterceptor()).addPathPatterns("/*");
        //这里是为了兼容springboot 1.5.X，1.5.x没有order这个方法
        try{
            Method method = ReflectUtil.getMethod(InterceptorRegistration.class, "order", Integer.class);
            if (ObjectUtil.isNotNull(method)){
                method.invoke(interceptorRegistration, Ordered.HIGHEST_PRECEDENCE);
            }
        } catch (Exception e){
            e.printStackTrace();
            log.info("耗时拦截器加载失败");
        }
    }
}
```

&nbsp; &nbsp; 配置`springboot`自动装配，在`resource/META-INF`目录下创建`spring.factories`内容如下：

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.agent.config.SpringWebInterceptorConfiguration
```




**traceId清理拦截器**

```java
/**
 * @author liupenghui
 * @date 2021/10/9 6:29 下午
 */
public class WebInterceptor implements HandlerInterceptor {

    private static final Logger log = LoggerFactory.getLogger(WebInterceptor.class);

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        log.info("拦截到请求");
        response.addHeader("traceId", TraceContext.getTraceId());
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        HandlerInterceptor.super.postHandle(request, response, handler, modelAndView);
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        log.info("清理traceId");
        TraceContext.remove();
    }
}
```

**接口耗时拦截器**

```java
/**
 * @author liupenghui
 * @date 2021/10/8 1:34 下午
 */
public class TimeInterceptor implements HandlerInterceptor {

    private static final Logger log = LoggerFactory.getLogger(TimeInterceptor.class);

    private static final ThreadLocal<StopWatch> STOP_WATCH_THREAD_LOCAL = new ThreadLocal<>();

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String url = request.getRequestURI();
        String parameters = JSON.toJSONString(request.getParameterMap());
        log.info("开始请求URL[{}]，参数为：{}", url, parameters);
        StopWatch stopWatch = new StopWatch();
        STOP_WATCH_THREAD_LOCAL.set(stopWatch);
        stopWatch.start();
        return true;
    }

    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        HandlerInterceptor.super.postHandle(request, response, handler, modelAndView);
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        StopWatch stopWatch = STOP_WATCH_THREAD_LOCAL.get();
        stopWatch.stop();
        log.info("结束URL[{}]的调用，耗时为:{}毫秒", request.getRequestURI(), stopWatch.getTime());
        STOP_WATCH_THREAD_LOCAL.remove();
    }
}
```

## 编写javaagent程序

&nbsp; &nbsp; logback增强和http接口拦截的能力都已经实现了，现在只需要编写javaagent就行了，代码很简单，在`premain`方法中`AspectLogEnhance.enhance();`即可，代码如下：

```java
/**
 * @author liupenghui
 * @date 2021/10/8 11:21 上午
 */
public class PreAgent {

    public static void premain(String args, Instrumentation inst) {
        AspectLogEnhance.enhance();
    }
}
```

关于javaagent技术的细节，这里不做介绍，可以参考我转载的文章[Javaagent使用指南](https://juejin.cn/post/7017781194481729549)

## 制作javaagent包

&nbsp; &nbsp; 程序使用maven打包，打包插件如下

```xml
<build>
        <finalName>java-agent</finalName>
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
                                        <Premain-Class>org.agent.PreAgent</Premain-Class>
                                    </manifestEntries>
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```