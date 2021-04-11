# 生产中使用ActiveMq消费慢问题排查过程及解决方式

## 目标

- 生产环境ActiveMQ消费慢问题始末
- 第一次代码优化后服务线程阻塞问题排查
- 最终问题解决

## 生产环境ActiveMQ消费慢问题始末

&emsp;&emsp; 公司一个系统生产环境应用ActiveMQ进行通信，由于上下层系统的特殊性，消息的对接使用的P2P的模式，上送服务需要对接上百个ActiveMQ的消息队列，下层服务的每一个实例都对接一个消息队列，并且消息量不大，所以消息生产者是一个单线程的程序，并且生产者使用同步的方式发送消息，就是说只有当消息成功被消费者消费掉后才会发下一条数据。

&emsp;&emsp; 突然在某一个时间开始，待处理的数据产生了积压，随着时间的推移积压的量越来越多，该系统的消息量其实是一直在增长的，生产环境的网络环境比较特殊，延迟较大，初步怀疑是网络+数据量增大的原因，所以针对数据量大的问题有同事想到了第一种解决方案（这个方案其实本身就是个坑），那就是将单线程的发送消息改为多线程，注意，这里仅仅是改了多线程发送，这也为后续的坑埋下了伏笔。

## 第一次代码优化后服务线程阻塞问题排查

&emsp;&emsp; 发生问题后同事很快想到了第一种解决方案，那就是多线程发送，这里其实就是一个坑，因为对ActiveMQ知识储备不足加上对Spring提供的JMSTemplate不是很了解，误以为并行发送就能解决问题，很快代码改完了，性能测试跑完也没问题，就这样程序上线了，问题紧接着就发生了，程序跑了没几分钟，卡住了，数据处理进行不下去了，过了一会儿发现在并没有完全卡死，就是因为处理的太慢了，每个一段时间还是有数据处理的。因为多线程发送可能导致ActiveMQ压力过大处理的更慢了相对于单线程压力大，处理速度就更慢了，同时建议立即回退程序，我给出的建议先将虚拟机的栈日志打印出来，看看程序具体是卡在什么地方，下面是我针对线程栈日志的分析。

- 线程栈日志：

```
"mySend-83" #509 prio=5 os_prio=0 tid=0x00007fb480048000 nid=0x35b2 waiting on condition [0x00007fb409566000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x00000000f9f9b008> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
	at java.util.concurrent.ArrayBlockingQueue.take(ArrayBlockingQueue.java:403)
	at org.apache.activemq.transport.FutureResponse.getResult(FutureResponse.java:48)
	at org.apache.activemq.transport.ResponseCorrelator.request(ResponseCorrelator.java:87)
	at org.apache.activemq.ActiveMQConnection.syncSendPacket(ActiveMQConnection.java:1382)
	at org.apache.activemq.ActiveMQConnection.syncSendPacket(ActiveMQConnection.java:1319)
	at org.apache.activemq.ActiveMQSession.send(ActiveMQSession.java:1967)
	- locked <0x00000000879b5d48> (a java.lang.Object)
	at org.apache.activemq.ActiveMQMessageProducer.send$original$UKNtu2e7(ActiveMQMessageProducer.java:288)
	at org.apache.activemq.ActiveMQMessageProducer.send$original$UKNtu2e7$accessor$V5Iy6ePf(ActiveMQMessageProducer.java)
	at org.apache.activemq.ActiveMQMessageProducer$auxiliary$ZkuXgd8X.call(Unknown Source)
	at org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.InstMethodsInter.intercept(InstMethodsInter.java:93)
	at org.apache.activemq.ActiveMQMessageProducer.send(ActiveMQMessageProducer.java)
	at org.apache.activemq.ActiveMQMessageProducer.send$original$UKNtu2e7(ActiveMQMessageProducer.java:223)
	at org.apache.activemq.ActiveMQMessageProducer.send$original$UKNtu2e7$accessor$V5Iy6ePf(ActiveMQMessageProducer.java)
	at org.apache.activemq.ActiveMQMessageProducer$auxiliary$Rh0cug33.call(Unknown Source)
	at org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.InstMethodsInter.intercept(InstMethodsInter.java:93)
	at org.apache.activemq.ActiveMQMessageProducer.send(ActiveMQMessageProducer.java)
	at org.apache.activemq.ActiveMQMessageProducerSupport.send(ActiveMQMessageProducerSupport.java:269)
	at org.springframework.jms.connection.CachedMessageProducer.send(CachedMessageProducer.java:181)
	at org.springframework.jms.core.JmsTemplate.doSend(JmsTemplate.java:626)
	at org.springframework.jms.core.JmsTemplate.doSend(JmsTemplate.java:597)
	at org.springframework.jms.core.JmsTemplate$4.doInJms(JmsTemplate.java:574)
	at org.springframework.jms.core.JmsTemplate.execute(JmsTemplate.java:484)
	at org.springframework.jms.core.JmsTemplate.send(JmsTemplate.java:570)
	at com.ruubypay.miss.obpsc.db.service.impl.SendMsgServiceImpl.processDataOne(SendMsgServiceImpl.java:47)
	at com.ruubypay.miss.obpsc.db.service.impl.ObpsCBlacklistChangeServiceImpl$2.run(ObpsCBlacklistChangeServiceImpl.java:172)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)

   Locked ownable synchronizers:
	- <0x000000008752f948> (a java.util.concurrent.ThreadPoolExecutor$Worker)
```

- 线程日志分析

根据以上日志可以很明显的找到卡到了com.ruubypay.miss.obpsc.db.service.impl.SendMsgServiceImpl.processDataOne的第47行，这一行的代码是发送消息，代码如下：
```
jmsQueueTemplate.send(queueName, new MessageCreator() {
				@Override
				public Message createMessage(Session session) throws JMSException {
					TextMessage textMessage = session.createTextMessage(msg);
					textMessage.setStringProperty("changeTimestamp", timestamp);
					return textMessage;
				}
			});
```

根据上面日志，进一步定位线程卡住的核心，通过`syncSendPacket`可以看出是使用的同步方式发送数据，根据`org.apache.activemq.transport.FutureResponse.getResult(FutureResponse.java:48)`这段日志可以看到在`FutureResponse`第48行卡住了，我们来看一看这里在干什么，可以看到卡在`responseSlot.take()`，`responseSlot`是一个`ArrayBlockingQueue`，这个阻塞队列就是用于获取消息处理成功的结果，所以大致流程搞清楚了，因为是同步发送需要等待消息消费结果，所以使用一个阻塞队列用于存放消费结果，发送线程一直在`take()`发送结果，如果没有结果就一直阻塞的获取，定位到了程序就是一直获取不到结果所以就阻塞在这里了。

```
    public Response getResult() throws IOException {
        boolean hasInterruptPending = Thread.interrupted();
        try {
            return responseSlot.take();
        } catch (InterruptedException e) {
            hasInterruptPending = false;
            throw dealWithInterrupt(e);
        } finally {
            if (hasInterruptPending) {
                Thread.currentThread().interrupt();
            }
        }
    }
```

- 问题排查

&emsp;&emsp; 基于上面的分析已经可以定位是阻塞等待消费结果导致的，但是为什么阻塞时间会那么长，只是网络延迟不会这么慢，所以开始对比性能测试环境的ActiveMQ的差异，最终发现，两个环境的MQ的持久化方式不同，生产环境使用的MySql，性能测试使用的是文件的方式，我们将性能测试的ActiveMQ的持久化改为Mysql，仍然没有出现问题，这就很奇怪了，所以我们重点看生产环境Mysql的运行情况，从Mysql的执行日志终于发现了问题，Mysql的所有执行的sql语句都很慢耗时都达到了十几秒甚至几十秒，所以我们定位是这个Mysql出了问题。

## 最终问题解决

&emsp;&emsp; 最后制定了切换Mysql的方式，最终解决问题。