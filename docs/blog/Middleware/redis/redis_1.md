# redis集群与高可用

## Docker搭建redis主从复制

### 一.安装Redis
    1.搜索redis镜像
        docker search redis
    2.拉取镜像
        docker pull redis
### 二.主从复制
#### 1.运行docker镜像，首先使用docker启动3个redis容器服务，分别使用到6379、6380、6381端口
    docker run --name redis-6379 -p 6379:6379 -d redis redis-server

    docker run --name redis-6380 -p 6380:6379 -d redis redis-server

    docker run --name redis-6381 -p 6381:6379 -d redis redis-server

#### 2.配置redis集群
    使用如下命令查看容器内网的ip地址等信息
    docker inspect containerid（容器ID），下面是执行命令后的信息片段，IPAddress即容器内网IP。

    "SandboxKey": "/var/run/docker/netns/6db03f3155e4",
            "SecondaryIPAddresses": null,
            "SecondaryIPv6Addresses": null,
            "EndpointID": "a9d1954bd522aa5fe0432e62c0454ca49fc06a907c9ecb8d6ecd105a43d7d103",
            "Gateway": "172.17.0.1",
            "GlobalIPv6Address": "",
            "GlobalIPv6PrefixLen": 0,
            "IPAddress": "172.17.0.5",
            "IPPrefixLen": 16,
            "IPv6Gateway": "",
            "MacAddress": "02:42:ac:11:00:05",
            "Networks": {
                "bridge": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": null,
                    "NetworkID": "42c4ad50844ace06fb7d67efa4c332911981ac1432ab7783acd5ad2745446d2a",
                    "EndpointID": "a9d1954bd522aa5fe0432e62c0454ca49fc06a907c9ecb8d6ecd105a43d7d103",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.5",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:11:00:05",
                    "DriverOpts": null
                }
            }
#### 3.我的docker三个redis的内网IP
    redis-6381  172.17.0.5
    redis-6380  172.17.0.4
    redis-6379  172.17.0.3
#### 4.进入docker容器内部（docker exec -it <container_id> redis-cli），查看当前redis角色（是master还是slave）（命令：info replication）
    # Replication
    role:master
    connected_slaves:0
    master_replid:590672be7b831c0b5e7c353a7b6228fbd5e1ec92
    master_replid2:0000000000000000000000000000000000000000
    master_repl_offset:0
    second_repl_offset:-1
    repl_backlog_active:0
    repl_backlog_size:1048576
    repl_backlog_first_byte_offset:0
    repl_backlog_histlen:0

可以看到，其实三台其实都是master角色；使用redis-cli命令修改redis-6380、redis-6381的主机为redis-6379
    
    SLAVEOF 172.17.0.3 6379

再次查看reids-6379主机的info，已经有两个从机了。

    # Replication
    role:master
    connected_slaves:2
    slave0:ip=172.17.0.4,port=6379,state=online,offset=112,lag=1
    slave1:ip=172.17.0.5,port=6379,state=online,offset=112,lag=1
    master_replid:91695f20fd0dadb32635b922da4315c79d626b7e
    master_replid2:0000000000000000000000000000000000000000
    master_repl_offset:112
    second_repl_offset:-1
    repl_backlog_active:1
    repl_backlog_size:1048576
    repl_backlog_first_byte_offset:1
    repl_backlog_histlen:112

#### 5.测试主从效果

主节点：

    127.0.0.1:6379> set ph redick
    OK
    127.0.0.1:6379> get ph
    "redick"
    127.0.0.1:6379>
从节点1:

    127.0.0.1:6379> get ph
    "redick"
从节点2:

    127.0.0.1:6379> get ph
    "redick"
可以看到从节点的端口号显示和主节点一样，为了验证是的确登陆的是从节点的redis-cli我们在从节点set一个值试一下：

    从1：
    127.0.0.1:6379> set k kkk
    (error) READONLY You can't write against a read only replica.

    从2：
    127.0.0.1:6379> set k kk
    (error) READONLY You can't write against a read only replica.

#### 至此，redis通过docker搭建主从已经完成

## Redis Sentinel（哨兵）配置

### Sentinel背景
      之前介绍了用docker来搭建redis主从环境，但这只是对数据添加了从库备份(主从复制)，当主库down掉的时候，从库是不会自动升级为主库的，也就是说，该redis主从集群并非是高可用的。
      目前来说，高可用(主从复制、主从切换)redis集群有两种方案，一种是redis-sentinel，只有一个master，各实例数据保持一致；一种是redis-cluster，也叫分布式redis集群，可以有多个master，数据分片分布在这些master上。
      本文介绍基于docker和redis-sentinel的高可用redis集群搭建，大多数情况下，redis-sentinel也需要做高可用，这里先对redis搭建一主二从环境，另外需要3个redis-sentinel监控redis master。

      很显然，只使用单个redis-sentinel进程来监控redis集群是不可靠的，由于redis-sentinel本身也有single-point-of-failure-problem(单点问题)，当出现问题时整个redis集群系统将无法按照预期的方式切换主从。官方推荐：一个健康的集群部署，至少需要3个Sentinel实例。另外，redis-sentinel只需要配置监控redis master，而集群之间可以通过master相互通信。
### redis-sentinel
    redis-sentinel作为独立的服务，用于管理多个redis实例，该系统主要执行以下三个任务：

- 监控 (Monitor): 检查redis主、从实例是否正常运作
- 通知 (Notification): 监控的redis服务出现问题时，可通过API发送通知告警
- 自动故障迁移 (Automatic Failover): 当检测到redis主库不能正常工作时，redis-sentinel会开始做自动故障判断、迁移等操作，先是移除失效redis主服务，然后将其中一个从服务器升级为新的主服务器，并让失效主服务器的其他从服务器改为复制新的主服务器。当客户端试图连接失效的主服务器时，集群也会向客户端返回最新主服务器的地址，使得集群可以使用新的主服务器来代替失效服务器

### sentinel.conf配置文件
    sentinel.conf是启动redis-sentinel的核心配置文件，可以从官网下载，Redis的安装包默认也是有的。

### 配置sentinel.conf
```
    # mymaster:自定义集群名，如果需要监控多个redis集群，只需要配置多次并定义不同的<master-name> <master-redis-ip>:主库ip <master-redis-port>:主库port <quorum>:最小投票数，由于有三台redis-sentinel实例，所以可以设置成2
    sentinel monitor mymaster <master-redis-ip> <master-redis-port> <quorum>
    
    # 注：redis主从库搭建的时候，要么都不配置密码(这样下面这句也不需要配置)，不然都需要设置成一样的密码
    sentinel auth-pass mymaster redispassword
    
    # 添加后台运行
    daemonize yes
```
将sentinel.conf配置文件复制三份，sentinel1.conf，sentinel2.conf和sentinel3.conf，分别编辑三个配置文件的port，分别为26379，26380，26381。

### 分别启动三个Sentinel
    redis-sentinel sentinel_1.conf
    redis-sentinel sentinel_2.conf
    redis-sentinel sentinel_3.conf

### 使用redis客户端，连接redis-sentinel API查看监控状况

    redis-cli -p 26379 (26380 | 26381)
    sentinel master mymaster 或 sentinel slaves mymaster

     1) "name"
     2) "mymaster"
     3) "ip"
     4) "192.168.0.104"
     5) "port"
     6) "6379"
     7) "runid"
     8) "3f6078b50fb2b63a2b05d26f60370285ca5c21c7"
     9) "flags"
    1)  "master"
    2)  "link-pending-commands"
    3)  "0"
    4)  "link-refcount"
    5)  "1"
    6)  "last-ping-sent"
    7)  "0"
    8)  "last-ok-ping-reply"
    9)  "178"
    10) "last-ping-reply"
    11) "178"
    12) "down-after-milliseconds"
    13) "5000"
    14) "info-refresh"
    15) "3936"
    16) "role-reported"
    17) "master"
    18) "role-reported-time"
    19) "144400"
    20) "config-epoch"
    21) "0"
    22) "num-slaves"
    23) "2"
    24) "num-other-sentinels"
    25) "2"
    26) "quorum"
    27) "2"
    28) "failover-timeout"
    29) "900000"
    30) "parallel-syncs"
    31) "2"

### 测试

    进入redis-master容器，休眠60秒redis服务：
    docker exec -it redis-master bash
    redis-cli -h <host> -p 6300 DEBUG sleep 60

进入redis-slave或redis-slave2容器，查看info Replication，可以看到master已经完成了切换。

redis-sentinel输出主库切换信息
  60秒后原redis主库恢复服务，但降级后当前redis服务已无法恢复原主库身份。

### 至此reids sentinel配置完成

## 基于Docker安装配置Redis Cluster

### 节点规划
    三主三从

| 容器名称 | 容器IP | 映射端口 | 运行模式 | 
| ---- | ---- | ------------ | ----- |
| redis-master-1 | 172.50.0.2 | 6391 -> 6391 16391 -> 16391 | master |
| redis-master-2 | 172.50.0.3 | 6392 -> 6392 16392 -> 16392 | master |
| redis-master-3 | 172.50.0.4 | 6393 -> 6393 16393 -> 16393 | master |
| redis-slave-1 | 172.30.0.2 | 6394 -> 6394 16391 -> 16394 | slave |
| redis-slave-2 | 172.30.0.3 | 6395 -> 6395 16391 -> 16395 | slave |
| redis-slave-3 | 172.30.0.4 | 6396 -> 6396 16391 -> 16396 | slave |

### 编写docker-compose.yml和redis.conf
- docker-compose.yml
```
version: "3.6"
services:
  redis-master-1: 
     image: redis:5.0 # 基础镜像
     container_name: redis-master-1 # 容器服务名
     working_dir: /config # 工作目录
     environment: # 环境变量
       - PORT=6391 # 跟 config/nodes-6391.conf 里的配置一样的端口
     ports: # 映射端口，对外提供服务
       - "6391:6391" # redis 的服务端口
       - "16391:16391" # redis 集群监控端口
     stdin_open: true # 标准输入打开
     networks: # docker 网络设置
        redis-master:
            ipv4_address: 172.50.0.2
     tty: true
     privileged: true # 拥有容器内命令执行的权限
     volumes: ["/Users/penghuiliu/docker-file/docker-redis/docker-redis-cluster-master/config:/config"] # 映射数据卷，配置目录
     entrypoint: # 设置服务默认的启动程序
       - /bin/bash
       - redis.sh
  redis-master-2:
       image: redis:5.0
       working_dir: /config
       container_name: redis-master-2
       environment:
              - PORT=6392
       networks:
          redis-master:
             ipv4_address: 172.50.0.3
       ports:
         - "6392:6392"
         - "16392:16392"
       stdin_open: true
       tty: true
       privileged: true
       volumes: ["/Users/penghuiliu/docker-file/docker-redis/docker-redis-cluster-master/config:/config"]
       entrypoint:
         - /bin/bash
         - redis.sh
  redis-master-3:
       image: redis:5.0
       container_name: redis-master-3
       working_dir: /config
       environment:
              - PORT=6393
       networks:
          redis-master:
            ipv4_address: 172.50.0.4
       ports:
         - "6393:6393"
         - "16393:16393"
       stdin_open: true
       tty: true
       privileged: true
       volumes: ["/Users/penghuiliu/docker-file/docker-redis/docker-redis-cluster-master/config:/config"]
       entrypoint:
         - /bin/bash
         - redis.sh
  redis-slave-1:
       image: redis:5.0
       container_name: redis-slave-1
       working_dir: /config
       environment:
            - PORT=6394
       networks:
          redis-slave:
             ipv4_address: 172.30.0.2
       ports:
         - "6394:6394"
         - "16394:16394"
       stdin_open: true
       tty: true
       privileged: true
       volumes: ["/Users/penghuiliu/docker-file/docker-redis/docker-redis-cluster-master/config:/config"]
       entrypoint:
         - /bin/bash
         - redis.sh
  redis-salve-2:
       image: redis:5.0
       working_dir: /config
       container_name: redis-salve-2
       environment:
             - PORT=6395
       ports:
         - "6395:6395"
         - "16395:16395"
       stdin_open: true
       networks:
          redis-slave:
              ipv4_address: 172.30.0.3
       tty: true
       privileged: true
       volumes: ["/Users/penghuiliu/docker-file/docker-redis/docker-redis-cluster-master/config:/config"]
       entrypoint:
         - /bin/bash
         - redis.sh
  redis-salve-3:
       image: redis:5.0
       container_name: redis-slave-3
       working_dir: /config
       environment:
          - PORT=6396
       ports:
         - "6396:6396"
         - "16396:16396"
       stdin_open: true
       networks:
          redis-slave:
            ipv4_address: 172.30.0.4
       tty: true
       privileged: true
       volumes: ["/Users/penghuiliu/docker-file/docker-redis/docker-redis-cluster-master/config:/config"]
       entrypoint:
         - /bin/bash
         - redis.sh
networks:
  redis-master:
     driver: bridge
     ipam:
       driver: default
       config:
          -
           subnet: 172.50.0.0/16
  redis-slave:
       driver: bridge
       ipam:
         driver: default
         config:
            -
             subnet: 172.30.0.0/16
```

### 启动redis集群
    启动：docker-compose up -d
    查看：docker ps/docker-compose ps

### 初始化集群
    1.宿主机IP：ifconfig，我们己IP：192.168.3.78

    2.查找redis-master-1的容器id：2d4596b188a8
    liupenghui:~ penghuiliu$ docker ps
    CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                                        NAMES
    58e97782e2fe        redis:5.0           "/bin/bash redis.sh"     2 minutes ago       Up 2 minutes        0.0.0.0:6393->6393/tcp, 6379/tcp, 0.0.0.0:16393->16393/tcp   redis-master-3
    f9bd7e330f82        redis:5.0           "/bin/bash redis.sh"     2 minutes ago       Up 2 minutes        0.0.0.0:6392->6392/tcp, 6379/tcp, 0.0.0.0:16392->16392/tcp   redis-master-2
    2d4596b188a8        redis:5.0           "/bin/bash redis.sh"     2 minutes ago       Up 2 minutes        0.0.0.0:6391->6391/tcp, 6379/tcp, 0.0.0.0:16391->16391/tcp   redis-master-1
    7b42859712fd        redis:5.0           "/bin/bash redis.sh"     2 minutes ago       Up 2 minutes        0.0.0.0:6395->6395/tcp, 6379/tcp, 0.0.0.0:16395->16395/tcp   redis-salve-2
    3becb97b314c        redis:5.0           "/bin/bash redis.sh"     2 minutes ago       Up 2 minutes        0.0.0.0:6396->6396/tcp, 6379/tcp, 0.0.0.0:16396->16396/tcp   redis-slave-3
    ae2ee02885e4        redis:5.0           "/bin/bash redis.sh"     2 minutes ago       Up 2 minutes        0.0.0.0:6394->6394/tcp, 6379/tcp, 0.0.0.0:16394->16394/tcp   redis-slave-1
    af951c611b57        ea93faa92337        "/docker-entrypoint.…"   3 weeks ago         Up 7 days           2888/tcp, 3888/tcp, 0.0.0.0:2181->2181/tcp, 8080/tcp         some-zookeeper
    fc9441156e5b        mysql:5.7           "docker-entrypoint.s…"   4 weeks ago         Up 7 days           33060/tcp, 0.0.0.0:3316->3306/tcp                            mysql-master
    c13e1e2cf151        mysql:5.7           "docker-entrypoint.s…"   4 weeks ago         Up 7 days           33060/tcp, 0.0.0.0:3326->3306/tcp                            mysql-slave

    3.进入master-1容器：
    liupenghui:~ penghuiliu$ docker exec -it 2d4596b188a8 /bin/bash

    4.创建三主三从的redis集群：
    root@2d4596b188a8:/config# redis-cli --cluster create 192.168.3.78:6391 192.168.3.78:6392 192.168.3.78:6393 192.168.3.78:6394 192.168.3.78:6395 192.168.3.78:6396 --cluster-replicas 1

    中途输入 “yes”确认初始化
    root@2d4596b188a8:/config# redis-cli --cluster create 192.168.3.78:6391 192.168.3.78:6392 192.168.3.78:6393 192.168.3.78:6394 192.168.3.78:6395 192.168.3.78:6396 --cluster-replicas 1
    >>> Performing hash slots allocation on 6 nodes...
    Master[0] -> Slots 0 - 5460
    Master[1] -> Slots 5461 - 10922
    Master[2] -> Slots 10923 - 16383
    Adding replica 192.168.3.78:6395 to 192.168.3.78:6391
    Adding replica 192.168.3.78:6396 to 192.168.3.78:6392
    Adding replica 192.168.3.78:6394 to 192.168.3.78:6393
    >>> Trying to optimize slaves allocation for anti-affinity
    [WARNING] Some slaves are in the same host as their master
    M: c66af09ca2a87bed9523dc449e3cc09a5071f10f 192.168.3.78:6391
    slots:[0-5460] (5461 slots) master
    M: 893b57a4360df40ce3bf40dbb16f53d1ad85b900 192.168.3.78:6392
    slots:[5461-10922] (5462 slots) master
    M: d0e40d450becd47a3a412973b3acedfd51740796 192.168.3.78:6393
    slots:[10923-16383] (5461 slots) master
    S: e7758429d2668d38f11ba22da75e88fa2f52ddb3 192.168.3.78:6394
    replicates 893b57a4360df40ce3bf40dbb16f53d1ad85b900
    S: b13e68cf772bf5591cb1a9ada8036f27010a89f2 192.168.3.78:6395
    replicates d0e40d450becd47a3a412973b3acedfd51740796
    S: b823fdd7f14baae537ec04ea6245da3024292497 192.168.3.78:6396
    replicates c66af09ca2a87bed9523dc449e3cc09a5071f10f
    Can I set the above configuration? (type 'yes' to accept): yes
    >>> Nodes configuration updated
    >>> Assign a different config epoch to each node
    >>> Sending CLUSTER MEET messages to join the cluster
    Waiting for the cluster to join
    .
    >>> Performing Cluster Check (using node 192.168.3.78:6391)
    M: c66af09ca2a87bed9523dc449e3cc09a5071f10f 192.168.3.78:6391
    slots:[0-5460] (5461 slots) master
    1 additional replica(s)
    S: e7758429d2668d38f11ba22da75e88fa2f52ddb3 172.50.0.1:6394
    slots: (0 slots) slave
    replicates 893b57a4360df40ce3bf40dbb16f53d1ad85b900
    M: 893b57a4360df40ce3bf40dbb16f53d1ad85b900 172.50.0.1:6392
    slots:[5461-10922] (5462 slots) master
    1 additional replica(s)
    S: b13e68cf772bf5591cb1a9ada8036f27010a89f2 172.50.0.1:6395
    slots: (0 slots) slave
    replicates d0e40d450becd47a3a412973b3acedfd51740796
    S: b823fdd7f14baae537ec04ea6245da3024292497 172.50.0.1:6396
    slots: (0 slots) slave
    replicates c66af09ca2a87bed9523dc449e3cc09a5071f10f
    M: d0e40d450becd47a3a412973b3acedfd51740796 172.50.0.1:6393
    slots:[10923-16383] (5461 slots) master
    1 additional replica(s)
    [OK] All nodes agree about slots configuration.
    >>> Check for open slots...
    >>> Check slots coverage...
    [OK] All 16384 slots covered.

    5.查看redis初始化结果：
    root@2d4596b188a8:/config# redis-cli -c -h 192.168.3.78 -p 6391
    192.168.3.78:6391> cluster nodes
    e7758429d2668d38f11ba22da75e88fa2f52ddb3 172.50.0.1:6394@16394 slave 893b57a4360df40ce3bf40dbb16f53d1ad85b900 0 1610008020447 4 connected
    c66af09ca2a87bed9523dc449e3cc09a5071f10f 172.50.0.2:6391@16391 myself,master - 0 1610008019000 1 connected 0-5460
    893b57a4360df40ce3bf40dbb16f53d1ad85b900 172.50.0.1:6392@16392 master - 0 1610008019000 2 connected 5461-10922
    b13e68cf772bf5591cb1a9ada8036f27010a89f2 172.50.0.1:6395@16395 slave d0e40d450becd47a3a412973b3acedfd51740796 0 1610008021459 5 connected
    b823fdd7f14baae537ec04ea6245da3024292497 172.50.0.1:6396@16396 slave c66af09ca2a87bed9523dc449e3cc09a5071f10f 0 1610008018414 6 connected
    d0e40d450becd47a3a412973b3acedfd51740796 172.50.0.1:6393@16393 master - 0 1610008019430 3 connected 10923-16383

    6.测试集群
    192.168.3.78:6391> set test testvalue
    -> Redirected to slot [6918] located at 172.50.0.1:6392
    OK

    test哈希槽计算后，被分配到了6392 服务上，所以会转到6392服务器上，在6392获取test的值
    172.50.0.1:6393> get test
    -> Redirected to slot [6918] located at 172.50.0.1:6392
    "testvalue"

### 至此使用docker部署配置redis cluster完成
