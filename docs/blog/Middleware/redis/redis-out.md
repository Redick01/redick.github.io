# Redis缓存淘汰

> volatile-lru：从已设置过期时间的数据集（server. db[i]. expires）中挑选最近最少使用的数据淘汰。
> volatile-ttl：从已设置过期时间的数据集（server. db[i]. expires）中挑选将要过期的数据淘汰。
> volatile-random：从已设置过期时间的数据集（server. db[i]. expires）中任意选择数据淘汰。
> allkeys-lru：从数据集（server. db[i]. dict）中挑选最近最少使用的数据淘汰。
> allkeys-random：从数据集（server. db[i]. dict）中任意选择数据淘汰。
> no-enviction（驱逐）：禁止驱逐数据。





redis集群三种方式的优缺点

Redis集群有三种工作模式：主从模式（Master-Slave）、哨兵模式（Sentinel）和Cluster模式。

1. 主从模式：

- 优点：数据备份，读写分离，容灾恢复。
- 缺点：故障转移（故障转移依赖于Sentinel）不是自动的，需手动干预。

1. 哨兵模式：

- 优点：自动故障转移，可以实现高可用。
- 缺点：哨兵节点不能分担主节点的读压力，Cluster模式可以。

1. Cluster模式：

- 优点：自动分片数据，可以线性扩展，没有10000个key的限制，可以平行读写。
- 缺点：不支持多个客户端对同一个key进行原子性操作。