# Istio使用Envoy转发Http请求错误码426 Upgrade Required <!-- {docsify-ignore-all} -->


## 背景

&nbsp; &nbsp; 此前将公司的几个服务进行了服务网格的技术改造，其中一个应用是为H5提供Http接口，在改造之前，H5通过域名调用后台接口，请求经过nginx进行转发，转发的目标是阿里云SLB（阿里云负载均衡产品）的ip端口，服务网格改造后，Nginx的转发目标变成了k8s的ingress网关的ip端口，接着问题就出现了，接口调不通，Http返回426 Upgrade Required。


## 解决办法

### 426 Upgrade Required解释

&nbsp; &nbsp; 该错误码意思是客户端需要升级协议版本


### 导致原因和解决办法

> 原因

&nbsp; &nbsp; Istio使用Envoy作为数据面转发HTTP请求，而Envoy默认要求使用HTTP/1.1或HTTP/2，当客户端使用HTTP/1.0时就会返回426 Upgrade Required，如果用nginx进行proxy_pass反向代理，默认会用 HTTP/1.0

> 解决：nginx显示指定 proxy_http_version为1.1

&nbsp; &nbsp; 修改nginx的配置文件nginx.conf，显示指定代理proxy_http_version为1.1，代码如下：

```json
server {
    ...

    location /http/path/ {
        proxy_http_version 1.1;
        proxy_pass http://ip:port;
    }
}
```

## 参考

[Istio常见问题状态码: 426 Upgrade Required](https://imroc.cc/istio/faq/426-status-code/)