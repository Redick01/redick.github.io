# 单主从模式RocketMQ 5.1部署 <!-- {docsify-ignore-all} -->

## 下载

下载地址：

https://rocketmq.apache.org/zh/download

## 机器规划

机器A：rocketmq name server & rocketmq dashboard
机器B：rocketmq broker master
机器C：rocketmq broker salve

## 安装并启动rocketmq name server & rocketmq dashboard

### 安装启动rocketmq name server

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
