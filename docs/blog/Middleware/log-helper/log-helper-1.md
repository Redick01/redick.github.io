# 基于Logback+logstash-encoder实现分布式日志链路追踪组件 - AOP核心能力 <!-- {docsify-ignore-all} -->


## 前言

&nbsp; &nbsp; 系统排查问题的过程中日志往往是开发人员的的第一选择，在单体应用中，一个好的日志书写习惯是能够通过一串数据将整个程序执行的流程串起来的，通俗点说就是日志能够形成调用链，开发人员能够方便的查询日志。但是在分布式系统中将日志形成调用链就很困难了，最简单的方式并且也是最笨的方式，就是以约定的方式将日志的链路标记（后面会直接命名为traceId）通过RPC接口参数传递到服务提供方，服务提供方的系统业务处理过程的日志中再添加这个traceId到日志中。这显然不符合一个好的系统的设计的，因为`traceId`属于分布式语意与业务无关，这么做严重的将分布式语意与业务参数耦合到一起显然是不可取的，所以大多数的公司都会根据自己公司的情况开发通用的日志组件用于分布式日志链路追踪；所以我这里基于我目前的实际情况开发了一个基于`Logback+logstash-encoder`的分布式日志链路追踪组件。

## 技术选型

&nbsp; &nbsp; 首先说一下技术选型吧，肯定会有人说为啥不用`log4j2`（当时并不是因为漏洞），这是因为公司用的线上日志查询系统是`ELK`，就是`elasticsearch`+`Logstash`+`Kibana`，所以运维为了方便日志解析要求打印的日志要格式化成json；并且性能的损耗也是可以容忍的，所以选择使用`logback`，并且通过集成开源的`logstash-logback-encoder`实现了日志格式化成json。

&nbsp; &nbsp; 基于这个需求，我们这边对日志工具的技术选型有如下几方面：

- logback
- logstash-logback-encoder
- Spring-AOP

## AOP切面接口定义

&nbsp; &nbsp; 首先定义AOP的切面接口，接口中包含从切面中获取数据的方法，代码如下：

```java
/**
 * 获取切面信息接口
 * @author penghuiliu
 * @date 2018/10/16.
 */
public interface AroundLogProxyChain {
    /**
     * 获取参数
     * @return 参数
     */
    Object[] getArgs();

    /**
     * 获取切点所在的目标对象
     * @return
     */
    Object getTarget();

    /**
     * 获取方法
     * @return Method
     */
    Method getMethod();

    /**
     * 获取目标Class
     * @return
     */
    Class<?> getClazz();

    /**
     * 获取切点
     * @return
     * @throws Throwable
     */
    Object getProceed() throws Throwable;

    /**
     * 获取切点方法签名对象
     * @return
     */
    Signature getSignature();

    /**
     * 执行方法
     * @param arguments 参数
     * @return 执行结果
     * @throws Throwable Throwable
     */
    Object doProxyChain(Object[] arguments) throws Throwable;
}
```