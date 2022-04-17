# 基于Zookeeper的服务分片 <!-- {docsify-ignore-all} -->

## 背景

&nbsp; &nbsp; 公司有一个服务需要做类似于分片的逻辑，一开始服务基于传统部署方式通过本地配置文件就可以指定该机器服务的分片，最近该服务进行了容器化部署，所以原来基于本地配置文件各自配各自的分片信息的方式就不适用了，原来的部署方式使得服务存在状态，是一种非云原生的方式，所以该服务要重新设计实现一套`分布式服务分片`逻辑。


## 技术方案

#### 分布式协调中间件

&nbsp; &nbsp; 要实现分布式服务分片的能力，需要有一个分布式中间件，如：`Redis`，`Mysql`，`Zookeeper`等等都可以，这里选用`Zookeeper`。


#### 基于Zookeeper的技术方案

&nbsp; &nbsp; 使用`Zookeeper`主要是基于`Zookeeper`的临时节点和节点变化监听机制；具体的技术设计如下：


###### **服务注册目录设计**


`Zookeeper`的数据存储结构类似于目录，服务注册后的目录类似如下结构：

解释下该目录结构，首先`/xxxx/xxxx/sharding`是区别于其他业务的的目录，该目录节点是持久的，`service`是服务目录，标识一个服务，该节点也是持久的，`ip1`，`ip2`是该服务注册到`Zookeeper`的机器列表节点，该节点是临时节点。

```shell
/xxxx/xxxx/sharding/service/ip1
-----|----|--------|-------/ip2
```

###### **服务分片处理流程**

- 服务启动，创建`CuratorFramework`客户端你，设置客户端连接状态监听；
- 向`Zookeeper`注册该机器的信息，这里设计简单，机器信息就是`ip`地址；
- 注册机器信息后，从`Zookeeper`获取所有注册信息；
- 根绝`Zookeeper`获取的所有注册机器信息根据分片算法进行分片计算。

###### **编码实现**

- **ZookeeperConfig**

`Zookeeper`的配置信息

```java
@Data
public class ZookeeperConfig {

    /**
     * zk集群地址
     */
    private String zkAddress;

    /**
     * 注册服务目录
     */
    private String nodePath;

    /**
     * 分片的服务名
     */
    private String serviceName;

    /**
     * 分片总数
     */
    private Integer shardingCount;

    public ZookeeperConfig(String zkAddress, String nodePath, String serviceName, Integer shardingCount) {
        this.zkAddress = zkAddress;
        this.nodePath = nodePath;
        this.serviceName = "/" + serviceName;
        this.shardingCount = shardingCount;
    }

    /**
     * 等待重试的间隔时间的初始值.
     * 单位毫秒.
     */
    private int baseSleepTimeMilliseconds = 1000;

    /**
     * 等待重试的间隔时间的最大值.
     * 单位毫秒.
     */
    private int maxSleepTimeMilliseconds = 3000;

    /**
     * 最大重试次数.
     */
    private int maxRetries = 3;

    /**
     * 会话超时时间.
     * 单位毫秒.
     */
    private int sessionTimeoutMilliseconds;

    /**
     * 连接超时时间.
     * 单位毫秒.
     */
    private int connectionTimeoutMilliseconds;
}
```

- **InstanceInfo注册机器**

```java
@AllArgsConstructor
@EqualsAndHashCode()
public class InstanceInfo {

    private String ip;

    public String getInstance() {
        return ip;
    }
}
```

- **ZookeeperShardingService分片服务**

```java
@Slf4j
public class ZookeeperShardingService {

    public final Map<String, List<Integer>> caches = new HashMap<>(16);

    private final CuratorFramework client;

    private final ZookeeperConfig zkConfig;

    private final ShardingStrategy shardingStrategy;

    private final InstanceInfo instanceInfo;

    private static final CountDownLatch COUNT_DOWN_LATCH = new CountDownLatch(1);


    public ZookeeperShardingService(ZookeeperConfig zkConfig, ShardingStrategy shardingStrategy) {
        this.zkConfig = zkConfig;
        log.info("开始初始化zk, ip列表是: {}.", zkConfig.getZkAddress());
        CuratorFrameworkFactory.Builder builder = CuratorFrameworkFactory.builder()
                .connectString(zkConfig.getZkAddress())
                .retryPolicy(new ExponentialBackoffRetry(zkConfig.getBaseSleepTimeMilliseconds(), zkConfig.getMaxRetries(), zkConfig.getMaxSleepTimeMilliseconds()));
        if (0 != zkConfig.getSessionTimeoutMilliseconds()) {
            builder.sessionTimeoutMs(zkConfig.getSessionTimeoutMilliseconds());
        }
        if (0 != zkConfig.getConnectionTimeoutMilliseconds()) {
            builder.connectionTimeoutMs(zkConfig.getConnectionTimeoutMilliseconds());
        }
        this.shardingStrategy = shardingStrategy;
        HostInfo host = new HostInfo();
        this.instanceInfo = new InstanceInfo(host.getAddress());
        client = builder.build();
        client.getConnectionStateListenable().addListener(new ConnectionListener());
        client.start();
        try {
            COUNT_DOWN_LATCH.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        // 注册服务节点监听
        registerPathChildListener(zkConfig.getNodePath() + zkConfig.getServiceName(), new ChildrenPathListener());
        try {
            if (!client.blockUntilConnected(zkConfig.getMaxSleepTimeMilliseconds() * zkConfig.getMaxRetries(), TimeUnit.MILLISECONDS)) {
                client.close();
                throw new KeeperException.OperationTimeoutException();
            }
        } catch (final Exception ex) {
            ex.printStackTrace();
            throw new RuntimeException(ex);
        }
    }

    /**
     * 子节点监听器
     * @param nodePath 主节点
     * @param listener 监听器
     */
    private void registerPathChildListener(String nodePath, PathChildrenCacheListener listener) {
        try {
            // 1. 创建一个PathChildrenCache
            PathChildrenCache pathChildrenCache = new PathChildrenCache(client, nodePath, true);
            // 2. 添加目录监听器
            pathChildrenCache.getListenable().addListener(listener);
            // 3. 启动监听器
            pathChildrenCache.start(PathChildrenCache.StartMode.BUILD_INITIAL_CACHE);
        } catch (Exception e) {
            log.error("注册子目录监听器出现异常,nodePath:{}",nodePath,e);
            throw new RuntimeException(e);
        }
    }

    /**
     * 服务启动，注册zk节点
     * @throws Exception 异常
     */
    private void zkOp() throws Exception {
        // 是否存在ruubypay-sharding主节点
        if (null == client.checkExists().forPath(zkConfig.getNodePath())) {
            client.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath(zkConfig.getNodePath(), Hashing.sha1().hashString("sharding", Charsets.UTF_8).toString().getBytes());
        }
        // 是否存服务主节点
        if (null == client.checkExists().forPath(zkConfig.getNodePath() + zkConfig.getServiceName())) {
            // 创建服务主节点
            client.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath(zkConfig.getNodePath() + zkConfig.getServiceName());
        }
        // 检查是否存在临时节点
        if (null == client.checkExists().forPath(zkConfig.getNodePath() + zkConfig.getServiceName() + "/" + instanceInfo.getInstance())) {
            System.out.println(zkConfig.getNodePath() + zkConfig.getServiceName() +  "/" + instanceInfo.getInstance());
            // 创建临时节点
            client.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL).forPath(zkConfig.getNodePath() + zkConfig.getServiceName() +
                    "/" + instanceInfo.getInstance(), zkConfig.getShardingCount().toString().getBytes(StandardCharsets.UTF_8));
        }
        shardingFromZk();
    }

    /**
     * 从zk获取机器列表并进行分片
     * @throws Exception 异常
     */
    private void shardingFromZk() throws Exception {
        // 从 serviceName 节点下获取所有Ip列表
        final GetChildrenBuilder childrenBuilder = client.getChildren();
        final List<String> instanceList = childrenBuilder.watched().forPath(zkConfig.getNodePath() + zkConfig.getServiceName());
        List<InstanceInfo> res = new ArrayList<>();
        instanceList.forEach(s -> {
            res.add(new InstanceInfo(s));
        });
        Map<InstanceInfo, List<Integer>> shardingResult = shardingStrategy.sharding(res, zkConfig.getShardingCount());
        // 先清一遍缓存
        caches.clear();
        shardingResult.forEach((k, v) -> {
            caches.put(k.getInstance().split("-")[0], v);
        });
    }

    /**
     * zk连接监听
     */
    private class ConnectionListener implements ConnectionStateListener {

        @Override
        public void stateChanged(CuratorFramework client, ConnectionState newState) {
            if (newState == ConnectionState.CONNECTED || newState == ConnectionState.LOST || newState == ConnectionState.RECONNECTED) {
                try {
                    zkOp();
                } catch (Exception e) {
                    e.printStackTrace();
                    throw new RuntimeException(e);
                } finally {
                    COUNT_DOWN_LATCH.countDown();
                }
            }
        }
    }

    /**
     * 子节点监听
     */
    private class ChildrenPathListener implements PathChildrenCacheListener {

        @Override
        public void childEvent(CuratorFramework client, PathChildrenCacheEvent event) {
            PathChildrenCacheEvent.Type type = event.getType();
            if (PathChildrenCacheEvent.Type.CHILD_ADDED == type || PathChildrenCacheEvent.Type.CHILD_REMOVED == type) {
                try {
                    shardingFromZk();
                } catch (Exception e) {
                    e.printStackTrace();
                    throw new RuntimeException(e);
                }
            }
        }
    }
}
```

- **分片算法**

采用平均分配的算法

```java
public interface ShardingStrategy {

    Map<InstanceInfo, List<Integer>> sharding(final List<InstanceInfo> list, Integer shardingCount);
}

public class AverageAllocationShardingStrategy implements ShardingStrategy {

    @Override
    public Map<InstanceInfo, List<Integer>> sharding(List<InstanceInfo> list, Integer shardingCount) {
        if (list.isEmpty()) {
            return null;
        }
        Map<InstanceInfo, List<Integer>> result = shardingAliquot(list, shardingCount);
        addAliquant(list, shardingCount, result);
        return result;
    }

    private Map<InstanceInfo, List<Integer>> shardingAliquot(final List<InstanceInfo> instanceInfos, final int shardingTotalCount) {
        Map<InstanceInfo, List<Integer>> result = new LinkedHashMap<>(shardingTotalCount, 1);
        int itemCountPerSharding = shardingTotalCount / instanceInfos.size();
        int count = 0;
        for (InstanceInfo each : instanceInfos) {
            List<Integer> shardingItems = new ArrayList<>(itemCountPerSharding + 1);
            for (int i = count * itemCountPerSharding; i < (count + 1) * itemCountPerSharding; i++) {
                shardingItems.add(i);
            }
            result.put(each, shardingItems);
            count++;
        }
        return result;
    }

    private void addAliquant(final List<InstanceInfo> instanceInfos, final int shardingTotalCount, final Map<InstanceInfo, List<Integer>> shardingResults) {
        int aliquant = shardingTotalCount % instanceInfos.size();
        int count = 0;
        for (Map.Entry<InstanceInfo, List<Integer>> entry : shardingResults.entrySet()) {
            if (count < aliquant) {
                entry.getValue().add(shardingTotalCount / instanceInfos.size() * instanceInfos.size() + count);
            }
            count++;
        }
    }
}
```


## 总结

&nbsp; &nbsp; 基于`Zookeeper`和简单的平均分配算法实现了一个简单的分布式分片服务，该分片服务目前满足公司需求，因为其简单，所以不一定满足其他场景，针对其他场景还需考虑其他因素，该示例供参考。