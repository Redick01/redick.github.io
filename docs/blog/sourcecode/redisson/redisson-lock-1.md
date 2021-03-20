# Redisson可重入锁源码分析

## 目标

- Redisson可重入锁简单应用
- Redisson可重入锁加锁/解锁源码分析
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

## Redisson可重入锁加锁/解锁源码分析

```
// 从redisson客户端中创建一个实现RLock接口的对象，名字为reentrant-lock-test
RLock rLock = client.getLock("reentrant-lock-test");
```

- **Redisson类getLock**

&nbsp; &nbsp; `getLock`接收一个字符串类型的参数，这里是锁的名字，该方法创建了一个`RedissonLock`对象实例，构造方法接收`CommandAsyncExecutor`类型参数，该对象大致是一个异步的线程池用于发送命令到`redis`这里不展开说明该类。
```
    @Override
    public RLock getLock(String name) {
        return new RedissonLock(connectionManager.getCommandExecutor(), name);
    }
```

- **加锁**

&nbsp; &nbsp; 下面是不带参数的可重入锁的实现

```
    @Override
    public void lock() {
        try {
            lock(-1, null, false);
        } catch (InterruptedException e) {
            throw new IllegalStateException();
        }
    }

    private void lock(long leaseTime, TimeUnit unit, boolean interruptibly) throws InterruptedException {
        // 获取当前线程ID
        long threadId = Thread.currentThread().getId();
        // 尝试获取锁，这里先不展开，后面详细介绍， ttl是毫秒的剩余过期时间
        Long ttl = tryAcquire(leaseTime, unit, threadId);
        // 获取锁成功
        if (ttl == null) {
            return;
        }
        // 一个基于响应式的 future对象，RFuture继承了JUC的Future
        RFuture<RedissonLockEntry> future = subscribe(threadId);
        // 根据参数interruptibly，订阅
        if (interruptibly) {
            commandExecutor.syncSubscriptionInterrupted(future);
        } else {
            commandExecutor.syncSubscription(future);
        }

        try {
            // 循环尝试获取锁
            while (true) {
                ttl = tryAcquire(leaseTime, unit, threadId);
                // 获取到锁
                if (ttl == null) {
                    // 跳出
                    break;
                }

                // 未获取到锁，并且过期时间大于等于0
                if (ttl >= 0) {
                    try {
                        // 获取 信号量，这里不详细看
                        future.getNow().getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                    } catch (InterruptedException e) {
                        if (interruptibly) {
                            throw e;
                        }
                        future.getNow().getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                    }
                } else {
                    if (interruptibly) {
                        future.getNow().getLatch().acquire();
                    } else {
                        future.getNow().getLatch().acquireUninterruptibly();
                    }
                }
            }
        } finally {
            // 取消订阅
            unsubscribe(future, threadId);
        }
//        get(lockAsync(leaseTime, unit));
    }

    // 尝试获取获取锁，这里返回的get方法接收一个RFuture参数，并返回结果
    private Long tryAcquire(long leaseTime, TimeUnit unit, long threadId) {
        return get(tryAcquireAsync(leaseTime, unit, threadId));
    }

    private <T> RFuture<Long> tryAcquireAsync(long leaseTime, TimeUnit unit, long threadId) {
        // 租用锁时间不等于-1也就是自定义了租用时间
        if (leaseTime != -1) {
            return tryLockInnerAsync(leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
        }
        // tryLockInnerAsync 是执行redis操作
        RFuture<Long> ttlRemainingFuture = tryLockInnerAsync(commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout(), TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);
        // 响应式编程模型的处理方式，由onComplete处理执行结果
        ttlRemainingFuture.onComplete((ttlRemaining, e) -> {
            if (e != null) {
                return;
            }

            // lock acquired
            if (ttlRemaining == null) {
                scheduleExpirationRenewal(threadId);
            }
        });
        return ttlRemainingFuture;
    }

    // 异步的 操作redis加锁处理
    <T> RFuture<T> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
        // 计算租用锁的时间
        internalLockLeaseTime = unit.toMillis(leaseTime);
        // 执行lua脚本，大致逻辑是，如果锁不存在，设置锁，并设置过期时间然后返回nil；
        // 如果当前线程锁存在，对参数+1，代表一次重入；
        // 如果不是当前线程获取这把锁，并且获取失败了，那么返回锁的剩余时间
        return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
                  "if (redis.call('exists', KEYS[1]) == 0) then " +
                      "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                      "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                      "return nil; " +
                  "end; " +
                  "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                      "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                      "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                      "return nil; " +
                  "end; " +
                  "return redis.call('pttl', KEYS[1]);",
                    Collections.<Object>singletonList(getName()), internalLockLeaseTime, getLockName(threadId));
    }
```

- **释放锁**

&nbsp; &nbsp; 下面是释放锁的流程

```
    @Override
    public RFuture<Void> unlockAsync(long threadId) {
        // RPromise是Redisson封装的对象，实现了JUC的Future，可以对future进行标记，并且通知所有的listener
        RPromise<Void> result = new RedissonPromise<Void>();
        // 解锁，后面详细看一下
        RFuture<Boolean> future = unlockInnerAsync(threadId);
        // 异步处理完成后的处理
        future.onComplete((opStatus, e) -> {
            if (e != null) {
                // 该方法中的处理逻辑是从内存中清除当前线程的一些信息
                cancelExpirationRenewal(threadId);
                result.tryFailure(e);
                return;
            }
            // 尝试解锁不是当前线程的锁
            if (opStatus == null) {
                IllegalMonitorStateException cause = new IllegalMonitorStateException("attempt to unlock lock, not locked by current thread by node id: "
                        + id + " thread-id: " + threadId);
                result.tryFailure(cause);
                return;
            }
            
            cancelExpirationRenewal(threadId);
            result.trySuccess(null);
        });

        return result;
    }

    // 解锁的lua脚本，大致逻辑是
    // 1.解锁的是否是当前线程持有的锁，如果不是返回nil
    // 2.获取将当前线程持有锁的技术-1并且返沪计数值，如果技术大于0那就重新设置过期时间，返回0
    // 3.否则已经完全能够释放锁，del掉redis数据，并发布
    protected RFuture<Boolean> unlockInnerAsync(long threadId) {
        return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
                "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
                    "return nil;" +
                "end; " +
                "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
                "if (counter > 0) then " +
                    "redis.call('pexpire', KEYS[1], ARGV[2]); " +
                    "return 0; " +
                "else " +
                    "redis.call('del', KEYS[1]); " +
                    "redis.call('publish', KEYS[2], ARGV[1]); " +
                    "return 1; "+
                "end; " +
                "return nil;",
                Arrays.<Object>asList(getName(), getChannelName()), LockPubSub.UNLOCK_MESSAGE, internalLockLeaseTime, getLockName(threadId));

    }
```

## 总结

&nbsp; &nbsp; Redisson可重入锁大致源码就看过一遍了，其中有些不是核心锁实现的流程没有介绍，Redisson的可重入锁实现了`JUC`框架的`Lock`接口，`reids`操作的是自己实现的协议网络部分使用的netty，加锁和解锁的核心就是两个lua的脚本，Redisson使用redis的哈希表结构实现，`rLock.lock(expireTime, TimeUnit.SECONDS);`和`rLock.lock();`的实现类似`tryLock`的实现有些差异，但是lua脚本部分是完全一样的。