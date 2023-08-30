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

## 配置Nginx转发+负载均衡



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

 1.轮询（默认）

​     每个请求按照请求时间顺序分配到不同的后端服务器，如果后端服务器挂了，则自动剔除

 2.权重

​    指定轮询的频率，weight和访问率成正比，用于后端服务器性能不均匀的情况

 http {
	upstream ipHashLoadBalanceServer {
        ip_hash;
        server www.address1.com weight=3;// 或者ip+端口 ， 不需要加入http/https前缀
        server www.address2.com; // default weight=1
        server www.address3.com;
    }
    server {
        listen 80;
        location / {
            proxy_pass http://loadBalanceServer;
        }
    }
 }
3.ip_hash

​    客户端ip地址被用作hash key来判断客户端请求应该发送到哪个服务器，这种方法保证了来自相同客户端的请求总是发送到相同服务器（如果服务器可用的话

upstream myapp1 {
    ip_hash;
    server srv1.example.com;
    server srv2.example.com;
    server srv3.example.com;
}
4.最少连接

​    nginx会尽量不让负载繁忙的应用服务器上负载过多的请求，相反的，会把新的请求发送到比较不繁忙的服务器。

upstream myapp1 {
        least_conn;
        server srv1.example.com;
        server srv2.example.com;
        server srv3.example.com;
}

#### 故障下线和备份服务设置

1.down

​    假如有一台主机是出了故障，或者下线了，要暂时移出，那可以把它标为down，表示请求是会略过这台主机的。

upstream downServer {
        server www.address1.com; // 或者ip+端口 ， 不需要加入http/https前缀
        server www.address2.com down;
}
2.backup

​    backup是指备份的机器，相对于备份的机器来说，其他的机器就相当于主要服务器，只要当主要服务器不可用的时候，才会用到备用服务器。

upstream backupServer {
        server www.address1.com; // 或者ip+端口 ， 不需要加入http/https前缀
        server www.address2.com backup;
}
3.max_fails和fail_timeout

​    默认情况下，max_fails的值为1，表示的是请求失败的次数，请求1次失败就换到下台主机。另外还有一个参数是fail_timeout，表示的是请求失败的超时时间，在设定的时间内没有成功，那作为失败处理。

upstream backupServer {
        server www.address1.com max_fails=2; // 或者ip+端口 ， 不需要加入http/https前缀
        server www.address2.com backup;
}

#### proxy_pass参数

- proxy_set_header：设置反向代理向后端发送的http请求头信息，如添加host主机头部字段，让后端服务器能够获取到真实客户端的IP信息等
- client_body_buffer_size：指定客户端请求主体缓冲区大小
- proxy_connect_timeout：反向代理和后端节点连接的超时时间，也是建立握手后等待响应的时间
- proxy_send_timeout：表示代理后端服务器的数据回传时间，在规定时间内后端若数据未传完，nginx会断开连接
- proxy_read_timeout：设置Nginx从代理服务器获取数据的超时时间
- proxy_buffer：设置缓冲区的数量大小



