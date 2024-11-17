# 物联网与边缘计算 <!-- {docsify-ignore-all} -->



### MQTT

MQTT是一个轻量级的消息协议，专门设计用于低带宽、高延迟或不稳定的网络条件下的远程设备通信。MQTT是发布/订阅模式的协议，适用于物联网设备，如传感器、手机、平板等。

1. 主要特点

- 轻量级：MQTT协议非常简单，它的header只有2字节到5字节。
- 开放式：任何人都可以实现MQTT协议。
- 简单的消息格式：MQTT消息可以被理解为Topic（主题）、Payload（负载）和QoS（服务质量）。
- 支持QoS（服务质量）：MQTT支持三种服务质量等级，以确保消息传递的可靠性。
- 使用TCP/IP提供传输：MQTT可以在基于TCP/IP的网络上运行、

2. 应用场景

- 硬件和设备通信：用于监控和控制外部设备。
- 远程通信：用于远程设备、移动通信和web应用。
- 物联网（IoT）：用于链接和监控物联网设备。

3. MQTT协议中的主要角色

- 发布者（Publisher）：发布消息到主题（Topic）的设备。
- 代理（Broker）：中介，接收来自发布者的消息并将他们转发到订阅者。
- 订阅者（Subscriber）：订阅特定主题的设备，以接收发布者发送的消息。

4.  MQTT协议消息类型：

- CONNECT：客户端连接到MQTT代理
- CONNACK：确认连接请求
- PUBLISH；发布消息
- PUBACK：发布确认
- SUBSCRIBE：客户端请求订阅主题
- SUBACK：确认订阅请求
- UNSUBSCRIBE：客户端请求取消订阅
- UNSUBACK：确认取消订阅
- PINGREQ/PINGRESP：心跳
- DISCONNECT：客户端断开连接