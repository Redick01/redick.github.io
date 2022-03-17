# 开源动态线程池dynamic-tp支持zookeeper配置中心 <!-- {docsify-ignore-all} -->


## 前言

&nbsp; &nbsp; `dynamic-tp`是一个轻量级的动态线程池插件，它是一个基于配置中心的动态线程池，线程池的参数可以通过配置中心配置进行动态的修改，在配置中心的支持上最开始的时候支持`Nacos`和`Apollo`，由于笔者公司用的配置中心是`Zookeeper`，所以就想着扩展支持`Zookeeper`，在了解源码支持发现`dynamic-tp`的扩展能力做的很好，提供了扩展接口，只要我开发对应的配置中心模块即可，最终笔者实现了`Zookeeper`的支持并贡献到社区。接下来我通过源码解析方式介绍下`Zookeeper`配置中心的接入。


## 配置刷新

&nbsp; &nbsp; `dynamic-tp`提供了一个刷新配置的接口`Refresher`，抽象类`AbstractRefresher`实现刷新配置接口的刷新配置方法`refresh`，该方法能根据配置类型内容和配置解析配置并刷新动态线程池的相关配置，由`DtpRegistry`负责刷新线程池配置，事件发布订阅模式操作Web容器参数，代码如下：

```java
public interface Refresher {

    /**
     * Refresh with specify content.
     * @param content content
     * @param fileType file type
     */
    void refresh(String content, ConfigFileTypeEnum fileType);
}
```

```java
@Slf4j
public abstract class AbstractRefresher implements Refresher {

    @Resource
    private DtpProperties dtpProperties;

    @Resource
    private ApplicationEventMulticaster applicationEventMulticaster;

    @Override
    public void refresh(String content, ConfigFileTypeEnum fileTypeEnum) {

        if (StringUtils.isBlank(content) || Objects.isNull(fileTypeEnum)) {
            return;
        }

        try {
            // 根据配置内容和配置类型将配置内容转成Map
            val prop = ConfigHandler.getInstance().parseConfig(content, fileTypeEnum);
            doRefresh(prop);
        } catch (IOException e) {
            log.error("DynamicTp refresh error, content: {}, fileType: {}",
                    content, fileTypeEnum, e);
        }
    }

    private void doRefresh(Map<Object, Object> properties) {
        // 将Map中的配置转换成DtpProperties
        ConfigurationPropertySource sources = new MapConfigurationPropertySource(properties);
        Binder binder = new Binder(sources);
        ResolvableType type = ResolvableType.forClass(DtpProperties.class);
        Bindable<?> target = Bindable.of(type).withExistingValue(dtpProperties);
        binder.bind(MAIN_PROPERTIES_PREFIX, target);
        // 刷新动态线程池配置
        DtpRegistry.refresh(dtpProperties);
        // 发布刷新实现，该事件用于控制Web容器线程池参数控制
        publishEvent();
    }

    private void publishEvent() {
        RefreshEvent event = new RefreshEvent(this, dtpProperties);
        applicationEventMulticaster.multicastEvent(event);
    }
}
```

## Zookeeper配置中心接入扩展实现

&nbsp; &nbsp; 基于`AbstractRefresher`就可以实现`Zookeeper`配置中心的扩展了，`Zookeeper`的扩展实现继承`AbstractRefresher`，`Zookeeper`的扩展实现只需要监听配置中心的配置变更即可拿到配置内容，然后通过`refresh`刷新配置即可。代码如下：

&nbsp; &nbsp; `ZookeeperRefresher`继承`AbstractRefresher`，实现`InitializingBean`，`afterPropertiesSet`方法逻辑从配置`DtpProperties`获取`Zookeeper`的配置信息，`CuratorFrameworkFactory`创建客户端，设置监听器，这里有两种监听器，一个是连接监听`ConnectionStateListener`，一个是节点变动监听`CuratorListener`，出发监听后`loadNode`负责从`Zookeeper`获取配置文件配置并组装配置内容，然后通过`refresh`刷新配置，注意，`Zookeeper`配置目前配置类型仅支持`properties`。

```java
@Slf4j
public class ZookeeperRefresher extends AbstractRefresher implements InitializingBean {

    @Resource
    private DtpProperties dtpProperties;

    private CuratorFramework curatorFramework;

    @Override
    public void afterPropertiesSet() throws Exception {

        DtpProperties.Zookeeper zookeeper = dtpProperties.getZookeeper();
        curatorFramework = CuratorFrameworkFactory.newClient(zookeeper.getZkConnectStr(),
                new ExponentialBackoffRetry(1000, 3));
        String nodePath = ZKPaths.makePath(ZKPaths.makePath(zookeeper.getRootNode(),
                zookeeper.getConfigVersion()), zookeeper.getNode());

        final ConnectionStateListener connectionStateListener = (client, newState) -> {
            if (newState == ConnectionState.CONNECTED || newState == ConnectionState.RECONNECTED) {
                loadNode(nodePath);
            }};

        final CuratorListener curatorListener = (client, curatorEvent) -> {
            final WatchedEvent watchedEvent = curatorEvent.getWatchedEvent();
            if (null != watchedEvent) {
                switch (watchedEvent.getType()) {
                    case NodeChildrenChanged:
                    case NodeDataChanged:
                        loadNode(nodePath);
                        break;
                    default:
                        break;
                }
            }};
        curatorFramework.getConnectionStateListenable().addListener(connectionStateListener);
        curatorFramework.getCuratorListenable().addListener(curatorListener);
        curatorFramework.start();
        log.info("DynamicTp refresher, add listener success, nodePath: {}", nodePath);
    }


    /**
     * load config and refresh
     * @param nodePath config path
     */
    public void loadNode(String nodePath) {
        try {
            final GetChildrenBuilder childrenBuilder = curatorFramework.getChildren();
            final List<String> children = childrenBuilder.watched().forPath(nodePath);
            StringBuilder content = new StringBuilder();
            children.forEach(c -> {
                String n = ZKPaths.makePath(nodePath, c);
                final String nodeName = ZKPaths.getNodeFromPath(n);
                final GetDataBuilder data = curatorFramework.getData();
                String value = "";
                try {
                    value = new String(data.watched().forPath(n), StandardCharsets.UTF_8);
                } catch (Exception e) {
                    log.error("zk config value watched exception.", e);
                }
                content.append(nodeName).append("=").append(value).append("\n");
            });
            refresh(content.toString(), ConfigFileTypeEnum.PROPERTIES);
        } catch (Exception e) {
            log.error("load zk node error, nodePath is {}", nodePath, e);
        }
    }
}
```

## 总结

&nbsp; &nbsp; `dynamic-tp`对应支持配置中心的扩展能力做的非常好，笔者通过`Zookeeper`客户端`CuratorFramework`设置监听的方式进行接入，主要监听`CuratorFramework`客户端连接建立和断开的事件和节点变动的事件实现了动态线程池参数的更新。