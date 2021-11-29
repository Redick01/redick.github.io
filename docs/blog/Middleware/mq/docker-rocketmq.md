docker search rocketmq

docker pull rocketmqinc/rocketmq


docker run -d -p 9876:9876 -v /tmp/data/rocketmq/namesrv/logs:/root/logs -v /tmp/data/rocketmq/namesrv/store:/root/store --name rmqnamesrv -e "MAX_POSSIBLE_HEAP=100000000" rocketmqinc/rocketmq sh mqnamesrv


brokerClusterName = DefaultCluster
brokerName = broker-a
brokerId = 0
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH
# 如果是本地程序调用云主机 mq，这个需要设置成 云主机 IP
# 如果Docker环境需要设置成宿主机IP
brokerIP1 = 

docker run -d -p 10911:10911 -p 10909:10909 -v  /tmp/data/rocketmq/broker/logs:/root/logs -v  /tmp/data/rocketmq/broker/store:/root/store -v  /tmp/etc/rocketmq/broker/broker.conf:/opt/rocketmq/conf/broker.conf --name rmqbroker --link rmqnamesrv:namesrv -e "NAMESRV_ADDR=namesrv:9876" rocketmqinc/rocketmq sh mqbroker -c /opt/rocketmq/conf/broker.conf


docker search rocketmq-console

docker pull styletang/rocketmq-console-ng

docker run -d -p 8080:8080 -e "JAVA_OPTS=-Drocketmq.config.namesrvAddr=192.168.58.45:9876 -Drocketmq.config.isVIPChannel=false" styletang/rocketmq-console-ng
