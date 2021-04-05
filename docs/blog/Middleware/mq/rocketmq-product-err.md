# 阿里云RocketMQ订阅关系一致性


订阅关系一致指的是同一个消费者Group ID下所有Consumer实例所订阅的Topic、Group ID、Tag必须完全一致。一旦订阅关系不一致，消息消费的逻辑就会混乱，甚至导致消息丢失。本文提供订阅关系一致的正确示例代码以及订阅关系不一致的错误示例代码，帮助您顺畅地订阅消息。



## 背景信息

消息队列RocketMQ版里的一个消费者Group ID代表一个Consumer实例群组。对于大多数分布式应用来说，一个消费者Group ID下通常会挂载多个Consumer实例。

由于消息队列RocketMQ版的订阅关系主要由Topic+Tag共同组成，因此，保持订阅关系一致意味着同一个消费者Group ID下所有的实例需在以下两方面均保持一致：

- 订阅的Topic必须一致
- 订阅的Topic中的Tag必须一致（包括Tag的数量和Tag的顺序）



## 正确订阅关系图片示例

多个Group ID订阅了多个Topic，并且每个Group ID里的多个消费者实例的订阅关系保持了一致。

[![消息正确订阅关系](https://static-aliyun-doc.oss-accelerate.aliyuncs.com/assets/img/zh-CN/3098680061/p169720.png)](https://static-aliyun-doc.oss-accelerate.aliyuncs.com/assets/img/zh-CN/3098680061/p169720.png)



## 正确订阅关系代码示例

以下例子中，同一个Group ID下的实例订阅相同的Topic和Tag。

- Consumer实例1-1：

  ```java
      Properties properties = new Properties();
      properties.put(PropertyKeyConst.GROUP_ID, "GID_jodie_test_1");
      Consumer consumer = ONSFactory.createConsumer(properties);
      consumer.subscribe("jodie_test_A", "Tag1||2", new MessageListener() {
          public Action consume(Message message, ConsumeContext context) {
              System.out.println(message.getMsgID());
              return Action.CommitMessage;
          }
      });                    
  ```

- Consumer实例1-2：

  ```Java
      Properties properties = new Properties();
      properties.put(PropertyKeyConst.GROUP_ID, "GID_jodie_test_1");
      Consumer consumer = ONSFactory.createConsumer(properties);
      consumer.subscribe("jodie_test_A", "Tag1||2", new MessageListener() {
          public Action consume(Message message, ConsumeContext context) {
              System.out.println(message.getMsgID());
              return Action.CommitMessage;
          }
      });
      consumer.subscribe("jodie_test_A", "Tag1||2", new MessageListener() {
          public Action consume(Message message, ConsumeContext context) {
              System.out.println(message.getMsgID());
              return Action.CommitMessage;
          }
      });      
  ```

## 错误订阅关系图片示例

单个Group ID订阅了多个Topic，但是该Group ID里的多个消费者实例的订阅关系并没有保持一致。

[![错误订阅关系](https://static-aliyun-doc.oss-accelerate.aliyuncs.com/assets/img/zh-CN/3098680061/p169724.png)](https://static-aliyun-doc.oss-accelerate.aliyuncs.com/assets/img/zh-CN/3098680061/p169724.png)

## 错误订阅关系代码示例一

以下例子中，同一个Group ID下的两个实例订阅的Topic不一致。

- Consumer实例1-1：

  ```java
      Properties properties = new Properties();
      properties.put(PropertyKeyConst.GROUP_ID, "GID_jodie_test_1");
      Consumer consumer = ONSFactory.createConsumer(properties);
      consumer.subscribe("jodie_test_A", "*", new MessageListener() {
          public Action consume(Message message, ConsumeContext context) {
              System.out.println(message.getMsgID());
              return Action.CommitMessage;
          }
      });                    
  ```

- Consumer实例1-2：

  ```Java
      Properties properties = new Properties();
      properties.put(PropertyKeyConst.GROUP_ID, "GID_jodie_test_1");
      Consumer consumer = ONSFactory.createConsumer(properties);
      consumer.subscribe("jodie_test_B ", "*", new MessageListener() {
          public Action consume(Message message, ConsumeContext context) {
              System.out.println(message.getMsgID());
              return Action.CommitMessage;
          }
      });                    
  ```

## 错误订阅关系代码示例二

以下例子中，同一个Group ID下订阅Topic的Tag数量不一致。Consumer实例2-1订阅了TagA，而Consumer实例2-2未指定Tag。

- Consumer实例2-1：

  ```java
      Properties properties = new Properties();
      properties.put(PropertyKeyConst.GROUP_ID, "GID_jodie_test_2");
      Consumer consumer = ONSFactory.createConsumer(properties);
      consumer.subscribe("jodie_test_A", "TagA", new MessageListener() {
          public Action consume(Message message, ConsumeContext context) {
              System.out.println(message.getMsgID());
              return Action.CommitMessage;
          }
      });                    
  ```

- Consumer实例2-2：

  ```
      Properties properties = new Properties();
      properties.put(PropertyKeyConst.GROUP_ID, "GID_jodie_test_2");
      Consumer consumer = ONSFactory.createConsumer(properties);
      consumer.subscribe("jodie_test_A", "*", new MessageListener() {
          public Action consume(Message message, ConsumeContext context) {
              System.out.println(message.getMsgID());
              return Action.CommitMessage;
          }
      });                   
  ```

## 错误订阅关系代码示例三

以下例子中，同一个Group ID下订阅Topic的Tag顺序不一致。Consumer实例3-1和实例3-2订阅了相同的Topic且订阅的Tag数量一致，但Tag的顺序不一致。

- Consumer实例3-1：

  ```java
      Properties properties = new Properties();
      properties.put(PropertyKeyConst.GROUP_ID, "GID_jodie_test_3");
      Consumer consumer = ONSFactory.createConsumer(properties);
      consumer.subscribe("jodie_test_A", "TagA||B", new MessageListener() {
          public Action consume(Message message, ConsumeContext context) {
              System.out.println(message.getMsgID());
              return Action.CommitMessage;
          }
      });                 
  ```

- Consumer实例3-2：

  ```java
      Properties properties = new Properties();
      properties.put(PropertyKeyConst.GROUP_ID, "GID_jodie_test_3");
      Consumer consumer = ONSFactory.createConsumer(properties);
      consumer.subscribe("jodie_test_A", "TagB||A", new MessageListener() {
          public Action consume(Message message, ConsumeContext context) {
              System.out.println(message.getMsgID());
              return Action.CommitMessage;
          }
      });                   
  ```

## 错误订阅关系代码示例四

以下例子中，同一个Group ID下订阅Topic个数不一致，且订阅的Topic的Tag不一致。

- Consumer实例4-1：

  ```java
      Properties properties = new Properties();
      properties.put(PropertyKeyConst.GROUP_ID, "GID_jodie_test_4");
      Consumer consumer = ONSFactory.createConsumer(properties);
      consumer.subscribe("jodie_test_A", "TagA", new MessageListener() {
          public Action consume(Message message, ConsumeContext context) {
              System.out.println(message.getMsgID());
              return Action.CommitMessage;
          }
      });
      consumer.subscribe("jodie_test_B", "TagB", new MessageListener() {
          public Action consume(Message message, ConsumeContext context) {
              System.out.println(message.getMsgID());
              return Action.CommitMessage;
          }
      });                    
  ```

- Consumer实例4-2：

  ```java
      Properties properties = new Properties();
      properties.put(PropertyKeyConst.GROUP_ID, "GID_jodie_test_4");
      Consumer consumer = ONSFactory.createConsumer(properties);
      consumer.subscribe("jodie_test_A", "TagB", new MessageListener() {
          public Action consume(Message message, ConsumeContext context) {
              System.out.println(message.getMsgID());
              return Action.CommitMessage;
          }
      });                   
  ```