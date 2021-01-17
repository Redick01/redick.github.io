# 各种消息队列中间件的安装与简单测试

## RabbitMQ

- **安装**

- - 直接安装

- - - macos：brew install rabbitmq
- - - linux：apt/yum installrabbit-serve

- - docker 安装

    ```
    # pull镜像 注意不带后缀就不会有web控制台
    docker pull rabbitmq:management
    # 运行镜像
    docker run -itd --name rabbitmq-test -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=admin -p 15672:15672 -p 5672:5672 rabbitmq:management 
    # 进入容器
    docker exec -it rabbitmq-test /bin/bash
    # 进入容器后操作
    > rabbitmqctl list_queues、rabbitmqctl status 
    > rabbitmqadmin declare queue name=kk01 -u admin -p admin 
    > rabbitmqadmin get queue=kk01 -u admin -p admin
    ```

- - 简单测试 结成spring-amqp操作rabbitmq测试

    参考：https://blog.csdn.net/qq_38455201/article/details/80308771
         https://www.cnblogs.com/handsomeye/p/9135623.html

## Pulsar

- **Pulsar介绍**


- **安装**

- - 下载安装

    1、下载安装 通过 http://pulsar.apache.org/zh-CN/download/ 下载2.7.0版本 解压压缩包，即可。详细文档可以参见：http://pulsar.apache.org/docs/zh-CN/ 
    ```
    > bin/pulsar standalone 
    > bin/pulsar-client consume topic1 -s "first-subscription" 
    > bin/pulsar-client produce topic1 --messages "hello-pulsar"
    ```
- - docker安装

    ```
    # 运行不起来，自动就pull了
    docker run -it \
    -p 6650:6650 \
    -p 8080:8080 \
    --mount source=pulsardata,target=/pulsar/data \
    --mount source=pulsarconf,target=/pulsar/conf \
    apachepulsar/pulsar:2.7.0 \
    bin/pulsar standalone
    ```
    
  