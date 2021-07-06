# 1. Service Mesh（服务网格）演进

- 什么是服务网格
- 服务网格演进总结

## 1.1. 什么是服务网格

&nbsp; &nbsp; 用一句话简单概括什么是Service Mesh，可以将其比作是应用程序或者说微服务间的TCP/IP，负责服务之间的网络调用、限流、熔断和监控。杜宇编写应用程序来说无需关心TCP/IP这一层，同样使用Service Mesh无需关心服务之间的那些通过微服务框架实现的事情，比如 限流，熔断，负载均衡等，现在只需要交给Service Mesh就可以了。为了更清晰的理解Service Mesh就需要了解微服务和Service Mesh技术的历史发展脉络，参考资料[什么是Service Mesh](https://zhuanlan.zhihu.com/p/61901608)来看一下微服务技术的发展脉络。


- **时代0：**开发人员想象中不同服务之间的通信方式，抽象表示如下：

![avatar](_media/../../../../_media/image/structure/servicemesh/time0-style.png)

- **时代1：原始通信时代**

通信需要底层能够传输字节码和电子信号的物理层来完成，在TCP协议出现之前，服务需要自己处理网络通信所面临的丢包、乱序、重试等一系列流控问题，因此服务实现中，除了业务逻辑外，还夹杂着对网络传输问题的处理逻辑。

![avatar](_media/../../../../_media/image/structure/servicemesh/time1-style.png)

- **时代2：TCP时代**

为了避免每个服务都需要自己实现一套相似的网络传输处理逻辑，TCP协议出现了，它解决了网络传输中通用的流量控制问题，将技术栈下移，从服务的实现中抽离出来，成为操作系统网络层的一部分。

![avatar](_media/../../../../_media/image/structure/servicemesh/time2-style.png)

- **时代3：第一代微服务**

有了TCP，服务之间的通信不再是难题，以GFS/BigTable/MapReduce为代表的分布式系统得以蓬勃发展。这时分布式系统特有的一些通信特点出现了，如熔断策略、负载均衡、服务发现、认证和授权、调用链路追踪和监控等等，于是业务系统就要根据需要来实现这些通信语意，使得业务代码和分布式系统的语意实现耦合在一起。

![avatar](_media/../../../../_media/image/structure/servicemesh/time3-style.png)

- **时代4：第二代微服务**

为了避免每个服务都需要自己实现一套分布式系统通信的语义功能，随着技术的发展，一些面向微服务架构的开发框架出现了，如Alibaba的Dubbo、Twitter的Finagle和Spring Cloud等等，这些框架实现了分布式系统的通用语意，如负载均衡、服务注册发现、限流、熔断、重试等，使得应用程序在一定程序上屏蔽了这些通信细节，使得开发人员使用较少的框架代码就能开发出健壮的分布式系统。

![avatar](_media/../../../../_media/image/structure/servicemesh/time4-style.png)

- **时代5：第一代Service Mesh**

第二代的微服务看起来已经很完美了，是日今日有很多公司仍然采用的是这种架构，但是随着分布式系统越来越庞大开发人员很快又发现了一些问题：

1. 虽然框架本身屏蔽了分布式系统通信的一些通用功能实现细节，但开发者却要花更多精力去掌握和管理复杂的框架本身，在实际应用中，去追踪和解决框架出现的问题也绝非易事；
2. 开发框架通常只支持一种或几种特定的语言，回过头来看文章最开始对微服务的定义，一个重要的特性就是语言无关，但那些没有框架支持的语言编写的服务，很难融入面向微服务的架构体系，想因地制宜的用多种语言实现架构体系中的不同模块也很难做到；
3. 框架以库的形式和服务集成，复杂项目依赖时的库版本兼容问题非常棘手，同时，框架库的升级也无法对服务透明，服务会因为和业务无关的lib库升级而被迫升级；

因此以Linkerd，Envoy，NginxMesh为代表的代理模式（边车模式）应运而生，这就是第一代Service Mesh，它将分布式服务的通信抽象为单独一层，在这一层中实现负载均衡、服务发现、认证授权、监控追踪、流量控制等分布式系统所需要的功能，作为一个和服务对等的代理服务，和服务部署在一起，接管服务的流量，通过代理之间的通信间接完成服务之间的通信请求，这样上边所说的三个问题也迎刃而解。

![avatar](_media/../../../../_media/image/structure/servicemesh/time5-1.jpeg)

如果我们从一个全局视角来看，就会得到如下部署图：

![avatar](_media/../../../../_media/image/structure/servicemesh/time5-2.jpeg)

如果我们暂时略去业务服务，只看Service Mesh的单机组件组成的网络：

![avatar](_media/../../../../_media/image/structure/servicemesh/time5-3.jpeg)

所谓Service Mesh就是服务网格了，他看起来就像一个由若干服务代理所组成的错综复杂的网格。

- **时代6：第二代Service Mesh**

第一代Service Mesh由一系列独立运行的单机代理服务构成，为了提供统一的上层运维入口，演化出了集中式的控制面板，所有的单机代理组件通过和控制面板交互进行网络拓扑策略的更新和单机数据的汇报。这就是以Istio为代表的第二代Service Mesh。

![avatar](_media/../../../../_media/image/structure/servicemesh/service-mesh-arch.png)

单机代理组件（数据面板）和控制面板的Service Mesh全局部署视图如下：

![avatar](_media/../../../../_media/image/structure/servicemesh/time6-1.jpeg)

## 1.2. 服务网格演进总结

&nbsp; &nbsp; 微服务系统架构演进大致如此，下面是Service Mesh这个词发明人对Service Mesh的定义：

  服务网格是一个基础设施层，用于处理服务间通信。云原生应用有着复杂的服务拓扑，服务网格保证请求在这些拓扑中可靠地穿梭。在实际应用当中，服务网格通常是由一系列轻量级的网络代理组成的，它们与应用程序部署在一起，但对应用程序透明。

这个定义中可以提炼四个关键词：

**基础设施层+请求在这些拓扑中可靠穿梭：**这两个词加起来描述了Service Mesh定位于基础设施，在功能上应用于程序见通信的中间层
**网络代理：**这描述了Service Mesh的实现形态，Service Mesh实现轻量级的网络代理
**对应用透明：**这描述了Service Mesh的关键特点，应用程序无感知，Service Mesh能够解决以Spring Cloud为代表的第二代微服务框架所面临的三个本质问题；

总结一下Service Mesh优点：

- 屏蔽分布式系统通信的复杂性(负载均衡、服务发现、认证授权、监控追踪、流量控制等等)，服务只用关注业务逻辑；
- 真正的语言无关，服务可以用任何语言编写，只需和Service Mesh通信即可；
- 对应用透明，Service Mesh组件可以单独升级；

Service Mesh目前也面临一些挑战：

- Service Mesh组件以代理模式计算并转发请求，一定程度上会降低通信系统性能，并增加系统资源开销；
- Service Mesh组件接管了网络流量，因此服务的整体稳定性依赖于Service Mesh，同时额外引入的大量Service Mesh服务实例的运维和管理也是一个挑战；



## 1.3. 参考

https://zhuanlan.zhihu.com/p/61901608
https://jimmysong.io/blog/what-is-a-service-mesh/