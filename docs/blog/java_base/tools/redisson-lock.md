# Redisson分布式锁1-应用

## 目标

- 基于Redisson实现分布式锁
- 总结

## 基于Redisson实现分布式锁

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

### 公平锁

```
    /**
     * 公平锁
     * @param waitTime 等待时间
     * @param expireTime 过期时间
     */
    private void fairLock(long waitTime, long expireTime) {
        RLock rLock = client.getFairLock("fair-lock-test");
        try {
            // 最常见的使用方法
            rLock.lock();

            // 支持过期解锁功能, 10秒钟以后自动解锁,无需调用unlock方法手动解锁
            rLock.lock(expireTime, TimeUnit.SECONDS);

            // 尝试加锁，最多等待100秒，上锁以后10秒自动解锁
            boolean result = rLock.tryLock(waitTime, expireTime, TimeUnit.SECONDS);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            rLock.unlock();
        }
    }
```

### 联锁（MultiLock）

```
    /**
     * 联锁
     * @param redisson1 客户端1
     * @param redisson2 客户端2
     * @param redisson3 客户端3
     */
    public void testMultiLock(RedissonClient redisson1,
                              RedissonClient redisson2,
                              RedissonClient redisson3){

        RLock lock1 = redisson1.getLock("lock1");
        RLock lock2 = redisson2.getLock("lock2");
        RLock lock3 = redisson3.getLock("lock3");

        RedissonMultiLock lock = new RedissonMultiLock(lock1, lock2, lock3);

        try {
            // 同时加锁：lock1 lock2 lock3, 所有的锁都上锁成功才算成功。
            lock.lock();

            // 尝试加锁，最多等待100秒，上锁以后10秒自动解锁
            boolean result = lock.tryLock(100, 10, TimeUnit.SECONDS);

        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }

    }
```

### 红锁（RedLock）

```
    public void testRedLock(RedissonClient redisson1,
                            RedissonClient redisson2,
                            RedissonClient redisson3) {
        RLock lock1 = redisson1.getLock("lock1");
        RLock lock2 = redisson2.getLock("lock2");
        RLock lock3 = redisson3.getLock("lock3");
        RedissonRedLock redLock = new RedissonRedLock(lock1, lock2, lock3);
        try {
            // 同时加锁：lock1 lock2 lock3, 红锁在大部分节点上加锁成功就算成功。
            redLock.lock();

            // 尝试加锁，最多等待100秒，上锁以后10秒自动解锁
            boolean result = redLock.tryLock(100, 10, TimeUnit.SECONDS);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            redLock.unlock();
        }
    }
```

### 读写锁（ReadWriteLock）

&nbsp; &nbsp; Redisson的分布式可重入读写锁RReadWriteLock Java对象实现了java.util.concurrent.locks.ReadWriteLock接口。同时还支持自动过期解锁。该对象允许同时有多个读取锁，但是最多只能有一个写入锁；以下代码仅供参考

```
    public void readWriteLock() {
        RReadWriteLock rReadWriteLock = client.getReadWriteLock("test-read-write-lock");
        try {
            // 最常见的使用方法
            rReadWriteLock.readLock().lock();
            // 或
            rReadWriteLock.readLock().lock();

            // 支持过期解锁功能
            // 10秒钟以后自动解锁
            // 无需调用unlock方法手动解锁
            rReadWriteLock.readLock().lock(10, TimeUnit.SECONDS);
            // 或
            rReadWriteLock.writeLock().lock(10, TimeUnit.SECONDS);

            // 尝试加锁，最多等待100秒，上锁以后10秒自动解锁
            boolean res1 = rReadWriteLock.readLock().tryLock(100, 10, TimeUnit.SECONDS);
            // 或
            boolean res2 = rReadWriteLock.writeLock().tryLock(100, 10, TimeUnit.SECONDS);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            rReadWriteLock.readLock().unlock();
            rReadWriteLock.readLock().unlock();
        }
```

### 信号量（Semaphore）

&nbsp; &nbsp; Redisson的分布式信号量（Semaphore）Java对象RSemaphore采用了与java.util.concurrent.Semaphore相似的接口和用法。

```
    /**
     * 信号量
     */
    public void testSemaphore() {
        RSemaphore rSemaphore = client.getSemaphore("test-semaphore");
        try {
            // 获得信号量（许可证），等待，直到获取到
            rSemaphore.acquire();
            // 异步，获得信号量
            rSemaphore.acquireAsync();
            // 指定信号量（许可证）数量
            rSemaphore.acquire(23);
            // 尝试获取许可证，获取到返回true，否则返回false
            boolean result = rSemaphore.tryAcquire();
            // 异步 尝试获取许可证，获取到返回true，否则返回false
            rSemaphore.tryAcquireAsync();
            // 带等待时间尝试获取许可证，获取到返回true，否则返回false
            rSemaphore.tryAcquire(23, TimeUnit.SECONDS);
            // 带等待时间 异步 尝试获取许可证，获取到返回true，否则返回false
            rSemaphore.tryAcquireAsync(23, TimeUnit.SECONDS);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            rSemaphore.release();
            rSemaphore.releaseAsync();
        }
    }
```

### 可过期性信号量（PermitExpirableSemaphore）

&nbsp; &nbsp; Redisson的可过期性信号量（PermitExpirableSemaphore）实在RSemaphore对象的基础上，为每个信号增加了一个过期时间。每个信号可以通过独立的ID来辨识，释放时只能通过提交这个ID才能释放。

```
    /**
     * 可过期性信号量
     * @throws InterruptedException
     */
    public void testPermitExpirableSemaphor() throws InterruptedException {
        RPermitExpirableSemaphore semaphore = client.getPermitExpirableSemaphore("mySemaphore");
        String permitId1 = semaphore.acquire();
        // 获取一个信号，有效期只有2秒钟。
        String permitId2 = semaphore.acquire(2, TimeUnit.SECONDS);
        // ...
        semaphore.release(permitId1);
        semaphore.release(permitId2);
    }
```

### 闭锁（CountDownLatch）

&nbsp; &nbsp; Redisson的分布式闭锁（CountDownLatch）Java对象RCountDownLatch采用了与java.util.concurrent.CountDownLatch相似的接口和用法。

```
    public void testCountDownLatch() throws Exception {
        RCountDownLatch latch1 = client.getCountDownLatch("anyCountDownLatch");
        latch1.trySetCount(1);
        latch1.await();

        // 在其他线程或其他JVM里
        RCountDownLatch latch2 = client.getCountDownLatch("anyCountDownLatch");
        latch2.countDown();
    }
```
## 总结

&nbsp; &nbsp; Redisson实现了很多种分布式锁，并且这些分布式锁在不进行深层次的研究时感觉类似于JUC的锁，后续我们来研究下本片介绍的分布式锁实现。