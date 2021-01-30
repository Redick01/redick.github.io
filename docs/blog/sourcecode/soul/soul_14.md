# soul网关源码分析之熔断插件

## 目标

- soul网关集成并配置熔断插件
- 测试不通熔断参数的熔断结果
- 分析soul网关熔断插件的原理
- 总结

## soul网关集成病配置熔断插件

​    soul网关的hystrix插件是网关用来对流量进行熔断的核心实现，使用信号量的方式来处理请求

- soul网关集成hystrix插件

```
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-plugin-hystrix</artifactId>
            <version>${soul-version}</version>
        </dependency>  
```

- soul-admin开启hystrix插件

![](../../../_media/image/source_code/soul/soul14/1612009283128.jpg)

hystrix插件选择器创建

![](../../../_media/image/source_code/soul/soul14/1612009433062.jpg)

hystrix插件规则创建

![](../../../_media/image/source_code/soul/soul14/1612009507733.jpg)

Hystrix处理详解：

1. 跳闸最小请求数量 ：最小的请求量，至少要达到这个量才会触发熔断

2. 错误半分比阀值 ： 这段时间内，发生异常的百分比。

3. 最大并发量 ： 最大的并发量

4. 跳闸休眠时间(ms) ：熔断以后恢复的时间。

5. 分组Key： 一般设置为:contextPath

6. 命令Key: 一般设置为具体的 路径接口。

## 测试不同熔断参数的熔断结果

- **case1**

  - 熔断插件配置：跳闸最小请求数量5，错误百分比阀值50，最大并发量10测试正常情况
  - 测试条件：
