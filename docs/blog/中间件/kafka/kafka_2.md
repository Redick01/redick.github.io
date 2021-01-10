# SpringBoot集成Kafka

## SpringBoot快速集成Kafka

- Maven依赖

```
<!-- 启动springbootstarter支持 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

- application.properties配置

```
# 自动注入kafka配置
spring.kafka.bootstrap-servers=192.168.3.78:9001,192.168.3.78:9002,192.168.3.78:9003
spring.kafka.consumer.group-id=myGroup
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.producer.key-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.value-serializer=org.apache.kafka.common.serialization.StringSerializer
```

- 生产者

```
@Component
@Slf4j
public class KafkaProducer {

    @Resource
    private KafkaTemplate<String, String> kafkaTemplate;

    public void sendMessage(String message) {
        log.info("发送消息：{}", message);
        kafkaTemplate.send("test32", message);
    }

    public void sendMessage(String key, String message) {
        log.info("发送消息：{}", message);
        kafkaTemplate.send("test32", key, message);
    }
}
```

- 消费者

```
public class ActiveMQConsumer {

    private static final String URL = "tcp://192.168.3.78:61616";

    /**
     * Queue Name
     */
    private static final String QUEUE_NAME = "queue-demo";

    /**
     * Topic Name
     */
    private static final String TOPIC_NAME = "topic-demo";


    /**
     * 消费-Topic 订阅/发布模式
     * @throws JMSException
     */
    public void consumerTopic() throws JMSException {
        //1.创建ConnectionFactory
        ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory(URL);

        //2.创建连接
        Connection connection = connectionFactory.createConnection();

        //3.启动连接
        connection.start();

        //4.创建会话
        Session session = connection.createSession(Boolean.FALSE, Session.AUTO_ACKNOWLEDGE);
        //5.创建一个目标
        Destination destination = session.createTopic(TOPIC_NAME);
        //6.创建消费者
        MessageConsumer consumer = session.createConsumer(destination);

        //7.创建消息监听器
        consumer.setMessageListener(new MessageListener() {
            @Override
            public void onMessage(Message message) {
                TextMessage textMessage= (TextMessage) message;
                try {
                    System.out.println("消费Topic消息：" + textMessage.getText());
                } catch (JMSException e) {
                    e.printStackTrace();
                }
            }
        });
    }

    /**
     * 消费-Queue模式
     * @throws JMSException
     */
    public void consumerQueue() throws JMSException {
        //1.创建ConnectionFactory
        ConnectionFactory connectionFactory = new ActiveMQConnectionFactory(URL);

        //2.创建连接
        Connection connection = connectionFactory.createConnection();

        //3.启动连接
        connection.start();
        //4.创建会话
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);

        //5.创建一个目标
        Destination destination = session.createQueue(QUEUE_NAME);

        //6.创建消费者
        MessageConsumer consumer = session.createConsumer(destination);

        //7.创建消息监听器
        consumer.setMessageListener(new MessageListener() {
            @Override
            public void onMessage(Message message) {
                TextMessage textMessage= (TextMessage) message;
                try {
                    System.out.println("消费Queue消息：" + textMessage.getText());
                } catch (JMSException e) {
                    e.printStackTrace();
                }
            }
        });
    }
}
```

- 测试

```
@SpringBootApplication
@ComponentScan(basePackages = {"com.homework.kafka"})
public class Application {

    @Resource
    private KafkaProducer kafkaProducer;

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).run(args);
    }

    /**
     * 测试kafka集群消息生产和消费
     */
    @PostConstruct
    public void sendKafka() {
        for (int i = 0; i < 100; i++) {
            kafkaProducer.sendMessage("redick" + i);
        }
    }
}
```

## 自定义配置集成kafka

- application.properties配置

```
#producer
kafka.producer.bootstrapServers=192.168.3.78:9001,192.168.3.78:9002,192.168.3.78:9003
# consumer
kafka.consumer.bootstrapServers=192.168.3.78:9001,192.168.3.78:9002,192.168.3.78:9003
kafka.consumer.groupId=myGroup
```

- 消费者配置

```
    @EnableKafka
    @Configuration
    public class KafkaConsumerConfig {

        @Value("${kafka.consumer.bootstrapServers}")
        private String consumerBootstrapServers;

        @Value("${kafka.consumer.groupId}")
        private String consumerGroupId;
        @Bean
        public ConsumerFactory<String, String> consumerFactory() {
            Map<String, Object> props = new HashMap<>();
            props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, consumerBootstrapServers);
            props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 50);
            props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
            props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
            return new DefaultKafkaConsumerFactory<>(props);
        }

        @Bean
        public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory() {
            ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
            factory.setConcurrency(3);
            factory.setConsumerFactory(consumerFactory());
            return factory;
        }
    }
```

- 生产者配置

```
    @Configuration
    public class KafkaProducerConfig {

        @Value("${kafka.producer.bootstrapServers}")
        private String producerBootstrapServers;

        @Bean
        public ProducerFactory<String, String> producerFactory(){
            Map<String, Object> configs = new HashMap<>();
            configs.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, producerBootstrapServers);
            configs.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
            configs.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,StringSerializer.class);
            return new DefaultKafkaProducerFactory<>(configs);
        }

        @Bean
        public KafkaTemplate<String, String> kafkaTemplate() {
            return new KafkaTemplate<>(producerFactory());
        }
    }
```
