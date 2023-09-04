# 单机版Nginx安装及配置-某地铁APP <!-- {docsify-ignore-all} -->

## 官网下载Nginx

http://nginx.org/en/download.html

注：下载稳定版，即Stateable Version的，选择对应操作系统，我这里是Linux，就选择了 nginx-1.24.0

## 解压安装

```powershell
tar -xvf nginx-1.24.0.tar
```

- 安装C++库和openssl等

```powershell
yum -y install gcc-c++
yum -y install pcre pcre-devel
yum -y install zlib zlib-devel
yum -y install openssl openssl-devel
```

- 安装

顺序执行下列命令

```powershell
./configure
make
make install
```


## 常用命令

```powershell
./nginx -s stop		#停止nginx
./nginx	-s quit		#安全退出
./nginx -s reload	#修改了文件之后重新加载该程序文件
ps aux|grep nginx	#查看nginx进程
sbin/nginx -c /conf/nginx.vonf #指定配置文件启动
```



## 配置负载均衡

### 七层负载均衡

#### nginx的负载均衡语法

```
http {
   upstream [你的负载均衡机制名称，随便设置一个就好] {
	 server [ip地址]:[端口值];
	 server [ip地址]:[端口值];
	 server [ip地址]:[端口值];
	 server [ip地址]:[端口值];
 }
 server {
	 listen [nginx监听端口];
	 server_name [head中的host对应的值]
	 location / {
	   proxy_pass http:// [你的负载均衡机制名称，对应上面upstream的值];
         }
    }
}
```

#### nginx的负载均衡策略

1. 轮询（Round Robin默认）

​     轮询是最常见的一种负载均衡策略。Nginx默认使用轮询策略，将请求按照顺序分配到每个服务器，当请求到达最后一个服务器后，再从第一个服务器继续轮询。，如果后端服务器挂了，则自动剔除。

```conf
upstream backend {
   server [ip地址]:[端口值];
	 server [ip地址]:[端口值];
	 server [ip地址]:[端口值];
	 server [ip地址]:[端口值];
}

server {
   listen 80;
   server_name example.com;
   location / {
       proxy_pass http://backend;
   }
}
```



2. 权重(Weighted Load Balancing)

​    指定轮询的频率，weight和访问率成正比，用于后端服务器性能不均匀的情况

```shell
upstream backend {
    server [ip地址]:[端口值] weight=3;
	  server [ip地址]:[端口值] weight=2;
	  server [ip地址]:[端口值] weight=1;
}

server {
    listen 80;
    server_name example.com;
    location / {
        proxy_pass http://backend;
    }
}
```



3. IP Hash

​    IP Hash是一种漂亮的负载均衡策略，具有Session保持的优点。算法的基本思路是通过对客户端的IP地址取Hash值，将此Hash值与服务器列表中的IP地址的Hash值进行比较，找到具有匹配Hash值的服务器。这样相同IP的请求总是被转发到同一台后端服务器处理，保证Session信息在同一台服务器上处理。

```conf
upstream backend {
    ip_hash; #使用IP hash策略
    server [ip地址]:[端口值];
	  server [ip地址]:[端口值];
	  server [ip地址]:[端口值];
	  server [ip地址]:[端口值];
}

server {
    listen 80;
    server_name example.com;
    location / {
        proxy_pass http://backend;
    }
}
```



4. 最少连接（Least Connections）

​    nginx会尽量不让负载繁忙的应用服务器上负载过多的请求，相反的，会把新的请求发送到比较不繁忙的服务器。

```conf
upstream backend {
    least_conn; #使用Least Connections策略
    server [ip地址]:[端口值];
	  server [ip地址]:[端口值];
	  server [ip地址]:[端口值];
	  server [ip地址]:[端口值];
}

server {
    listen 80;
    server_name example.com;
    location / {
        proxy_pass http://backend;
    }
}
```

5. 随机(Random)

​    Random会将请求随机发送到后端服务器上，这种策略比较简单，但是不保证对后端服务器的负载均衡性。

```shell
upstream backend {
    random; #使用Random策略
    server [ip地址]:[端口值];
	  server [ip地址]:[端口值];
	  server [ip地址]:[端口值];
	  server [ip地址]:[端口值];
}

server {
    listen 80;
    server_name example.com;
    location / {
        proxy_pass http://backend;
    }
}
```

6. URL Hash

​    URL Hash会根据请求的URL的Hash值来将请求发送到后端服务器。相同URL的请求总是被转发到同一台后端服务器处理，从而保证Session信息在同一台服务器上处理。

```conf
upstream backend {
    hash $request_uri; #使用URL Hash策略
    server [ip地址]:[端口值];
	  server [ip地址]:[端口值];
	  server [ip地址]:[端口值];
	  server [ip地址]:[端口值];
}

server {
    listen 80;
    server_name example.com;
    location / {
        proxy_pass http://backend;
    }
}
```



#### 故障下线和备份服务设置

1.down

​    假如有一台主机是出了故障，或者下线了，要暂时移出，那可以把它标为down，表示请求是会略过这台主机的。

```
upstream downServer {
        server www.address1.com; # 或者ip+端口 ， 不需要加入http/https前缀
        server www.address2.com down;
}
```

2.backup

​    backup是指备份的机器，相对于备份的机器来说，其他的机器就相当于主要服务器，只要当主要服务器不可用的时候，才会用到备用服务器。

```
upstream backupServer {
        server www.address1.com; # 或者ip+端口 ， 不需要加入http/https前缀
        server www.address2.com backup;
}
```

3.max_fails和fail_timeout

​    默认情况下，max_fails的值为1，表示的是请求失败的次数，请求1次失败就换到下台主机。另外还有一个参数是fail_timeout，表示的是请求失败的超时时间，在设定的时间内没有成功，那作为失败处理。

```
upstream backupServer {
        server www.address1.com max_fails=2; # 或者ip+端口 ， 不需要加入http/https前缀
        server www.address2.com backup;
}
```



#### proxy_pass参数

- proxy_set_header：设置反向代理向后端发送的http请求头信息，如添加host主机头部字段，让后端服务器能够获取到真实客户端的IP信息等
- client_body_buffer_size：指定客户端请求主体缓冲区大小
- proxy_connect_timeout：反向代理和后端节点连接的超时时间，也是建立握手后等待响应的时间
- proxy_send_timeout：表示代理后端服务器的数据回传时间，在规定时间内后端若数据未传完，nginx会断开连接
- proxy_read_timeout：设置Nginx从代理服务器获取数据的超时时间
- proxy_buffer：设置缓冲区的数量大小



