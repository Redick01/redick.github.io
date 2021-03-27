# 单元测试之-embedded-redisserver

## 目标

- 单元测试redis执行的lua脚本
- 技术选型之embedded-redisserver
- 我在项目单元测试中的使用

## 单元测试redis执行的lua脚本

&nbsp; &nbsp; 最近在参加开源项目`soul`网关时，基于redis实现了一个限流的漏桶算法，在提交pr的过程中发现`soul`网关的限流插件中没有对每个算法Lua脚本的单元测试，换句话说，贡献者提供了一个算法的实现，java代码我们都可以单元测试，但是真正执行限流逻辑的与与redis交互执行的lua脚本时没有单元测试的，这就导致提交的lua脚本不确定运行结果是否符合预计，基于这个打算对lua脚本进行单元测试。

## 技术选型之embedded-redisserver

&nbsp; &nbsp; 要对lua脚本进行单元测试，那么就需要一个redis服务端的环境，但是在单元测试中这是不可能的，所以要尝试去`mock`一个redis环境，针对这个需求选用了embedded-redis，该组件是可以在java程序内部嵌入一个redis。

## 我在项目单元测试中的使用

- **pom**

```
        <dependency>
            <groupId>com.github.kstyrc</groupId>
            <artifactId>embedded-redis</artifactId>
            <version>0.6</version>
            <scope>test</scope>
        </dependency>
```

- **在单元测试中构建一个RedisServer**

&nbsp; &nbsp; `@BeforeClass`意思是在执行单元测试之前执行，方法逻辑是构建一个`RedisServer`，端口号63792，最大内存`64m`，这个要注意如果不设置最大内存，在`CI`编译项目时可能会导致连续的内存空间不足从而无法完成CI的构建，报错内容`Can’t start redis server. Check logs for details.`，至此，一个内嵌的Redis就这样集成到了单元测试中了。

```
    private static RedisServer redisServer;

    @BeforeClass
    public static void startup() {
        redisServer = RedisServer.builder()
                .port(63792)
                .setting("maxmemory 64m")
                .build();
        redisServer.start();
        RateLimiterPluginDataHandler handler = new RateLimiterPluginDataHandler();
        RateLimiterConfig config = new RateLimiterConfig();
        config.setUrl("127.0.0.1:63792");
        PluginData pluginData = PluginData.builder()
                .enabled(true)
                .config(GsonUtils.getInstance().toJson(config))
                .build();

        handler.handlerPlugin(pluginData);
    }

    @AfterClass
    public static void end() {
        redisServer.stop();
    }
```

- **单元测试**

- - Lua脚本

```
-- current key
local leaky_bucket_key = KEYS[1]
-- last update key
local last_bucket_key = KEYS[2]
-- capacity
local capacity = tonumber(ARGV[2])
-- the rate of leak water
local rate = tonumber(ARGV[1])
-- request count
local requested = tonumber(ARGV[4])
-- current timestamp
local now = tonumber(ARGV[3])
-- the key life time
local key_lifetime = math.ceil((capacity / rate) + 1)


-- the yield of water in the bucket default 0
local key_bucket_count = tonumber(redis.call("GET", leaky_bucket_key)) or 0

-- the last update time default now
local last_time = tonumber(redis.call("GET", last_bucket_key)) or now

-- the time difference
local millis_since_last_leak = now - last_time

-- the yield of water had lasted
local leaks = millis_since_last_leak * rate

if leaks > 0 then
    -- clean up the bucket
    if leaks >= key_bucket_count then
        key_bucket_count = 0
    else
        -- compute the yield of water in the bucket
        key_bucket_count = key_bucket_count - leaks
    end
    last_time = now
end

-- is allowed pass default not allow
local is_allow = 0

local new_bucket_count = key_bucket_count + requested
-- allow
if new_bucket_count <= capacity then
    is_allow = 1
else
    -- not allow
    return {is_allow, new_bucket_count}
end

-- update the key bucket water yield
redis.call("SETEX", leaky_bucket_key, key_lifetime, new_bucket_count)

-- update last update time
redis.call("SETEX", last_bucket_key, key_lifetime, now)

-- return
return {is_allow, new_bucket_count}
```

- - 单元测试

```
    @Test
    @SuppressWarnings("unchecked")
    public void leakyBucketLuaTest() {
        // 实例化限流算法对象
        RateLimiterAlgorithm<?> rateLimiterAlgorithm = RateLimiterAlgorithmFactory.newInstance("leakyBucket");
        // Lua脚本对象
        RedisScript<?> script = rateLimiterAlgorithm.getScript();
        // 限流的测试key
        List<String> keys = Stream.of("test-leakyBucket").collect(Collectors.toList());
        // Lua 参数
        List<String> scriptArgs = Arrays.asList(10 + "", 100 + "", Instant.now().getEpochSecond() + "", "1");
        // 使用ReactiveRedisTemplate执行Lua脚本，这是一个响应式的Redis操作客户端，
        Flux<List<Long>> resultFlux = Singleton.INST.get(ReactiveRedisTemplate.class).execute(script, keys, scriptArgs);
        // 单元测试 响应式的Flux返回值
        StepVerifier
                // Lua脚本执行结果
                .create(resultFlux)
                .expectSubscription()
                // 期望的Lua执行结果
                .expectNext(Arrays.asList(1L, 1L))
                .expectComplete()
                .verify();
    }
```

## 总结

&nbsp; &nbsp; embedded-redisserver组建在单元测试构建Redis环境上很方便，该组件能够指定redis配置文件，主从，哨兵，详细的使用可以参考[github地址](https://github.com/kstyrc/embedded-redis)，但是该项目也有缺陷，比如：大量缺陷未修复。并且CI显示目前这个版本在windows上的构建是失败的；更新特别慢或者说压根没更新，maven仓库只有这一个版本。