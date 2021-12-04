# 基于Docker部署Skywalking - 单机版 <!-- {docsify-ignore-all} -->

## Skywalking简介

&nbsp; &nbsp; `Skywalking`是分布式系统的应用程序性能监视工具，专为微服务、云原生架构和基于容器（Docker、K8s、Mesos）架构而设计。提供分布式追踪、服务网格遥测分析、度量聚合和可视化一体化解决方案。

## Docker部署Skywalking步骤

- **部署Elasticsearch**
- **部署Skywalking OAP**
- **部署Skywalking UI**
- **应用程序配合Skywalking Agent部署**

## 部署Elasticsearch

&nbsp; &nbsp; 这里基于`Docker`简单单机部署，普通部署和集群部署可以参考[官方文档](https://docs.es.shiyueshuyi.xyz/#/setup/install/linux)。

#### 拉取镜像

```shell
docker pull elasticsearch:7.6.2
```

#### 指定单机启动


<span style="color:red;">注：通过ES_JAVA_OPTS设置ES初始化内存，否则在验证时可能会起不来</span>

```shell
docker run --restart=always -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" \
-e ES_JAVA_OPTS="-Xms512m -Xmx512m" \
--name='elasticsearch' --cpuset-cpus="1" -m 2G -d elasticsearch:7.6.2
```

## 部署Skywalking OAP

#### 拉取镜像

```shell
docker pull apache/skywalking-oap-server:8.3.0-es7
```

#### 启动Skywalking OAP

<span style="color:red;">注：--link后面的第一个参数和elasticsearch容器名一致； -e SW_STORAGE_ES_CLUSTER_NODES：es7也可改为你es服务器部署的Ip地址，即ip:9200</span>

```shell
docker run --name oap --restart always -d --restart=always -e TZ=Asia/Shanghai -p 12800:12800 -p 11800:11800 --link elasticsearch:elasticsearch -e SW_STORAGE=elasticsearch7 -e SW_STORAGE_ES_CLUSTER_NODES=elasticsearch:9200 apache/skywalking-oap-server:8.3.0-es7
```

## 部署Skywalking UI

#### 拉取镜像

```shell
docker pull apache/skywalking-ui:8.3.0
```

#### 启动Skywalking UI

<span style="color:red;">注：--link后面的第一个参数和skywalking OAP容器名一致；</span>

```shell
docker run -d --name skywalking-ui \
--restart=always \
-e TZ=Asia/Shanghai \
-p 8088:8080 \
--link oap:oap \
-e SW_OAP_ADDRESS=oap:12800 \
apache/skywalking-ui:8.3.0
```
