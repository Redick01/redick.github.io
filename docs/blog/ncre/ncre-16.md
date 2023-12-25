# 云原生 <!-- {docsify-ignore-all} -->



### 云原生架构定义

​    从技术角度讲，云原生架构是基于云原生技术的一组架构设计原则和设计模式的集合，旨在将云应用中的非业务代码进行最大化的玻璃，从而让云来接管大量的肺功能特性，如：弹性、任性、安全、可观测性、灰度发布等，使业务不再具有非功能性方面的困扰，同时具有轻量、敏捷、高度自动化的特点。由于云原生是面向“云”的，所以也依赖于云的三种概念，IAAS、PAAS、SAAS。

1. 代码结构发生巨大变化
2. 非功能特性大量委托
3. 高度自动化的软件交付

#### 云计算下的高可用

1. 虚拟机：硬件故障，自动热迁移，迁移过程无感知
2. 容器：主要解决应用自身问题，检测到问题后会自动对容器下线和拉起新容器并自动进行流量切换
3. 云服务：构建无状态，结合负载均衡产品带来更强的高可用能力



### 云原生的设计原则

1. 服务化原则
2. 可观测性
3. 零信任原则
4. 弹性原则
5. 韧性原则：拒绝系统不可用，核心目标是提高系统平均无故障时间，从架构上异步包括异步化能力、重试/熔断/反压/限流、主从模式、集群模式、单元化、跨region容灾，异地多活容灾等
6. 所有过程自动化原则：自动化运维，CICD流水线，实现软件交付和运维的完全自动化



### 主要架构模式

1. 服务化架构模式
2. Mesh化架构模式
3. Serverless模式：将“部署”这个动作从运维手中“收走”，使开发者不用关系应用的运行地点、操作系统、网络配置、CPU性能等。从架构抽象上，当业务流量来到的时候云会拉起或调度一个进程进行处理，处理完会关闭或调度业务进程等待下一次触发，把应用的运行也交给了云。适合事件驱动的数据计算任务、计算时间段的请求/响应应用、没有复杂相互调用的长周期任务。
4. 可观测模式
5. 事件驱动模式
6. 存储计算分离模式
7. 分布式事务模式



### 云原生架构的反模式

1. 庞大的单体应用
2. 单体服务“硬拆”
3. 缺乏自动化能力的微服务



### 云原生架构相关的技术

1. 容器技术：作为标准的软件单元，将应用的所有依赖打包，使应用不受环境限制，可以在不同环境下可靠、快速地运行。
2. 容器编排：k8s已经成为容器编排的事实标准，被广泛应用于自动部署，扩展和管理容器化应用。资源调度、应用部署与管理、自动修复、服务发现与服务负载、弹性伸缩、声明式API、可扩展性架构、可移植性。



### 云原生微服务设计约束

1. 微服务个体约束，即单一职责，技术栈自由
2. 微服务与微服务之间的横向关系，主要是，微服务发现和微服务交互。微服务应面向失败设计原则
3. 微服务与数据层之间的纵向约束，数据访问都必须通过微服务的API来访问，保持高内聚，低耦合，微服务应无状态设计
4. 全局视角下的微服务分布式约束：从微服务一开始就要考虑以下因素：高效的运维整个系统，从技术上要准备全自动化的CI CD流水线，满足对开发效率的诉求，并在这个基础上支持蓝绿发布，金丝雀发布等策略，满足业务发布稳定性的诉求。面对复杂的系统，全链路、实时和多维度的可观测能力成为了标配。为了及时、有效地防范各类运维风险，需要从微服务体系多种非时间源荟聚并分析相关数据，然后再中心化的监控系统中进行多维度展现。

### 微服务主要技术

1. dubbo
2. spring cloud
3. Tars
4. SOFAStack
5. DARP：分布式应用运行时，微软退出的一种可移植、无服务器的、时间渠道的运行时