# Spring Cloud技术选型分析 <!-- {docsify-ignore-all} -->

### 私有云Spring Cloud架构现有问题

私有云的组件基本上是Spring Cloud Netflix的，Netflix在2018年开始开始减少在开源领域的投入，逐步开始停止维护其组件。目前私有云的组件存在如下问题：

1. 私有云的配置中心没有可视化界面；
2. 私有云配置中心不支持配置热更新；
3. 私有云的Eureka不能调节负载均衡；
4. 服务容错Hystrix目前停止维护；
5. 负载均衡Ribbon进入维护状态，不再进行扩展功能开发。

## Spring Cloud Netflix与Spring Cloud Alibaba开源组件对比

|         | Spring Cloud Netflix                                 | Spring Cloud Alibaba                            |
|---------|------------------------------------------------------|-------------------------------------------------|
| 注册中心    | Eureka（Eureka 2.0孵化失败）                               | Nacos （性能强劲，感知能力更强）                             |
| 配置中心    | Spring Cloud Config （搭建复杂，约定多，设计繁重，没有界面，上手难）         | Nacos  （搭建简单，可视化界面，配置管理更高效，学习成本低）               |
| API网关   | Zuul/Spring Cloud Gateway （Spring Cloud Gateway性能更好） | Spring Cloud Gateway （Spring Cloud Gateway性能更好） |                  |
| 服务容错    | Hystrix    （2020年停止维护）                               | Sentienl    （可视化配置，上手更容易）                       |
| 负载均衡    | Ribbon                                               | Spring Cloud LoadBalancer                       |
| RPC组件   | Spring Cloud OpenFeign/Rest Template                 | Spring Cloud OpenFeign/Rest Template            |                 |
| 分布式事务   | -                                                    | Seata                                           |
| 分布式链路追踪 | Spring Cloud Sleuth                                  | Spring Cloud Sleuth/Skywalking                  |
| 消息驱动    | Spring Cloud Stream Kafka                            | Spring Cloud Stream RocketMQ                    |
| 分布式调度   | -                                                    | -                                               |



### Spring Cloud Alibaba具有的优势

1. 注册中心基于Nacos，Nacos性能强劲，拥有更强的感知能力，并且支持简单的流量治理（调负载）；
2. 配置中心基于Nacos，提供可视化界面管理配置；
3. 服务容错组件Sentienl上手容易，并且可以与Nacos配置中心相结合，管理容错规则，Sentinel也提供一个可视化界面管理配置；
4. 服务容错组件Sentienl与Spring Cloud Gateway方便集成，便于通过网关控制服务容错；
5. 服务容错组件Sentienl支持OpenFeign和RestTemplate组件，通过简单配置即可实现服务容错，配置可由Nacos管理；
6. 支持分布式事务Seata；
7. 社区更活跃。

## Spring Cloud Alibaba构建微服务架构
<img src="../../../_media/image/spring/sc/WechatIMG258.jpg" style="width:50%;height:50%;">

## Spring Cloud Alibaba开源组件和商业组件对比

Spring Cloud Alibaba提供开源组件支持并且不，还有自家阿里云商业版组件的支持

|       | 开源组件                                 | 阿里云商业组件                              |
|-------|--------------------------------------|--------------------------------------|
| 注册中心  | Nacos                                | ANS                                  |
| 配置中心  | Nacos                                | ACM                                  |
| API网关 | Spring Cloud Gateway                 | -                                    |
| 服务容错  | Sentinel                             | AHAS                                 |
| 负载均衡  | Spring Cloud LoadBalancer            | -                                    |
| RCP组件 | Spring Cloud OpenFeign/Rest Template | Spring Cloud OpenFeign/Rest Template |
| 分布式事务 | Seata                                | -                                    |
| 分布式调度 | -                                    | SchedulerX                           |
| 链路追踪  | Spring Cloud Sleuth/Skywalking       | -                                    |
| 消息驱动  | Spring Cloud Stream RocketMQ         | Apache RocketMQ                      |
| 对象存储  | -                                    | OSS                                  |
| 短信服务  | -                                    | SMS                                  |


