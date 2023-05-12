# 单主从模式RocketMQ 5.1部署 <!-- {docsify-ignore-all} -->

## 下载

下载地址：

https://rocketmq.apache.org/zh/download

## 机器规划

机器A：rocketmq name server & rocketmq dashboard
机器B：rocketmq broker master
机器C：rocketmq broker salve

## 安装并启动rocketmq name server & rocketmq dashboard

### 安装启动单点rocketmq name server

&nbsp; &nbsp; 解压rocketmq的压缩包进入到bin目录下，执行如下命令：

```powershell
nohup sh mqnamesrv &
```

### 安装启动rocketmq dashboard

&nbsp; &nbsp; 下载dashboard是源码下载的，所以这里要自己打包成jar，所以需要你得机器上有maven，在打包之前修改dashboard的application.properties配置文件，如下：

这里主要配置dashboard端口和nameserver address

```properties
server.address=0.0.0.0
# 端口根据实际情况可以修改
server.port=8080

### SSL setting
#server.ssl.key-store=classpath:rmqcngkeystore.jks
#server.ssl.key-store-password=rocketmq
#server.ssl.keyStoreType=PKCS12
#server.ssl.keyAlias=rmqcngkey

#spring.application.index=true
spring.application.name=rocketmq-dashboard
spring.http.encoding.charset=UTF-8
spring.http.encoding.enabled=true
spring.http.encoding.force=true
logging.level.root=INFO
logging.config=classpath:logback.xml
#if this value is empty,use env value rocketmq.config.namesrvAddr  NAMESRV_ADDR | now, you can set it in ops page.default localhost:9876
# name server address 如果不配置默认是localhost:9876
rocketmq.config.namesrvAddr=
#if you use rocketmq version < 3.5.8, rocketmq.config.isVIPChannel should be false.default true
rocketmq.config.isVIPChannel=
#timeout for mqadminExt, default 5000ms
rocketmq.config.timeoutMillis=
#rocketmq-console's data path:dashboard/monitor
rocketmq.config.dataPath=/tmp/rocketmq-console/data
#set it false if you don't want use dashboard.default true
rocketmq.config.enableDashBoardCollect=true
#set the message track trace topic if you don't want use the default one
rocketmq.config.msgTrackTopicName=
rocketmq.config.ticketKey=ticket

#Must create userInfo file: ${rocketmq.config.dataPath}/users.properties if the login is required
rocketmq.config.loginRequired=false

#set the accessKey and secretKey if you used acl
#rocketmq.config.accessKey=
#rocketmq.config.secretKey=
rocketmq.config.useTLS=false
```

## Broker和Salve部署

### Master Broker部署

#### 创建程序目录及存储路径

- 创建程序目录并解压程序

```powershell
mkdir /usr/local/rocketmq
```

- 创建存储路径

```powershell
mkdir /usr/local/rocketmq/store
mkdir /usr/local/rocketmq/store/commitlog
mkdir /usr/local/rocketmq/store/consumequeue
mkdir /usr/local/rocketmq/store/index
```

#### Master配置文件修改

- 创建1主1从的文件夹存放配置文件

```powershell
mkdir /usr/local/rocketmq/conf/1m-1s-async
```

- 创建Master节点配置文件，配置内容如下

```properties
brokerClusterName=DefaultCluster
brokerName=broker-a
brokerId=0
deleteWhen=04
fileReservedTime=48
#broker角色
brokerRole=ASYNC_MASTER
flushDiskType=ASYNC_FLUSH
namesrvAddr=xx.xx.xx.xx:9876
#当前broker的ip
brokerIP1=xx.xx.xx.xx
#存储路径
storePathRootDir=/usr/local/rocketmq/store
#commitLog 存储路径
storePathCommitLog=/usr/local/rocketmq/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/usr/local/rocketmq/store/consumequeue
#消息索引存储路径
storePathIndex=/usr/local/rocketmq/store/index
#checkpoint 文件存储路径
storeCheckpoint=/usr/local/rocketmq/store/checkpoint
#abort 文件存储路径
abortFile=/usr/local/rocketmq/store/abort
```

#### 启动Master节点

注意：RocketMQ5.0增加了proxy模块，本文将broker和proxy在一起部署了，所以这个命令要为proxy指定namesrv的地址，否则无法启动。

```powershell
nohup sh bin/mqbroker -n xx.xx.xx.xx:9876 -c /usr/local/rocketmq/conf/1m-1s-async/broker-a.properties --enable-proxy &
```

启动成功会查看日志会有 boot success 的日志

### Salve Broker部署

&nbsp; &nbsp; Salve的部署和Broker类似，在机器C上也要创建程序目录及存储路径，这里不多赘述和Master节点一样。

#### Salve节点配置文件修改

- 创建1主1从的文件夹存放配置文件

```powershell
mkdir /usr/local/rocketmq/conf/1m-1s-async
```

- 创建Salve节点配置文件

&nbsp; &nbsp; 配置文件有一些变化，主要就是指定角色，brokerId等，具体如下：

```properties
brokerClusterName=DefaultCluster
brokerName=broker-a
brokerId=1
deleteWhen=04
fileReservedTime=48
brokerRole=SLAVE
flushDiskType=ASYNC_FLUSH
#启动sql过滤
enablePropertyFilter=true
namesrvAddr=192.168.246.183:9876
#存储路径
storePathRootDir=/usr/local/rocketmq/store
#commitLog 存储路径
storePathCommitLog=/usr/local/rocketmq/store/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/usr/local/rocketmq/store/consumequeue
#消息索引存储路径
storePathIndex=/usr/local/rocketmq/store/index
#checkpoint 文件存储路径
storeCheckpoint=/usr/local/rocketmq/store/checkpoint
#abort 文件存储路径
abortFile=/usr/local/rocketmq/store/abort
```


#### 启动Salve节点

注意：RocketMQ5.0增加了proxy模块，本文将broker和proxy在一起部署了，所以这个命令要为proxy指定namesrv的地址，否则无法启动。

```powershell
nohup sh bin/mqbroker -n xx.xx.xx.xx:9876 -c /usr/local/rocketmq/conf/1m-1s-async/broker-a-s.properties --enable-proxy &
```

启动成功会查看日志会有 boot success 的日志