# RocketMQ源码解析-Namesrv KV存储管理和路由信息管理核心源码分析


&nbsp; &nbsp; `Namesrv`为`rocketmq`的注册中心，主要提供的功能是KV存储管理和路由信息管理，该组件的核心代码不是很多，下面的源码分析过程也只针对`Namesrv`提供核心能展开分析



## KV存储管理模块 KVConfigManager

&nbsp; &nbsp; `KVConfigManager`提供key-value配置管理功能，下面代码就是配置管理的核心了，对应的解析可以参考源码注释，这部分代码比较简单，首先`org.apache.rocketmq.namesrv.NamesrvController`的`initialize()`会根据配置文件的位置家在配置文件中的配置内容并放到`configTable`中，然后其他组件在有配置变动的时候通过Netty客户端向Namesrv的Netty的服务端进行请求，然后`DefaultRequestProcessor`将请求分发到`KVConfigManager`的具体某一个处理方法上来实现配置的管理。

```
public class KVConfigManager {
    private static final InternalLogger log = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_LOGGER_NAME);

    private final NamesrvController namesrvController;

    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    // 配置哈希表
    private final HashMap<String/* Namespace */, HashMap<String/* Key */, String/* Value */>> configTable =
        new HashMap<String, HashMap<String, String>>();

    public KVConfigManager(NamesrvController namesrvController) {
        this.namesrvController = namesrvController;
    }

    /**
     * 加载配置
     */
    public void load() {
        String content = null;
        try {
            // 从文件中获取配置内容，this.namesrvController.getNamesrvConfig().getKvConfigPath()获取配置路径
            content = MixAll.file2String(this.namesrvController.getNamesrvConfig().getKvConfigPath());
        } catch (IOException e) {
            log.warn("Load KV config table exception", e);
        }
        if (content != null) {
            // 将配置反序列化，格式与configTable一致
            KVConfigSerializeWrapper kvConfigSerializeWrapper =
                KVConfigSerializeWrapper.fromJson(content, KVConfigSerializeWrapper.class);
            if (null != kvConfigSerializeWrapper) {
                // 将配置放到配置哈希表中
                this.configTable.putAll(kvConfigSerializeWrapper.getConfigTable());
                log.info("load KV config table OK");
            }
        }
    }

    /**
     * 加配置
     * @param namespace
     * @param key
     * @param value
     */
    public void putKVConfig(final String namespace, final String key, final String value) {
        try {
            this.lock.writeLock().lockInterruptibly();
            try {
                // 从配置表中根据namespace获取配置
                HashMap<String, String> kvTable = this.configTable.get(namespace);
                if (null == kvTable) {
                    // 配置不存在，创建一个新的配置，一个key-value配置map
                    kvTable = new HashMap<String, String>();
                    // 将新的配置放到配置表中
                    this.configTable.put(namespace, kvTable);
                    log.info("putKVConfig create new Namespace {}", namespace);
                }
                // 更新配置，这里是无论有无配置都会更新
                final String prev = kvTable.put(key, value);
                if (null != prev) {
                    log.info("putKVConfig update config item, Namespace: {} Key: {} Value: {}",
                        namespace, key, value);
                } else {
                    log.info("putKVConfig create new config item, Namespace: {} Key: {} Value: {}",
                        namespace, key, value);
                }
            } finally {
                this.lock.writeLock().unlock();
            }
        } catch (InterruptedException e) {
            log.error("putKVConfig InterruptedException", e);
        }
        // 操作配置文件
        this.persist();
    }

    /**
     * 配置文件的配置更新备份操作
     */
    public void persist() {
        try {
            this.lock.readLock().lockInterruptibly();
            try {
                KVConfigSerializeWrapper kvConfigSerializeWrapper = new KVConfigSerializeWrapper();
                kvConfigSerializeWrapper.setConfigTable(this.configTable);
                // 得到当前配置的内容
                String content = kvConfigSerializeWrapper.toJson();

                if (null != content) {
                    // 配置持久化的操作，主要是备份就文件，创建新文件
                    MixAll.string2File(content, this.namesrvController.getNamesrvConfig().getKvConfigPath());
                }
            } catch (IOException e) {
                log.error("persist kvconfig Exception, "
                    + this.namesrvController.getNamesrvConfig().getKvConfigPath(), e);
            } finally {
                this.lock.readLock().unlock();
            }
        } catch (InterruptedException e) {
            log.error("persist InterruptedException", e);
        }

    }

    /**
     * 删除配置
     * @param namespace
     * @param key
     */
    public void deleteKVConfig(final String namespace, final String key) {
        try {
            this.lock.writeLock().lockInterruptibly();
            try {
                // 根据namespace获取配置
                HashMap<String, String> kvTable = this.configTable.get(namespace);
                if (null != kvTable) {
                    // 移除配置
                    String value = kvTable.remove(key);
                    log.info("deleteKVConfig delete a config item, Namespace: {} Key: {} Value: {}",
                        namespace, key, value);
                }
            } finally {
                this.lock.writeLock().unlock();
            }
        } catch (InterruptedException e) {
            log.error("deleteKVConfig InterruptedException", e);
        }
        // 配置文件操作
        this.persist();
    }

    /**
     * 根据namespace将配置 转化成byte数组，用于Nstty的序列化
     * @param namespace
     * @return
     */
    public byte[] getKVListByNamespace(final String namespace) {
        try {
            this.lock.readLock().lockInterruptibly();
            try {
                HashMap<String, String> kvTable = this.configTable.get(namespace);
                if (null != kvTable) {
                    KVTable table = new KVTable();
                    table.setTable(kvTable);
                    return table.encode();
                }
            } finally {
                this.lock.readLock().unlock();
            }
        } catch (InterruptedException e) {
            log.error("getKVListByNamespace InterruptedException", e);
        }

        return null;
    }

    /**
     * 根据namespace和key获取具体配置值
     * @param namespace
     * @param key
     * @return
     */
    public String getKVConfig(final String namespace, final String key) {
        try {
            this.lock.readLock().lockInterruptibly();
            try {
                HashMap<String, String> kvTable = this.configTable.get(namespace);
                if (null != kvTable) {
                    return kvTable.get(key);
                }
            } finally {
                this.lock.readLock().unlock();
            }
        } catch (InterruptedException e) {
            log.error("getKVConfig InterruptedException", e);
        }

        return null;
    }

    /**
     * 打印所有配置，NamesrvController在initialize的时候会启动一个线程去定时的打印配置
     */
    public void printAllPeriodically() {
        try {
            this.lock.readLock().lockInterruptibly();
            try {
                log.info("--------------------------------------------------------");

                {
                    log.info("configTable SIZE: {}", this.configTable.size());
                    Iterator<Entry<String, HashMap<String, String>>> it =
                        this.configTable.entrySet().iterator();
                    while (it.hasNext()) {
                        Entry<String, HashMap<String, String>> next = it.next();
                        Iterator<Entry<String, String>> itSub = next.getValue().entrySet().iterator();
                        while (itSub.hasNext()) {
                            Entry<String, String> nextSub = itSub.next();
                            log.info("configTable NS: {} Key: {} Value: {}", next.getKey(), nextSub.getKey(),
                                nextSub.getValue());
                        }
                    }
                }
            } finally {
                this.lock.readLock().unlock();
            }
        } catch (InterruptedException e) {
            log.error("printAllPeriodically InterruptedException", e);
        }
    }
}
```

## RouteInfoManager路由信息管理模块

### RouteInfoManager管理内容

&nbsp; &nbsp; `RouteInfoManager`提供了路由信息管理，下面的源码分析了`RouteInfoManager`的核心能力，会忽略一些其他的非核心代码，这部分代码其他的模块基本都会提及

`RouteInfoManager`所管理的内容如下：
```
    // topic管理
    private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;
    // broker管理 brokerName-Broker信息
    private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
    // 集群管理
    private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
    // broker地址和 存活的broker信息
    private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
    private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
```

从路由表中删除`topic`
```
public void deleteTopic(final String topic)
```

获取所有`topic`列表，为byte数组，用于Netty网络传输序列化
```
    public byte[] getAllTopicList() {
        TopicList topicList = new TopicList();
        try {
            try {
                this.lock.readLock().lockInterruptibly();
                topicList.getTopicList().addAll(this.topicQueueTable.keySet());
            } finally {
                this.lock.readLock().unlock();
            }
        } catch (Exception e) {
            log.error("getAllTopicList Exception", e);
        }

        return topicList.encode();
    }
```

### 处理Broker的注册

注册`Broker`到`Namesrv`，在`org.apache.rocketmq.broker.BrokerController`的`start()`方法中Broker发起到Namesrv的注册，`registerBrokerAll()`方法通过`Netty`客户端将`Broker`路由信息发送到`Namesrv`的`Netty`服务端，`Namesrv`的`Netty`服务端接受到请求后，交给`DefaultRequestProcessor`的`processRequest`处理，最后委托给了`RouteInfoManager`的r`egisterBroker`方法，填充或者更新路由信息。注册代码如下：

- org.apache.rocketmq.broker.BrokerController的start方法注册Broker

```
        if (!messageStoreConfig.isEnableDLegerCommitLog()) {
            startProcessorByHa(messageStoreConfig.getBrokerRole());
            handleSlaveSynchronize(messageStoreConfig.getBrokerRole());
            // 第一次注册Broker路由到Namesrv
            this.registerBrokerAll(true, false, true);
        }
        // 向Namesrv 30s定时发送心跳包
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                try {
                    BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
                } catch (Throwable e) {
                    log.error("registerBrokerAll Exception", e);
                }
            }
        }, 1000 * 10, Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)), TimeUnit.MILLISECONDS);
```

- org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager的registerBroker注册Broker路由信息

```
    /**
     * 注册Broker
     * @param clusterName 集群名
     * @param brokerAddr broker address
     * @param brokerName broker名
     * @param brokerId broker ID
     * @param haServerAddr 高可用服务地址
     * @param topicConfigWrapper topic配置封装
     * @param filterServerList
     * @param channel netty channel
     * @return
     */
    public RegisterBrokerResult registerBroker(
        final String clusterName,
        final String brokerAddr,
        final String brokerName,
        final long brokerId,
        final String haServerAddr,
        final TopicConfigSerializeWrapper topicConfigWrapper,
        final List<String> filterServerList,
        final Channel channel) {
        RegisterBrokerResult result = new RegisterBrokerResult();
        try {
            try {
                this.lock.writeLock().lockInterruptibly();
                // 集群中broker名集合
                Set<String> brokerNames = this.clusterAddrTable.get(clusterName);
                if (null == brokerNames) {
                    brokerNames = new HashSet<String>();
                    // 不存在该集群，初始化一个空的集群
                    this.clusterAddrTable.put(clusterName, brokerNames);
                }
                // 向集群的broker集合中添加要注册的broker
                brokerNames.add(brokerName);

                boolean registerFirst = false;
                //  根据broker获取Broker的元数据
                BrokerData brokerData = this.brokerAddrTable.get(brokerName);
                if (null == brokerData) {
                    // 第一次注册
                    registerFirst = true;
                    // 初始化broker的原数据
                    brokerData = new BrokerData(clusterName, brokerName, new HashMap<Long, String>());
                    // 向管理器管理的broker哈希表中添加broker管理信息
                    this.brokerAddrTable.put(brokerName, brokerData);
                }
                // broker集群的地址集
                Map<Long, String> brokerAddrsMap = brokerData.getBrokerAddrs();
                //Switch slave to master: first remove <1, IP:PORT> in namesrv, then add <0, IP:PORT>
                //The same IP:PORT must only have one record in brokerAddrTable
                // 从broker转到主broker 先在namesrv中移除<1, IP:PORT>，然后添加<0, IP:PORT>，相同的IP:PORT只能有一个记录在brokerAddrTable中
                Iterator<Entry<Long, String>> it = brokerAddrsMap.entrySet().iterator();
                while (it.hasNext()) {
                    Entry<Long, String> item = it.next();
                    if (null != brokerAddr && brokerAddr.equals(item.getValue()) && brokerId != item.getKey()) {
                        it.remove();
                    }
                }

                String oldAddr = brokerData.getBrokerAddrs().put(brokerId, brokerAddr);
                registerFirst = registerFirst || (null == oldAddr);
                // 有topic并且是主broker
                if (null != topicConfigWrapper
                    && MixAll.MASTER_ID == brokerId) {
                    // topic的是否变更过，或者是第一次注册
                    if (this.isBrokerTopicConfigChanged(brokerAddr, topicConfigWrapper.getDataVersion())
                        || registerFirst) {
                        ConcurrentMap<String, TopicConfig> tcTable =
                            topicConfigWrapper.getTopicConfigTable();
                        if (tcTable != null) {
                            for (Map.Entry<String, TopicConfig> entry : tcTable.entrySet()) {
                                // 如果第一次注册，创建topic，否则更新
                                this.createAndUpdateQueueData(brokerName, entry.getValue());
                            }
                        }
                    }
                }
                // 这里返回null，第一次注册并且这个brokerAddr是第一个put
                BrokerLiveInfo prevBrokerLiveInfo = this.brokerLiveTable.put(brokerAddr,
                    new BrokerLiveInfo(
                        System.currentTimeMillis(),
                        topicConfigWrapper.getDataVersion(),
                        channel,
                        haServerAddr));
                if (null == prevBrokerLiveInfo) {
                    log.info("new broker registered, {} HAServer: {}", brokerAddr, haServerAddr);
                }

                if (filterServerList != null) {
                    if (filterServerList.isEmpty()) {
                        this.filterServerTable.remove(brokerAddr);
                    } else {
                        this.filterServerTable.put(brokerAddr, filterServerList);
                    }
                }
                // 高可用集群环境注册，将 主的地址和高可用的地址设置到 注册结果中
                if (MixAll.MASTER_ID != brokerId) {
                    String masterAddr = brokerData.getBrokerAddrs().get(MixAll.MASTER_ID);
                    if (masterAddr != null) {
                        BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.get(masterAddr);
                        if (brokerLiveInfo != null) {
                            result.setHaServerAddr(brokerLiveInfo.getHaServerAddr());
                            result.setMasterAddr(masterAddr);
                        }
                    }
                }
            } finally {
                this.lock.writeLock().unlock();
            }
        } catch (Exception e) {
            log.error("registerBroker Exception", e);
        }

        return result;
    }
```

### 处理Broker的注销

当`Broker`停止时调用`BrokerController`的`shutdown`方法，该方法会触发`Broker`的注销，实际上就是就是通过`Netty`的客户端向`Namesrv`的`Netty`的服务端发送注销请求，`Namesrv`的注销逻辑代码如下：

```
    /**
     * 注销broker
     * @param clusterName
     * @param brokerAddr
     * @param brokerName
     * @param brokerId
     */
    public void unregisterBroker(
        final String clusterName,
        final String brokerAddr,
        final String brokerName,
        final long brokerId) {
        try {
            try {
                this.lock.writeLock().lockInterruptibly();
                // 从brokerLiveTable中移除broker
                BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.remove(brokerAddr);
                log.info("unregisterBroker, remove from brokerLiveTable {}, {}",
                    brokerLiveInfo != null ? "OK" : "Failed",
                    brokerAddr
                );
                // 从filterServerTable中移除broker，FilterServer作用以后说
                this.filterServerTable.remove(brokerAddr);

                boolean removeBrokerName = false;
                BrokerData brokerData = this.brokerAddrTable.get(brokerName);
                if (null != brokerData) {
                    // 如果存在broker，根据brokerId从broker的哈希表中移除broker
                    String addr = brokerData.getBrokerAddrs().remove(brokerId);
                    log.info("unregisterBroker, remove addr from brokerAddrTable {}, {}",
                        addr != null ? "OK" : "Failed",
                        brokerAddr
                    );
                    // broker都移除了，从brokerAddrTable移除broker
                    if (brokerData.getBrokerAddrs().isEmpty()) {
                        this.brokerAddrTable.remove(brokerName);
                        log.info("unregisterBroker, remove name from brokerAddrTable OK, {}",
                            brokerName
                        );

                        removeBrokerName = true;
                    }
                }

                if (removeBrokerName) {
                    Set<String> nameSet = this.clusterAddrTable.get(clusterName);
                    if (nameSet != null) {
                        // 从集群中移除broker
                        boolean removed = nameSet.remove(brokerName);
                        log.info("unregisterBroker, remove name from clusterAddrTable {}, {}",
                            removed ? "OK" : "Failed",
                            brokerName);
                        // 集群中不存在broker
                        if (nameSet.isEmpty()) {
                            // 移除集群
                            this.clusterAddrTable.remove(clusterName);
                            log.info("unregisterBroker, remove cluster from clusterAddrTable {}",
                                clusterName
                            );
                        }
                    }
                    // 移除这个与这个broker相关的topic
                    this.removeTopicByBrokerName(brokerName);
                }
            } finally {
                this.lock.writeLock().unlock();
            }
        } catch (Exception e) {
            log.error("unregisterBroker Exception", e);
        }
    }
```

## 路由删除

在`org.apache.rocketmq.namesrv.NamesrvController`的`initialize()`方法中，启动了一个定时任务，该任务每隔10s中去检查一次`brokerLiveTable`列表，当连续120s没有收到来自Broker心跳的时候Namesrv就会移除该Broker的路由信息，源码如下：

```
    // 每隔10s检测Broker路由列表中Broker的存活情况
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            NamesrvController.this.routeInfoManager.scanNotActiveBroker();
        }
    }, 
```

```
    /**
     * 扫描brokerLiveTable，连续120s没有收到心跳就会移除broker路由
     */
    public void scanNotActiveBroker() {
        Iterator<Entry<String, BrokerLiveInfo>> it = this.brokerLiveTable.entrySet().iterator();
        while (it.hasNext()) {
            Entry<String, BrokerLiveInfo> next = it.next();
            long last = next.getValue().getLastUpdateTimestamp();
            if ((last + BROKER_CHANNEL_EXPIRED_TIME) < System.currentTimeMillis()) {
                RemotingUtil.closeChannel(next.getValue().getChannel());
                //移除broker路由
                it.remove();
                log.warn("The broker channel expired, {} {}ms", next.getKey(), BROKER_CHANNEL_EXPIRED_TIME);
                this.onChannelDestroy(next.getKey(), next.getValue().getChannel());
            }
        }
    }
```

## 总结

`RouterInfoManager`的部分代码先分析到这里，除了Broker的注册注销探活等能力外，还有所管理的内容的序列化等功能，这些能力会在以后的的源码分析中提及，下一片文章主要分析一下`NameServer`作为Netty服务的的启动流程，事件处理的源码。