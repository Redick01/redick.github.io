# soul网关源码分析之soul-gateway缓存刷新

## 目标

- soul-gateway本地缓存刷新
- 总结

## 概述

&nbsp; &nbsp; 上篇讲了[soul-admin与soul-gateway通过websocket的方式进行数据的同步](https://blog.csdn.net/qq_31279701/article/details/112725043)，本篇将详细介绍soul-gateway如果将内存中的缓存刷新的。

## 架构图

&nbsp; &nbsp; 从架构图中能够看出，数据变更会通过pull模式或者push模式到soul-web模块，上篇文档提到了soul-web模块负责接收代理请求，各种filter，实际上soul-web的另一个功能就是订阅`DataChangedEvent`分发的同步事件，`soul-web`模块会处理缓存数据的更新，但是在我们构建soul网关的时候，似乎并没有引入soul-web依赖，检查网关项目依赖包的时候的确发现了soul-web模块，这是由于我们`soul-spring-boot-starter-gateway`，这个包中引入了soul-web，并且我们查看这个包的`spring.factories`内容是`provides: soul-web`，意思就是告诉spring要扫描这个模块的配置。

![avatar](_media/../../../../_media/image/source_code/soul/config-strage-processor.png)

## soul-web之缓存数据刷新

- **SoulConfiguration**

&nbsp; &nbsp; soul-web模块为该模块的配置类，配置类中配置了plugin数据变化订阅者`PluginDataSubscriber`，从源码中可以看出`CommonPluginDataSubscriber`就是实际的plugin缓存变化的处理类

```
    @Bean
    public PluginDataSubscriber pluginDataSubscriber(final ObjectProvider<List<PluginDataHandler>> pluginDataHandlerList) {
        return new CommonPluginDataSubscriber(pluginDataHandlerList.getIfAvailable(Collections::emptyList));
    }
```

- **PluginDataSubscriber**

&nbsp; &nbsp; PluginDataSubscriber是一个接口，定义了对插件，选择器，规则的数据变化订阅，取消订阅，clean所有缓存，根据key clean缓存的方法，分别是：onSubscribe，unSubscribe，refreshPluginDataAll，refreshPluginDataSelf，onSelectorSubscribe，unSelectorSubscribe，refreshSelectorDataAll，refreshSelectorDataSelf，onRuleSubscribe，unRuleSubscribe，refreshRuleDataAll，refreshRuleDataSelf；这些方法实现的逻辑对应都是相同的，代码很简单，不做过多说明。

- **CommonPluginDataSubscriber**

&nbsp; &nbsp; 该类实现了`PluginDataSubscriber`接口的全部方法，该类的核心方法是`subscribeDataHandler`，订阅数据的处理逻辑全在这个方法中，该方法的逻辑是先校验数据是否为空，接着判断数据的类型，`数据类型`有插件数据，选择器数据，规则数据，每一个条件分之中的数据处理流程都是一样的，比如插件数据，分为`UPDATE`和`DELETE`两种`操作类型`，`UPDATE`类型首先会`cachePluginData`更新缓存，然后会根据`pluginData.getName()`获取插件数据的处理器，然后调用处理器的`handlerPlugin`方法处理数据，拿`divide`插件举例，我进到`DividePluginDataHandler`处理器，发现该处理并没有重写`handlerPlugin`方法，接着到`PluginDataHandler`接口中发现`handlerPlugin`是一个`default`方法，并且是个空方法，就是说操作类型是`UPDATE`更新缓存后就完成了逻辑。

&nbsp; &nbsp; 接下来，我们看一下操作类型`DELETE`处理逻辑，处理逻辑是`PLUGIN_MAP.remove(data.getName())`，根据数据name从缓存中移除了插件，并且调用处理器的`removePlugin`方法，查看代码这也是一个空的default方法。

```
    private <T> void subscribeDataHandler(final T classData, final DataEventTypeEnum dataType) {
        Optional.ofNullable(classData).ifPresent(data -> {
            if (data instanceof PluginData) {
                PluginData pluginData = (PluginData) data;
                if (dataType == DataEventTypeEnum.UPDATE) {
                    BaseDataCache.getInstance().cachePluginData(pluginData);
                    Optional.ofNullable(handlerMap.get(pluginData.getName())).ifPresent(handler -> handler.handlerPlugin(pluginData));
                } else if (dataType == DataEventTypeEnum.DELETE) {
                    BaseDataCache.getInstance().removePluginData(pluginData);
                    Optional.ofNullable(handlerMap.get(pluginData.getName())).ifPresent(handler -> handler.removePlugin(pluginData));
                }
            } else if (data instanceof SelectorData) {
                SelectorData selectorData = (SelectorData) data;
                if (dataType == DataEventTypeEnum.UPDATE) {
                    BaseDataCache.getInstance().cacheSelectData(selectorData);
                    Optional.ofNullable(handlerMap.get(selectorData.getPluginName())).ifPresent(handler -> handler.handlerSelector(selectorData));
                } else if (dataType == DataEventTypeEnum.DELETE) {
                    BaseDataCache.getInstance().removeSelectData(selectorData);
                    Optional.ofNullable(handlerMap.get(selectorData.getPluginName())).ifPresent(handler -> handler.removeSelector(selectorData));
                }
            } else if (data instanceof RuleData) {
                RuleData ruleData = (RuleData) data;
                if (dataType == DataEventTypeEnum.UPDATE) {
                    BaseDataCache.getInstance().cacheRuleData(ruleData);
                    Optional.ofNullable(handlerMap.get(ruleData.getPluginName())).ifPresent(handler -> handler.handlerRule(ruleData));
                } else if (dataType == DataEventTypeEnum.DELETE) {
                    BaseDataCache.getInstance().removeRuleData(ruleData);
                    Optional.ofNullable(handlerMap.get(ruleData.getPluginName())).ifPresent(handler -> handler.removeRule(ruleData));
                }
            }
        });
    }
```

## 总结

&nbsp; &nbsp; 至此，soul网关的缓存刷新详细流程也分析完了，选择器和规则缓存的刷新和插件数据的刷新是一样的，soul网关每次更新数据都是将整个插件的完整数据从soul-admin发送/订阅到soul网关，soul网关执行put到map中，替换原来的缓存。