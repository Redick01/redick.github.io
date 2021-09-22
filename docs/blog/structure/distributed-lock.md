总结一下，实现分布式的方式：
1）数据库for update，
2）redis的过期key/redission，
3）zookeeper临时节点，
4）etcd的lease，
5）内存网格hazelcast和ignite的纯java分布式锁。