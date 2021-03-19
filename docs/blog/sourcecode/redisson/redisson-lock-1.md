# Redisson可重入锁源码分析

## 目标

- Redisson可重入锁简单应用
- Redisson可重入锁源码分析
- 总结

## Redisson可重入锁简单应用

### pom依赖

```
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.12.0</version>
</dependency>
```

### 可重入锁

```
    /**
     * 可重入锁
     * @param waitTime 等待时间
     * @param expireTime 过期时间
     */
    private void reentrantLock(long waitTime, long expireTime) {
        RLock rLock = client.getLock("reentrant-lock-test");
        try {
            // 加锁方式1
            rLock.lock();
            // 加锁方式2，带过期时间
            rLock.lock(expireTime, TimeUnit.SECONDS);
            // 加锁方式3，带最多等待时间和过期时间
            rLock.tryLock(waitTime, expireTime, TimeUnit.SECONDS);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            rLock.unlock();
        }
    }
```