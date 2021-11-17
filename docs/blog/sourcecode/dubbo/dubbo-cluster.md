# Dubbo集群容错 <!-- {docsify-ignore-all} -->


&nbsp; &nbsp; Dubbo调用失败时提供了容错方案；当存在多个provider实例时Dubbo提供了路由和负载均衡的能力，下面时来自Dubbo官网的一张图，图片中展示了Dubbo集群容错的模型。

![avatar](_media/../../../../_media/image/source_code/dubbo/cluster.jpeg)

> 各节点关系

- 这里的 Invoker 是 Provider 的一个可调用 Service 的抽象，Invoker 封装了 Provider 地址及 Service 接口信息
- Directory 代表多个 Invoker，可以把它看成 List<Invoker> ，但与 List 不同的是，它的值可能是动态变化的，比如注册中心推送变更
- Cluster 将 Directory 中的多个 Invoker 伪装成一个 Invoker，对上层透明，伪装过程包含了容错逻辑，调用失败后，重试另一个
- Router 负责从多个 Invoker 中按路由规则选出子集，比如读写分离，应用隔离等
- LoadBalance 负责从多个 Invoker 中选出具体的一个用于本次调用，选的过程包含了负载均衡算法，调用失败后，需要重选


## 集群容错模式

> Dubbo提供的集群策略类型如下：

- **FailoverCluster失败转移：** 当出现失败时，重试其他服务器，通常用于读操作，但重试会带来更长延迟，该策略是dubbo默认策略。
- **FailfastCluster快速失败：** 只发起一次调用，失败立即报错，通常用于非幂等的请求。
- **FailbackCluster失败自动恢复：** 对于Invoker调用失败，后台记录失败请求，任务定时重发，通常用于通知。
- **BroadcastCluster广播调用：** 遍历调用所有Invoker，如果调用某个invoker异常了，直接捕获异常，不影响调用其他的invoker。
- **AvailableCluster获取可用的调用：** 遍历所有invoker并判断invoker.isAvalible，只要有一个为true就直接调用返回，不管是否成功。
- **FailsafeCluster失败安全：** 出现异常时直接忽略，通常用于写入审计日志。
- **ForkingCluster并行调用：** 只要一个成功即返回，通常用于实时性要求较高的操作，但需要浪费更多的服务资源。
- **MergeableCluster分组聚合：** 按组合并返回结果，比如某个服务接口有多种实现，可以用group区分，调用者调用多种实现并将得到的结果合并。

## 集群容错模式的配置

&nbsp; &nbsp; 集群模式的配置有如下两种方式，分为provider端和consumer端，两者取其一即可

- provider端

```xml
<dubbo:service cluster="failsafe" />
```

- consumer端

```xml
<dubbo:reference cluster="failsafe" />
```

## 以FailoverCluster模式的源码分析

&nbsp; &nbsp; 首先我们来看一下集群容错处理的入口`AbstractClusterInvoker`的`invoke`方法

```java
    @Override
    public Result invoke(final Invocation invocation) throws RpcException {
        checkWhetherDestroyed();

        // binding attachments into invocation.
        Map<String, Object> contextAttachments = RpcContext.getContext().getObjectAttachments();
        if (contextAttachments != null && contextAttachments.size() != 0) {
            ((RpcInvocation) invocation).addObjectAttachments(contextAttachments);
        }
        // 获取Invoker列表
        List<Invoker<T>> invokers = list(invocation);
        // 初始化负载均衡
        LoadBalance loadbalance = initLoadBalance(invokers, invocation);
        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
        // 调用 FailoverCluster#doInvoke方法 执行容错策略
        return doInvoke(invocation, invokers, loadbalance);
    }
```

> 获取Invoker列表

&nbsp; &nbsp; 获取Invoker列表