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

## 配置Nginx


## 常用命令

```powershell
./nginx -s stop		#停止nginx
./nginx	-s quit		#安全退出
./nginx -s reload	#修改了文件之后重新加载该程序文件
ps aux|grep nginx	#查看nginx进程
```