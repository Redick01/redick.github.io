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

### 获取Invoker列表

#### AbstractDirectory#list 代码如下：

```java
    @Override
    public List<Invoker<T>> list(Invocation invocation) throws RpcException {
        if (destroyed) {
            throw new RpcException("Directory already destroyed .url: " + getUrl());
        }
        // 调用RegistryDirectory#doList
        return doList(invocation);
    }
```

#### RegistryDirectory#doList 代码如下：

```java    
    @Override
    public List<Invoker<T>> doList(Invocation invocation) {
        // provider 没有或者 provider被设置城 disabled的处理
        if (forbidden) {
            // 1. No service provider 2. Service providers are disabled
            throw new RpcException(RpcException.FORBIDDEN_EXCEPTION, "No provider available from registry " +
                    getUrl().getAddress() + " for service " + getConsumerUrl().getServiceKey() + " on consumer " +
                    NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() +
                    ", please check status of providers(disabled, not registered or in blacklist).");
        }
        
        if (multiGroup) {
            return this.invokers == null ? Collections.emptyList() : this.invokers;
        }

        List<Invoker<T>> invokers = null;
        try {
            // Get invokers from cache, only runtime routers will be executed.
            // 执行Router链 过滤Invoker列表
            invokers = routerChain.route(getConsumerUrl(), invocation);
        } catch (Throwable t) {
            logger.error("Failed to execute router: " + getUrl() + ", cause: " + t.getMessage(), t);
        }

        return invokers == null ? Collections.emptyList() : invokers;
    }
```

#### RouterChain#route 执行Router链过滤Invoker列表

&nbsp; &nbsp; `RouterChain#route`方法迭代执行Router接口，Router接口根据路由规则过滤Invoker列表，代码如下：


```java
    public List<Invoker<T>> route(URL url, Invocation invocation) {
        List<Invoker<T>> finalInvokers = invokers;
        for (Router router : routers) {
            finalInvokers = router.route(finalInvokers, url, invocation);
        }
        return finalInvokers;
    }
```

### 初始化负载均衡

&nbsp; &nbsp; `SPI`机制获取`LoadBalance`实现

```java
    protected LoadBalance initLoadBalance(List<Invoker<T>> invokers, Invocation invocation) {
        if (CollectionUtils.isNotEmpty(invokers)) {
            return ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(invokers.get(0).getUrl()
                    .getMethodParameter(RpcUtils.getMethodName(invocation), LOADBALANCE_KEY, DEFAULT_LOADBALANCE));
        } else {
            return ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(DEFAULT_LOADBALANCE);
        }
    }
```

> Dubbo提供的`LoadBalance`实现如下：

- **RandomLoadBalance：**加权随机，该算法是Dubbo提供的默认算法
- **RoundRobinLoadBalance：**轮询算法
- **ShortestResponseLoadBalance：**最短响应时间优先+加权随机算法
- **LeastActiveLoadBalance：**最少活跃算法
- **ConsistentHashLoadBalance：**一致性hash算法

> 以RandomLoadBalance为例看一下代码

```java
public class RandomLoadBalance extends AbstractLoadBalance {

    public static final String NAME = "random";

    /**
     * Select one invoker between a list using a random criteria
     * @param invokers List of possible invokers
     * @param url URL
     * @param invocation Invocation
     * @param <T>
     * @return The selected invoker
     */
    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        // Invoker数量
        int length = invokers.size();
        // 每个Invoker的权重是否相同
        boolean sameWeight = true;
        // 每个invoker的权重
        int[] weights = new int[length];
        // 第一个invoker的权重
        int firstWeight = getWeight(invokers.get(0), invocation);
        weights[0] = firstWeight;
        // 权重的和
        int totalWeight = firstWeight;
        for (int i = 1; i < length; i++) {
            int weight = getWeight(invokers.get(i), invocation);
            // save for later use
            weights[i] = weight;
            // 求和
            totalWeight += weight;
            if (sameWeight && weight != firstWeight) {
                sameWeight = false;
            }
        }
        if (totalWeight > 0 && !sameWeight) {
            // 根据总的总权重和计算随机值
            int offset = ThreadLocalRandom.current().nextInt(totalWeight);
            // 随机值-invoker权重，当offset小于0时就获取返回那个invoker
            for (int i = 0; i < length; i++) {
                offset -= weights[i];
                if (offset < 0) {
                    return invokers.get(i);
                }
            }
        }
        // 如果所有的权重相同，就随即返回invoker
        return invokers.get(ThreadLocalRandom.current().nextInt(length));
    }

}
```

> 在应用中配置

- **服务端**

```xml
<dubbo:service interface="..." loadbalance="roundrobin" />
```

- **客户端**

```xml
<dubbo:reference interface="..." loadbalance="roundrobin" />
```

- **服务端方法级别**

```xml
<dubbo:service interface="...">
    <dubbo:method name="..." loadbalance="roundrobin"/>
</dubbo:service>
```

- **客户端方法级别**

```xml
<dubbo:reference interface="...">
    <dubbo:method name="..." loadbalance="roundrobin"/>
</dubbo:reference>
```


### FailoverClusterInvoker#doInvoke

```java
    @Override
    @SuppressWarnings({"unchecked", "rawtypes"})
    public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        List<Invoker<T>> copyInvokers = invokers;
        checkInvokers(copyInvokers, invocation);
        String methodName = RpcUtils.getMethodName(invocation);
        int len = getUrl().getMethodParameter(methodName, RETRIES_KEY, DEFAULT_RETRIES) + 1;
        if (len <= 0) {
            len = 1;
        }
        // retry loop.
        RpcException le = null; // last exception.
        List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyInvokers.size()); // invoked invokers.
        Set<String> providers = new HashSet<String>(len);
        for (int i = 0; i < len; i++) {
            //Reselect before retry to avoid a change of candidate `invokers`.
            //NOTE: if `invokers` changed, then `invoked` also lose accuracy.
            if (i > 0) {
                checkWhetherDestroyed();
                copyInvokers = list(invocation);
                // check again
                checkInvokers(copyInvokers, invocation);
            }
            // 负载均衡操作，最终会调用RandomLoadBalance#doSelect方法
            Invoker<T> invoker = select(loadbalance, invocation, copyInvokers, invoked);
            invoked.add(invoker);
            RpcContext.getContext().setInvokers((List) invoked);
            try {
                // 远程调用
                Result result = invoker.invoke(invocation);
                if (le != null && logger.isWarnEnabled()) {
                    logger.warn("Although retry the method " + methodName
                            + " in the service " + getInterface().getName()
                            + " was successful by the provider " + invoker.getUrl().getAddress()
                            + ", but there have been failed providers " + providers
                            + " (" + providers.size() + "/" + copyInvokers.size()
                            + ") from the registry " + directory.getUrl().getAddress()
                            + " on the consumer " + NetUtils.getLocalHost()
                            + " using the dubbo version " + Version.getVersion() + ". Last error is: "
                            + le.getMessage(), le);
                }
                return result;
            } catch (RpcException e) {
                if (e.isBiz()) { // biz exception.
                    throw e;
                }
                le = e;
            } catch (Throwable e) {
                le = new RpcException(e.getMessage(), e);
            } finally {
                providers.add(invoker.getUrl().getAddress());
            }
        }
        throw new RpcException(le.getCode(), "Failed to invoke the method "
                + methodName + " in the service " + getInterface().getName()
                + ". Tried " + len + " times of the providers " + providers
                + " (" + providers.size() + "/" + copyInvokers.size()
                + ") from the registry " + directory.getUrl().getAddress()
                + " on the consumer " + NetUtils.getLocalHost() + " using the dubbo version "
                + Version.getVersion() + ". Last error is: "
                + le.getMessage(), le.getCause() != null ? le.getCause() : le);
    }
```