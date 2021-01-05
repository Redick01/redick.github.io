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

## Redis Cluster