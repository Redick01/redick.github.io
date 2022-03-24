# 动态线程池dynamic-tp接入Consul配置中心 <!-- {docsify-ignore-all} -->


## 前言

&nbsp; &nbsp; 自从笔者给`dynamic-tp`接入了`Zookeeper`配置中心，就想着再扩展其他的配置中心，恰好笔者近期也在调研`Consul`配置中心，所以就想着将`Consul`配置中心接入到`dynamic-tp`。

[dynamic-tp快速接入：](https://juejin.cn/post/7073764210039062559)
[dynamic-tp官网：](https://github.com/lyh200/dynamic-tp)


## 接入Consul配置中心具体实现

&nbsp; &nbsp; `Consul`配置中心是通过定时任务做的配置变更，为了屏蔽底层实现，这里我选择对`SpringBoot`程序和`SpringCloud`应用进行接入，使用的包是`spring-cloud-starter-consul-config`，对于`SpringBoot`程序来说，集成起来相当容易，因为`spring-cloud-starter-consul-config`刷新配置的原理是刷新`Spring`容器中的配置，并且`Spring`提供了原生的监听接口`SmartApplicationListener`，实现该接口并监听`RefreshScopeRefreshedEvent`事件，刷新完毕后`Spring`发布该事件，我们就可以通过新的配置刷新线程池配置了。

#### 实现代码

&nbsp; &nbsp; 配置刷新完毕后`DtpProperties`已经是最新的配置了，直接去刷新`dynamic-tp`的配置即可。

&nbsp; &nbsp; 

```java
@Slf4j
public class CloudConsulRefresher extends AbstractRefresher implements SmartApplicationListener {

    @Resource
    private DtpProperties dtpProperties;

    @Override
    public boolean supportsEventType(@NonNull Class<? extends ApplicationEvent> eventType) {
        return RefreshScopeRefreshedEvent.class.isAssignableFrom(eventType);
    }

    @Override
    public void onApplicationEvent(@NonNull ApplicationEvent event) {
        if (event instanceof RefreshScopeRefreshedEvent) {
            doRefresh(dtpProperties);
        }
    }
}
```

## 总结

&nbsp; &nbsp; 不得不说`SpringCloud`提供的配置中心客户端简直太简单了，同样的`SpringCloud`也为`Zookeeper`,`Nacos`提供了相应的`config-starter`，之前笔者提供了基于`CuratorFramework`的实现，为的是非`SpringBoot`的程序接入，针对`SpringCloud`笔者也对`spring-cloud-starter-zookeeper-config`进行了实现，实现与`Consul`完全一样。
