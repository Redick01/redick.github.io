# Google Guava Cache在项目中应用

## 目标

- 使用缓存背景
- 技术选型
- 使用Guava Cache构建缓存
- 缓存测试


## 使用缓存背景

&nbsp; &nbsp; 接入缓存的项目是一个交易处理服务，主要流程就是从`RocketMQ`中消费初始交易，并根据业务规则进行交易的处理；原本，交易中的用户相关信息都是没有校验的，以交易中的数据为准的，但是随着交易量的增加，以及第三方商户的接入，交易中的用户数据变得不再可靠，面对此问题就需要进行用户信息校验，由于我们架构是微服务，用户信息在用户中心，最直观的方式就是消费每条交易的时候调用用户中心的接口来校验用户信息，但是业务高峰期交易量非常大，这就会导致用户中心的压力增大，鉴于此我们引入了缓存。

## 技术选型

&nbsp; &nbsp; 面对技术选型，无非是集中式缓存和本地缓存两种，集中式缓存可以使用`redis`，`memcache`等，鉴于该系统此前使用`redis`进行交易处理，并且资源有限，所以在不考虑引入其他中间件的情况下我们就放弃了集中式缓存，转而选择了本地缓存，这其中也不是盲目的就进行了选型的，首先交易量可预测，并且除了某些特定的时间点交易量相对比较平滑，其次用户量也是可预测的，由于已经过了业务初期用户增加的量也是很平滑的，还有个最重要的原因是我们系统的用户有个特点，那就是热点用户，基本上每天产生的交易，可能都来自那么一撮（百万级）的人。

&nbsp; &nbsp; 本地缓存，我列出以下实现

- 自定义全局的`Map`
- Guava Cache

&nbsp; &nbsp; 我在平时用到一个冷数据的时候使用的`Map`，因为冷数据都是些配置数据或参数数据，是不会变的，所以在考虑过期，更新缓存的时候都是可预知的，并且数据量也不多且是固定的，非常简单；但是在缓存用户数据的时候考虑的就相对多一些，比如用户数据量很大，缓存数据有过期，缓存的数据有热点的特点，所以如果在自定义`map`的话就需要自己实现一大堆的缓存策略，鉴于此，直接选择使用Guava Cache

&nbsp; &nbsp; 刚才提到过，我们用户的是有热点用户这么一说的，也就是基本上每天都产生交易的用户就那么一堆人，所以在使用缓存的时候我们就要考虑以下几个点：
- 产生交易的用户量的估值
- 缓存的淘汰策略

&nbsp; &nbsp; 产生交易的用户量的估值的目的是设置缓存的大小，因为用户量实际上很大，所以不能一股脑的都将用户数据缓存起来；缓存的淘汰策略针对的就是热点用户，根据交易量估算出活跃用户量为100w，所以我设置缓存的大小为100w，并且设置过期策略为LRU（最近最少使用）

## 使用Guava Cache构建缓存

```
@Slf4j
public final class CacheC {

    /**
     * 清除缓存容量
     * 当缓存容量达到配置最大容量时，Guava Cache会基于LRU算法清除缓存
     */
    private static final int MAX_NUM = 1000000;

    /**
     * 交通卡号和商户号缓存
     * key-卡号 value-值
     */
    @SuppressWarnings("UnstableApiUsage")
    private static Cache<String, String> CACHE;

    /**
     * 添加缓存
     * @param cardNum 交通卡号
     * @param merchantNo 商户号
     * @return 商户号
     */
    @SuppressWarnings("UnstableApiUsage")
    public static String addCache(String cardNum, String merchantNo) {
        if (Objects.isNull(CACHE)) {
            CACHE = CacheBuilder.newBuilder()
                    .maximumSize(MAX_NUM)
                    .concurrencyLevel(8)
                    .build();
        }
        log.info(LogUtil.marker(cardNum), "添加卡号缓存");
        CACHE.put(cardNum, merchantNo);
        return merchantNo;
    }

    /**
     * 从缓存中获取商户号
     * @param cardNum 卡号
     * @return 商户号
     */
    @SuppressWarnings("UnstableApiUsage")
    public static String getCache(String cardNum) {
        if (Objects.isNull(CACHE)) {
            return null;
        }
        String value = CACHE.getIfPresent(cardNum);
        if (StringUtils.isBlank(value)) {
            log.info(LogUtil.marker(cardNum), "未命中缓存");
        } else {
            log.info(LogUtil.marker(value), "命中缓存");
        }
        return value;
    }
}
```

## 缓存测试

@RunWith(MockitoJUnitRunner.class)
public class CacheCTest {

    private static final int count = 10000;

    @Test
    public void testCacheC() {
        for (int i = 0; i < count; i++) {
            CacheC.addCache(i + "", i + "");
        }

        for (int i = 0; i < count; i++) {
            String value = CacheC.getCache(1 + "");
            if (StringUtils.isBlank(value)) {
                System.out.println("没走缓存");
            } else {
                System.out.println("走了缓存");
            }
        }
    }
}

## 总结

   Guava Cache使用起来非常简单，并且提供了缓存淘汰策略的实现，并且它还提供了一些其他非常有意思的功能，例如统计等，下篇文章我们来看一下Guava Cache的其他的特性