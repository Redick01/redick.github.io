# 分布式锁的几种实现方式 <!-- {docsify-ignore-all} -->

- 什么是分布式锁
- 分布式锁的几种不同实现方式

## 什么是分布式锁

&nbsp; &nbsp; 在介绍分布式锁之前，我们先来简单的说一下什么是锁。在单进程的应用中会存在被多个线程所同时操作的共享变量，在多线程操作共享变量的时候就会引发线程安全问题，比如银行转账问题以及多线程的i++问题等等，都会导致操作共享变量后的结果与预期结果不一致的情况，这就是线程安全问题，一般的单进程的应用编程语言都会提供API或线程安全的原子变量的实现，如Java的`synchronized`、`Lock`以及`JUC`包所提供的一些线程安全的集合类和其他原子类库。OK，单进程应用的线程安全问题我们已经能够得到保障了，我们稍微把问题复杂一些，如果不是两个单进程的应用（集群模式），这两个应用会同时的操作一份共享数据，比如文件或数据库中的数据，这个时候编程语言提供的线程同步能力显然是不能够保证共享数据处理的一致性的了，这时候就衍生出了分布式锁，分布式锁提供了在集群环境下多进程内处理任务线程的排他能力，与单进程锁不同，分布式锁需要用到一个中间层来管理和处理锁状态，有了一个中间层的介入就会引入一些其他的复杂问题，比如：**网络延迟**，**中间层的稳定性**等问题，这其实才是一个稳定的分布式锁的关键，本篇文章旨在介绍目前比较普遍的几种分布式锁的实现方式，至于分布式锁稳定性的问题不在本编文章中，感兴趣的同学可以自行深入研究。


## 分布式锁的几种不同实现方式

&nbsp; &nbsp; 前面简单介绍了锁的由来，从单机的锁到分布式锁大致做了一个简单的描述，下面我们来看看目前用的比较多的集中分布式锁的实现，大致包含了如下几种：`使用Redis过期key实现`、`zookeeper的临时节点实现`、`MySQL的for update`、`etcd的lease`、`内存网格hazelcast和ignite的纯java分布式锁`。下面我们开始简单介绍一下几种不同的实现

### Redis实现分布式锁

&nbsp; &nbsp; `Redis`实现分布式锁利用了其过期key的特性，具体的实现`redssion`已经提供了实现，也可以使用其`setnx`命令进行实现，这里不详细说明， 我之前有[结合公司业务基于redis实现分布式的文章](https://juejin.cn/post/6939925570071298055)，感兴趣的同学可以看看。或者直接使用`redssion`实现的分布式锁，`redssion`提供的锁实现更完善，功能也更强大，`redssion`不仅实现了锁还实现了栅栏，信号量等功能。

#### Redis实现分布式锁优缺点

> 优点

1. 高性能，支持高并发

> 缺点

1. 可靠性稍低，单机Redis是CP模型

### zookeeper实现分布式锁

> `Zookeeper`实现分布式锁主要是利用了`Zookeeper`的以下特性：

1. 临时节点：客户端与zookeeper断开节点删除
2. 临时节点递增有序：临时节点递增有序性保证了锁的公平性。
3. 节点监听：保障占有锁的传递有序而且高效，可以防止羊群效应，所谓羊群效应就是一个节点挂了，所有节点都做出反应

> `Zookeeper`实现分布式锁流程

&nbsp; &nbsp; 下面是使用`ZkClient`以实验性目的实现的分布式锁，旨在介绍zookeeper实现的分布式锁流程，仅供参考。如果想使用工业级分布式锁可以使用`apache-curator`框架实现，或者结合自己的业务需求来实现自己的分布式锁，比如支持重入等能力。

#### **lock()加锁**

1. **尝试获取锁tryLock()**

（1）创建临时节点；

（2）获取所有节点，对节点排序

（3）判断当前线程是否获取到锁，即判断当前线程节点是否是最小序号的节点，如果是最小序号节点就获取到锁，否则获取锁失败

（4）获取锁失败后找到自己的前一个节点，用于监听前一个节点变化

1. **获取锁失败等待锁waitForLock()**

（1）创建当前获取锁节点上一个节点的变更监听

（2）处理上一个节点删除变更通知，监听到节点删除通知结束等待继续获取锁

#### **释放锁release()**

（1）业务处理完成，删除临时节点

> `Zookeeper`实现分布式锁总结

&nbsp; &nbsp; 简单来说当客户端A和B都来申请锁，假设客户端A先申请到了锁，这时zk客户端会在`/lock`节点下创建一个临时节点，大致就是`lock1-000001`，客户端B也来尝试获取锁，创建了临时节点`lock1-000002`，此时客户端A和B都会获取所有临时节点并排序，只有客户端生成临时节点序号最小的能够获取到锁，也就是此时客户端A获取到锁，客户端没有获取到锁就会阻塞尝试获取锁，这时客户端B会监听上一个临时节点的变动，也就是客户端A创建的临时节点的变动，当客户端A处理完业务释放锁就会删除掉临时节点`lock1-000001`，客户端B接收到通知后就尝试继续获取锁，依此类推。


#### **zookeeper实现分布式锁优缺点**

> 优点

1. 能够确保锁的一致性

> 缺点

1. 性能不高

&nbsp;
&nbsp;
&nbsp;

```java
public class ZkLockUtil {

    private volatile String PATH;

    private volatile String BEFORE_PATH;

    private static final String NODE = "/lock";

    private ZkLockUtil() {}

    private final ZkClient client = new ZkClient("127.0.0.1:2181");

    public boolean lockFailFast(String path) {
        return tryLock();
    }

    public void lock() {
        // 获取锁
        if (tryLock()) {
            return;
        }
        // 等待
        waitForLock();
        // 再次获取锁
        lock();
    }

    public void release() {
        client.delete(PATH);
    }

    private static volatile boolean waitLock = true;

    private void waitForLock() {
        IZkDataListener listener = new IZkDataListener() {
            @Override
            public void handleDataChange(String dataPath, Object data) throws Exception {

            }

            @Override
            public void handleDataDeleted(String dataPath) throws Exception {
                System.out.println("节点删除");
                waitLock = false;
            }
        };
        client.subscribeDataChanges(BEFORE_PATH, listener);
        if (client.exists(BEFORE_PATH)) {
            System.out.println("加锁失败，等待");
            while (true) {
                if (!waitLock) {
                    break;
                }
            }
        }
        // 释放监听
        client.unsubscribeDataChanges(BEFORE_PATH, listener);
    }

    /**
     * 加锁
     *
     *
     ** @return
     */
    private boolean tryLock() {
        // 创建节点
        if (StringUtils.isBlank(PATH)) {
            PATH = client.createEphemeralSequential(NODE + "/", "lock1");
        }
        // 对节点排序
        List<String> children = client.getChildren(NODE);
        Collections.sort(children);
        if (PATH.equals(NODE + "/" + children.get(0))) {
            System.out.println("获取锁成功");
            return true;
        } else {
            // 不是最小节点，找到自己的前一个，以此类推
            int i = Collections.binarySearch(children, PATH.substring(NODE.length() + 1));
            BEFORE_PATH = NODE + "/" + children.get(i - 1);
        }
        return false;
    }

    private static int inventory = 2;

    public static void main(String[] args) {
        try {
            for (int i = 1; i <= 10; i++) {
                new Thread(() -> {
                    ZkLockUtil zkLockUtil = new ZkLockUtil();
                    try {
                        zkLockUtil.lock();
                        if (inventory > 0) {
                            inventory--;
                        }
                        System.out.println(inventory);
                    } finally {
                        zkLockUtil.release();
                        System.out.println("释放锁");
                    }
                }).start();
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```


### MySQL实现分布式锁

&nbsp; &nbsp; MySQL实现分布式锁利用了MySQL InnoDB引擎的行锁和唯一索引实现。

#### **实现逻辑**

- **创建一张锁资源表，包含主要字段如下**

```table
lock_resource 表名
id 主键
resource_name 锁定资源（创建该字段的唯一索引）
node_thread_info 持有锁线程信息
count 重入次数
create_at 创建时间
update_at 更新时间
desc 描述
```

- **实现锁的前提是要支持并在代码中开启事务，如以下伪代码**

```java
@Service
public class MySqlLock {

    @Transactional(rollbackFor = LockException.class)
    public void lock() {

    }
}

class LockException extends RuntimeException {

    public LockException(String message) {
        super(message);
    }
}
```

- **lock()阻塞获取锁**

1. 根据锁定资源获取锁，获取不到insert锁资源数据
2. 获取到锁资源，判断锁数据node_thread_info是否是当前线程信息，如果是获取锁成功，不是的话自旋获取锁
3. tryLock()的实现是非阻塞快速失败的锁实现
4. 伪代码如下

```java
    @Transactional(rollbackFor = LockException.class, propagation = Propagation.REQUIRED)
    public void lock() {
        while (true) {
            if (tryLock()) {
                return;
            }
            // 休眠3ms时间后重试
            LockSupport.parkNanos(1000 * 1000 * 3);
        }
    }

    /**
     * 伪代码
     * @return
     */
    @Transactional(rollbackFor = LockException.class, propagation = Propagation.REQUIRED)
    public boolean tryLock(String resourceName) {
        String currentNodeInfo = address.getHostName() + Thread.currentThread().getId();
        // 从数据库中获取锁资源
        if (("select * from lock_resource where resource_name = '" + resourceName + "' for update;") == "success") {
            // 当前机器节点信息等于锁资源的机器节点信息，获取锁成功
            if (currentNodeInfo.equals("node_thread_info")) {
                // 重入次数 + 1 ，update 数据库
                String result = "update lock_resource set count = count + 1, update_at = current_time where resource_name = '" + resourceName + "';";
                return true;
            } else {
                // 获取锁失败
                return false;
            }
        } else {
            try {
                String result = "insert into lock_resource value (?, ?, ?, ?);";
                return true;
            } catch (DuplicateKeyException e) {
                // 获取锁失败
                return false;
            }
        }
    }
```

- **释放锁release()**

1. 获取锁资源
2. 获取不到就直接返回true
3. 获取到锁资源，判断释放的锁是不是自己加的
4. 是自己加的锁，判断重入次数，如果重入次数大于1，递减重试次数，并返回释放成功
5. 是自己加的锁，重入次数小于等于0，删除锁资源，返回释放锁成功

```java
    /**
     * 释放锁 伪代码
     * @param resourceName
     * @return
     */
    @Transactional(rollbackFor = LockException.class, propagation = Propagation.REQUIRED)
    public boolean release(String resourceName) {
        String currentNodeInfo = address.getHostName() + Thread.currentThread().getId();
        // 从数据库中获取锁资源
        if (("select * from lock_resource where resource_name = '" + resourceName + "' for update;") == "success") {
            // 当前机器节点信息等于锁资源的机器节点信息，是要释放的锁
            if (currentNodeInfo.equals("node_thread_info")) {
                // 判断重入次数
                if (count > 1) {
                    // 重入次数递减
                    String result = "update lock_resource set count = count - 1, update_at = current_time where resource_name = '" + resourceName + "';";
                } else {
                    // 删除锁资源
                    String result = "delete from lock_resource where resource_name = '" + resourceName + "'";
                }
                return true;
            } else {
                // 获取锁失败
                return false;
            }
        } else {
            // 要释放的锁不存在
            return true;
        }
    }
```

- **锁超时**

&nbsp; &nbsp; 上面的加锁，释放锁流程看似没问题，但是会有一个问题，假如客户端在获取到锁资源后宕机了，这时候锁资源就会一直在数据库中存在，对于这个问题可以启动一个定时任务，用于处理超时的锁，具体的逻辑如下：

1. 启动一个定时任务
2. 预估业务处理时间5s以内完成
3. 定时任务扫描5秒钟以前创建还没被释放的锁，将其释放掉，为增加容错，可以将时间间隔扩大。

#### **MySQL实现分布式锁总结**

&nbsp; &nbsp; MySQL实现的分布式锁逻辑清晰容易理解，不需要依赖其他第三方中间件，其实现依赖MySQL的行锁和唯一索引；但是其缺点也比较明显，理解起来虽然容易，但是实现起来也比较繁琐，还需要自己控制事务已经处理锁超时，并且性能低，对于高并发场景不是很适合。


## 总结

&nbsp; &nbsp; 本文主要介绍了三种分布式锁的实现，基于内存数据库Redis实现的分布式锁非常适合高性能场景；zookeeper实现的分布式锁适用于性能要求不是很高，可用性要求较高的场景；MySQL实现分布式锁适用于性能要求不高，不依赖于其他第三方中间件，可用性要求较高的场景。总的来说使用Redis实现的分布式锁应用比较多，Redis Cluster模式也能够保证高可用并且Redis性能很强。本文的代码不可用于生产系统，仅以伪代码的形式说明其实现原理；

&nbsp; &nbsp; 实现分布式锁的方案还有其他的，感兴趣的同学可以研究下，比如基于ETCD，hazelcast等实现的分布式锁。