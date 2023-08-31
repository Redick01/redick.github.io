# Nginx配置ssl证书 <!-- {docsify-ignore-all} -->



开始配置:

1.进入到nginx目录,查看有没有http_ssl_module模块

./nginx -V
2.如果没有,找到源码,输入以下命令进行安装(如果有,跳转到第6步)

#prefix后面的路径是你安装nginx的路径
./configure --prefix=/usr/local/nginx --with-http_ssl_module
3.configure执行完成后,输入make,注意:千万不要make install,这样会覆盖原有的配置

4.make完成后,停止nginx服务,进入objs目录,将nginx启动程序,拷贝到安装目录下,替换原有的启动程序

5.启动nginx,输入./nginx -V,查看是否安装成功

6.新建一个目录cert,把申请下来的证书上传上去

7.打开配置文件nginx.conf,加入以下配置

http{

    server{
        listen 443 ssl;
        #对应你的域名
        server_name test.com;
        ssl_certificate /usr/local/nginx/cert/ssl.crt;
        ssl_certificate_key /usr/local/nginx/cert/ssl.key;
        ssl_session_timeout 5m;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
        ssl_prefer_server_ciphers on;
        #如果是静态文件,直接指向目录,如果是动态应用,用proxy_pass转发一下
        location / {
                root /usr/local/service/ROOT;
                index index.html;
        }
    }
    #监听80端口,并重定向到443
    server{
        listen 80;
        server_name test.com;
        rewrite ^/(.*)$ https://test.com:443/$1 permanent;
    }
}

8.重启nginx

./nginx -s reload
