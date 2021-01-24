# Docker安装nacos

- **拉取镜像**

```
docker pull nacos/nacos-server
```

- **启动服务**

```
docker run --env MODE=standalone --name nacos -d -p 8848:8848 nacos/nacos-server

# 进入控制台
docker exec -it nacos bash
```

- **Web 管理地址：**

```
http://127.0.0.1:8848/nacos/
默认端口号是：8848
默认账号密码：nacos/nacos
```