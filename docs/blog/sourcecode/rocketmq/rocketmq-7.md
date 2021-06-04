# RocketMQ源码解析-消费者消费消息

## 目标

- 推模式
- 拉模式

PullMessageService

NettyRemotingAbstract-》processRequestCommand

ClientRemotingProcessor-》processRequest-》consumeMessageDirectly

MQClientInstance-》consumeMessageDirectly

ConsumeMessageConcurrentlyService-》consumeMessageDirectly


