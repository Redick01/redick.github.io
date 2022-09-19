# TCP/IP四层模型讲解 <!-- {docsify-ignore-all} -->


​    这几天抽空学了下OSI七层模型和TCP/IP四层模型，以前总觉得这些纯理论层面的东西对实战没有什么意义，现在回头看看还是非常有意义的，不过放在当时我也懒得学这么枯燥的东西。

​    接下来把今天学到的东西整理成了笔记，无论是对于网络安全，还是建站、服务器维护等等领域，都是非常实用的知识点，还是建议大家花几分钟来好好稳固下基础。



## OSI七层模型

 

**[![TCP/IP四层模型](http://www.vuln.cn/wp-content/uploads/2015/09/1.png)](http://www.vuln.cn/wp-content/uploads/2015/09/1.png)**

> 应用层：各种应用层协议，如：域名系统(Domain Name System，DNS)，文件传输协议(File Transfer Protocol，FTP)，简单邮件传送协议(Simple Mail Transfer Protocol, SMTP)，超文本传输协议(HyperText Transfer Protocol，HTTP)，简单网络管理协议(simple Network Management Protocol，SNMP)，远程登录协议(Telnet)
>
> 表示层：用来解码不同的格式为机器语言，以及其他功能。
>
> 会话层：判断是否需要网络传输。
>
> 传输层：识别端口来指定服务器，如指定80端口的www服务。
>
> 网络层：提供逻辑地址选路，即发送ip地址到接收的ip地址。
>
> 数据链路层：成帧，识别MAC地址来访问媒介，如交换机的功能。
>
> 物理层：设备之间的比特流传输。

## TCP/IP四层模型

[![TCP/IP四层模型](http://www.vuln.cn/wp-content/uploads/2015/09/2.png)](http://www.vuln.cn/wp-content/uploads/2015/09/2.png)

[![tcp/ip七层模型](http://www.vuln.cn/wp-content/uploads/2015/09/62fde561h7e4e59008066690.jpg)](http://www.vuln.cn/wp-content/uploads/2015/09/62fde561h7e4e59008066690.jpg)

> 网络接口层：公网到达局域网后需要转化为对应的MAC地址。交换机解析判断数据要发给MAC地址对应的哪台电脑。使用的是arp协议。
>
> 网际互联层：网际协议（IP）、互联网组管理协议（IGMP）、互联网控制报文协议（ICMP）（ping的协议）
>
> 传输层：传输控制协议（TCP）（可靠的）、用户数据包协议（UDP）（不可靠的）

### TCP/IP三次握手

ack为回应包，应用为http协议浏览协议，（tcp协议类似打电话沟通）

[![TCP/IP四层模型](http://www.vuln.cn/wp-content/uploads/2015/09/3.png)](http://www.vuln.cn/wp-content/uploads/2015/09/3.png)

   **为什么是三次握手：**

[![TCP/IP四层模型](http://www.vuln.cn/wp-content/uploads/2015/09/4.png)](http://www.vuln.cn/wp-content/uploads/2015/09/4.png) 

UDP协议：传输更快，应用为：qq通信。（类似发短信）

应用层：为用户提供所需的各种服务：例如ftp、www、

## 数据封装过程

[![TCP/IP四层模型](http://www.vuln.cn/wp-content/uploads/2015/09/5.png)](http://www.vuln.cn/wp-content/uploads/2015/09/5.png)

### TCP/IP模型与OSI模型的比较：

**共同点：**

1、OSI参考模型和TCP/IP参考模型都采用了层次结构的概念。
2、都能够提供面向链接也无链接两重通信服务机制。

**不同点：**

1、前者是七层模型，后者是四层结构

2、对可靠性要求不同（后者要求更高）

3、OSI模型是协议开发前设计的，具有通用性，TCP/IP是先有协议集然后建立模型，不适用于非TCP/IP网络。

### IP包头

[![TCP/IP四层模型](http://www.vuln.cn/wp-content/uploads/2015/09/6.png)](http://www.vuln.cn/wp-content/uploads/2015/09/6.png)

因为多了一个选项，所以包头不一定是20个字节，每接收一个数据都要检测这个包头字节多少，比较浪费资源，所以IPV6采用了固定包头。

## IP地址

00000000.00000000.00000000.00000000

11111111.11111111.11111111.11111111

0.0.0.0

255.255.255.255

### IP地址分类

[![TCP/IP四层模型](http://www.vuln.cn/wp-content/uploads/2015/09/7.png)](http://www.vuln.cn/wp-content/uploads/2015/09/7.png)

**其中：**

127.0.0.0网段只有一个ip：127.0.0.1表示本机

 ip第一位数只有从1到223

A类：第一个数固定为一个网段，只有126个网段，一个网段中后三位数可变化，所以主机数多。

B类，前两个数固定为一个网段

C类：前三个数固定为一个网段

### 子网掩码的使用

   子网掩码必须与ip同时使用，只要跟255对应的ip变化，就表示不同的网段；跟0对应的ip变化，就表示同网段下的不同主机。

A类

![img](file:///C:/Users/Sofia/AppData/Local/Temp/enhtmlclip/Image(7).png)[![TCP/IP四层模型](http://www.vuln.cn/wp-content/uploads/2015/09/8.png)](http://www.vuln.cn/wp-content/uploads/2015/09/8.png)

B类

![img](file:///C:/Users/Sofia/AppData/Local/Temp/enhtmlclip/Image(8).png)[![TCP/IP四层模型](http://www.vuln.cn/wp-content/uploads/2015/09/9.png)](http://www.vuln.cn/wp-content/uploads/2015/09/9.png)

C类

[![TCP/IP四层模型](http://www.vuln.cn/wp-content/uploads/2015/09/10.png)](http://www.vuln.cn/wp-content/uploads/2015/09/10.png)

### 变长子网掩码及子网规划

   [![TCP/IP四层模型](http://www.vuln.cn/wp-content/uploads/2015/09/11.png)](http://www.vuln.cn/wp-content/uploads/2015/09/11.png)

B类IP也可以使用C类子网掩码，即前三个数固定为同一网段。

计算方法：全部换算为二进制，上下两个数都为一则等于1，不同则为0，都为0 则等于0；广播地址：子网掩码位为0的，全部换为1得到广播地址。

## 端口的作用

TCP协议包头

[![TCP/IP四层模型](http://www.vuln.cn/wp-content/uploads/2015/09/12.png)](http://www.vuln.cn/wp-content/uploads/2015/09/12.png)

UDP协议包头

[![TCP/IP四层模型](http://www.vuln.cn/wp-content/uploads/2015/09/13.png)](http://www.vuln.cn/wp-content/uploads/2015/09/13.png)

## 网关的作用

网关在我们的一般概念中都是充当路由器的，当然，这是其中之一的功能。

如图

[![网关的作用](http://www.vuln.cn/wp-content/uploads/2015/09/wangguan.jpg)](http://www.vuln.cn/wp-content/uploads/2015/09/wangguan.jpg)