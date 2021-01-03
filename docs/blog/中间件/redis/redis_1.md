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