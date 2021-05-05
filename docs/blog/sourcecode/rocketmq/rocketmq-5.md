# RocketMQ源码解析-Producer发送消息源码分析

- RocketMQ消息结构
- 同步发送
- 异步发送
- 单向发送

## RocketMQ消息结构

```
public class Message implements Serializable {
    private static final long serialVersionUID = 8445773977080406428L;

    /** topic */
    private String topic;
    private int flag;
    /** 消息的其他属性。例如 Tags */
    private Map<String, String> properties;
    /** 消息内容*/
    private byte[] body;
    /** 事务ID*/
    private String transactionId;
    // 省略
    ...
}
```

## 同步发送SYNC

### 同步发送Demo

&nbsp; &nbsp; 参考`org.apache.rocketmq.example.simple.Producer`

```
public class Producer {
    public static void main(String[] args) throws MQClientException, InterruptedException {

        DefaultMQProducer producer = new DefaultMQProducer("ProducerGroupName");
        producer.start();

        for (int i = 0; i < 128; i++)
            try {
                {
                    Message msg = new Message("TopicTest",
                        "TagA",
                        "OrderID188",
                        "Hello world".getBytes(RemotingHelper.DEFAULT_CHARSET));
                    SendResult sendResult = producer.send(msg);
                    System.out.printf("%s%n", sendResult);
                }

            } catch (Exception e) {
                e.printStackTrace();
            }

        producer.shutdown();
    }
}
```

### DefaultMQProducer#send()
```
    @Override
    public SendResult send(
        Message msg) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
        // msg检查
        Validators.checkMessage(msg, this);
        msg.setTopic(withNamespace(msg.getTopic()));
        return this.defaultMQProducerImpl.send(msg);
    }
```
### DefaultMQProducerImpl#sendDefaultImpl()
```
    /**
     * DEFAULT SYNC -------------------------------------------------------
     */
    public SendResult send(
        Message msg) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
        return send(msg, this.defaultMQProducer.getSendMsgTimeout());
    }
    public SendResult send(Message msg,
        long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
        return this.sendDefaultImpl(msg, CommunicationMode.SYNC, null, timeout);
    }
```

## 异步发送ASYNC

## 单向发送OneWay


## 总结