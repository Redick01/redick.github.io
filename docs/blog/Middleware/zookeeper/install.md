 # 文档名称：三节点zookeeper服务安装配置说明


## 功能说明
ZooKeeper是一个分布式的，开放源码的分布式应用程序协调服务，它包含一个简单的原语集，分布式应用程序可以基于它实现同步服务，配置维护和命名服务等。Zookeeper是hadoop的一个子项目，其发展历程无需赘述。在分布式应用中，由于工程师不能很好地使用锁机制，以及基于消息的协调机制不适合在某些应用中使用，因此需要有一种可靠的、可扩展的、分布式的、可配置的协调机制来统一系统的状态。Zookeeper的目的就在于此。
***

## 实现原理
Zookeeper的核心是原子广播，这个机制保证了各个Server之间的同步。实现这个机制的协议叫做Zab协议。Zab协议有两种模式，它们分别是恢复模式（选主）和广播模式（同步）。当服务启动或者在领导者崩溃后，Zab就进入了恢复模式，当领导者被选举出来，且大多数Server完成了和leader的状态同步以后，恢复模式就结束了。状态同步保证了leader和Server具有相同的系统状态。

为了保证事务的顺序一致性，zookeeper采用了递增的事务id号（zxid）来标识事务。所有的提议（proposal）都在被提出的时候加上了zxid。实现中zxid是一个64位的数字，它高32位是epoch用来标识leader关系是否改变，每次一个leader被选出来，它都会有一个新的epoch，标识当前属于那个leader的统治时期。低32位用于递增计数。

每个Server在工作过程中有三种状态：

- LOOKING：当前Server不知道leader是谁，正在搜寻

- LEADING：当前Server即为选举出来的leader

- FOLLOWING：leader已经选举出来，当前Server与之同步

***

## 安装信息
### 安装环境
- **os**:     centos7.3	最小化安装
- **server:** 自己准备三台服务器
- **版本：** zookeeper-3.4.6.tar.gz
- **java:** jdk7/jdk8

### 安装方法
- 解压安装包

      tar -zxvf zookeeper-3.4.6.tar.gz -C /usr/local
      ln -s /usr/local/zookeeper-3.4.6 /usr/local/zookeeper

- 创建logs,data目录(每台机器的zookeeper目录下)
  
      mkdir /usr/local/zookeeper/data
      mkdir /usr/local/zookeeper/logs

- 复制配置文件

      cd /usr/local/zookeeper-3.4.6/conf
      cp zoo_sample.cfg zoo.cfg
- 修改配置文件(具体看下面的配置文件解释)
- 创建myid文件

    在每个data文件夹下创建一个文件名称为myid，文件的内容就是此zookeeper的编号1、2、3 
      
      注意：myid文件要自己创建，在dataDir目录下
      $echo "1">>/usr/local/zookeeper/data/myid(剩下以此类推)
      
***

## 配置信息

### 目录及文件说明
- **安装目录：**
    ```
    /usr/local/zookerper-3.4.6
    ```
- **目录说明：**
    ```
    bin:启动文件
    conf:配置文件
    logs:存放日志
    ```
### 配置文件说明

配置文件需要在每台服务器中都要编写，以下是一个配置文件的样本：
```
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/usr/local/zookeeper/data
dataLogDir=/usr/local/zookeeper/logs
clientPort=2181
autopurge.snapRetainCount=90
autopurge.purgeInterval=24
server.1=zk1:2888:3888
server.2=zk2:2888:3888
server.3=zk3:2888:3888
```
zoo.cfg 配置项说明如下：
    
    tickTime：这个时间是作为zookeeper服务器之间或客户端与服务器之间维持心跳的时间间隔,也就是说每个tickTime时间就会发送一个心跳。
    initLimit：这个配置项是用来配置zookeeper接受客户端（这里所说的客户端不是用户连接zookeeper服务器的客户端,而是zookeeper服务器集群中连接到leader的follower 服务器）初始化连接时最长能忍受多少个心跳时间间隔数。
              当已经超过10个心跳的时间（也就是tickTime）长度后 zookeeper 服务器还没有收到客户端的返回信息,那么表明这个客户端连接失败。总的时间长度就是 10*2000=20秒。
    syncLimit：这个配置项标识leader与follower之间发送消息,请求和应答时间长度,最长不能超过多少个tickTime的时间长度,总的时间长度就是5*2000=10秒。
    dataDir：顾名思义就是zookeeper保存数据的目录,默认情况下zookeeper将写数据的日志文件也保存在这个目录里；
    clientPort：这个端口就是客户端连接Zookeeper服务器的端口,Zookeeper会监听这个端口接受客户端的访问请求；
    server.A=B:C:D中的A是一个数字,表示这个是第几号服务器,B是这个服务器的IP地址，C第一个端口用来集群成员的信息交换,表示这个服务器与集群中的leader服务器交换信息的端口，D是在leader挂掉时专门用来进行选举leader所用的端口。
    autopurge.snapRetainCount：这个参数和下面的参数搭配使用，这个参数指定了需要保留的文件数目。默认是保留3个
    autopurge.purgeInterval：ZK提供了自动清理事务日志和快照文件的功能，这个参数指定了清理频率，单位是小时，需要配置一个1或更大的整数，默认是0，表示不开启自动清理功能
***

## 使用方法
- 启动
```
/usr/local/zookeeper/bin/zkServer.sh start
```
- 停止
```
/usr/local/zookeeper/bin/zkServer.sh stop
```
- 查看服务状态
```
/usr/local/zookeeper/bin/zkServer.sh status 
```


***



    