# RocketMQ源码解析-消费者消费消息

## 目标

- 推模式
- 拉模式

&nbsp; &nbsp; 在消费者启动初始化`MQClientInstance`时会初始化`PullMessageService`线程，该线程在生产者启动时也会初始化，但是生产者启动该线程并没有用，只有消费者才有用，该线程也是消费者消费消息的核心线程。

## Push模式下的消息消费





NettyRemotingAbstract-》processRequestCommand

ClientRemotingProcessor-》processRequest-》consumeMessageDirectly

MQClientInstance-》consumeMessageDirectly

ConsumeMessageConcurrentlyService-》consumeMessageDirectly


