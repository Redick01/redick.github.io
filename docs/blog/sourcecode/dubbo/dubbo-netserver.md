# Dubbo服务启动-底层通信 <!-- {docsify-ignore-all} -->


## Dubbo协议打开服务器

&nbsp; &nbsp; 书接上回[Dubbo Provider发布服务](../../sourcecode/dubbo/dubbo-export-service.md)Provider会通过`RegistryProtocol#export`注册服务，通过`DubboProtocol`发布服务吗，`DubboProtocol`发布服务时会打开服务器，`DubboProtocol`中有一个`serviceMap`，存储ip:port和`ExchangeServer`的映射关系，刚开始创建的时候会检查`serviceMap`，如果key对应的`ExchangeServer`不存在会调用`createServer`创建Server，代码如下：

```java
    private void openServer(URL url) {
        // ip端口
        String key = url.getAddress();
        // 客户端发布用于服务器的服务
        boolean isServer = url.getParameter(IS_SERVER_KEY, true);
        if (isServer) {
            // 从serviceMap查找Server
            ProtocolServer server = serverMap.get(key);
            // serverMap中没有Server，这里使用两次查找
            if (server == null) {
                synchronized (this) {
                    server = serverMap.get(key);
                    if (server == null) {
                        // 创建Server并放到serviceMap中
                        serverMap.put(key, createServer(url));
                    }
                }
            } else {
                // server supports reset, use together with override
                server.reset(url);
            }
        }
    }
    
    private ProtocolServer createServer(URL url) {
        // url
        url = URLBuilder.from(url)
                // send readonly event when server closes, it's enabled by default
                .addParameterIfAbsent(CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString())
                // enable heartbeat by default
                .addParameterIfAbsent(HEARTBEAT_KEY, String.valueOf(DEFAULT_HEARTBEAT))
                .addParameter(CODEC_KEY, DubboCodec.NAME)
                .build();
        // 默认服务器采用netty
        String str = url.getParameter(SERVER_KEY, DEFAULT_REMOTING_SERVER);

        if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
            throw new RpcException("Unsupported server type: " + str + ", url: " + url);
        }

        ExchangeServer server;
        try {
            server = Exchangers.bind(url, requestHandler);
        } catch (RemotingException e) {
            throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
        }

        str = url.getParameter(CLIENT_KEY);
        if (str != null && str.length() > 0) {
            // 支持的传输层类型
            Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions();
            if (!supportedTypes.contains(str)) {
                throw new RpcException("Unsupported client type: " + str);
            }
        }

        return new DubboProtocolServer(server);
    }   
```

### 打开服务器流程

底层通信分为两层，信息交换层和传输层，传输层使用netty等实现。

```
|- Exchanger 信息交换层  header
	|- Transporters 传输层 netty
    	|- NettyServer
        	|- HeaderExchangeServer
```

1. 创建服务器时，`dubbo:protocol`标签中设置服务器实现类型，比如:dubbo协议的mina,netty等,http协议的jetty,servlet等，dubbo协议默认的服务器是netty。
2. `Exchangers.bind`会根据url启动服务器，首先根据url通过SPI机制获取`Exchanger`，默认的`Exchanger`是`header`，对应的是`HeaderExchanger`，`Exchanger`是信息交换层。
3. 获取到`Exchanger`调用其`bind`方法，会使用`Transporters`的`bind`。代码如下

```java
    /**
    * DubboProtocol bind方法首先根据url通过SPI机制获取`Exchanger`，默认的`Exchanger`是`header`
    *
    *
    */
    public static ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        if (handler == null) {
            throw new IllegalArgumentException("handler == null");
        }
        url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
        return getExchanger(url).bind(url, handler);
    }

    public static Exchanger getExchanger(URL url) {
        String type = url.getParameter(Constants.EXCHANGER_KEY, Constants.DEFAULT_EXCHANGER);
        return getExchanger(type);
    }

    public static Exchanger getExchanger(String type) {
        return ExtensionLoader.getExtensionLoader(Exchanger.class).getExtension(type);
    }

/**
 * DefaultMessenger
 *
 *
 */
public class HeaderExchanger implements Exchanger {

    public static final String NAME = "header";

    @Override
    public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))), true);
    }

    @Override
    public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }

}
```

4. `Transporters`的`bind`方法会`getTransporter()`获取`Transporter`，这里也是基于SPI机制，默认是`netty`

```java
    public static RemotingServer bind(URL url, ChannelHandler... handlers) throws RemotingException {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        if (handlers == null || handlers.length == 0) {
            throw new IllegalArgumentException("handlers == null");
        }
        ChannelHandler handler;
        if (handlers.length == 1) {
            handler = handlers[0];
        } else {
            handler = new ChannelHandlerDispatcher(handlers);
        }
        return getTransporter().bind(url, handler);
    }

    public static Transporter getTransporter() {
        return ExtensionLoader.getExtensionLoader(Transporter.class).getAdaptiveExtension();
    }
```