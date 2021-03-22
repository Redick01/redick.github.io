# Redisson分布式锁开门狗

## 目标

- Redisson分布式锁看门狗机制
- 源码分析

## Redisson分布式锁看门狗机制

&nbsp; &nbsp; 我们在[上篇文章](https://juejin.cn/post/6942117752198922247)中分析了`Redisson`加锁和解锁的流程，总体看逻辑还算清晰，主要是使用异步执行`lua`脚本进行加锁，但是其中有些细节，之前并没有细说，比如`RedissonLock`中的`internalLockLeaseTime`，该变量默认初始值`30000`毫秒，并且`Config`类中默认值变量的命名也很有意思`lockWatchdogTimeout`，字面上的意思`监视锁的狗`，这就引入了`Redisson`锁的看门狗机制，但是看门狗究竟是做什么的呢？我们继续思考。

&nbsp; &nbsp; 假设在一个分布式的环境中，多个服务实例都来获取锁，这个时候服务实例1获取到了锁，假设服务实例1在获取锁后挂掉、hang住、宕机会怎么样呢？回顾之前的加锁流程，在加锁时我们给锁设置了一个过期时间，所以当服务实例1宕机后，当锁达到超时时间，锁还是会释放掉的；假设，服务实例1没有宕机，而是业务执行时间过长，超过了锁的超时时间呢？这个时候就会导致，服务实例1还在执行，但是锁释放了，这时其他服务实例是能够抢占这个锁的，所以，这种情况下就需要过期时间能够延续的机制，这时`看门狗`就出现了。下面我们看一下`Redisson`可重入锁的看门狗机制实现的源码

## 源码分析

### Redisson异步加锁

&nbsp; &nbsp; 在`RedisExecutor`类中在异步执行完`Lua`脚本后会设置一个监听，代码如下：

```
            writeFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    checkWriteFuture(writeFuture, attemptPromise, connection);
                }
            });
```
&nbsp; &nbsp; 


```
    private <T> RFuture<Long> tryAcquireAsync(long leaseTime, TimeUnit unit, long threadId) {
        if (leaseTime != -1) {
            return tryLockInnerAsync(leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
        }
        RFuture<Long> ttlRemainingFuture = tryLockInnerAsync(commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout(), TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);
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
```