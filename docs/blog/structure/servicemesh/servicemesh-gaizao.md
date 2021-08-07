# 公司服务Service Mesh改造

- 产品选型

## 产品选型

> 阿里云k8s集群，阿里云服务网格


> 自建Harbor镜像仓库


## 镜像推送、拉取问题

> 开发机器docker解决docker login失败问题

> 阿里云k8s集群拉取镜像失败问题解决
  
  harbor仓库权限设置公开哦

> Dockerfile构建，推送

docker build -t miss-interface-unionpaycard:4.5.2.2 .

docker tag miss-interface-unionpaycard:4.5.2.2 172.17.10.157/miss-interface-unionpaycard/2021.08.07:4.5.2.2

docker push 172.17.10.157/miss-interface-unionpaycard/2021.08.07:4.5.2.2

> 进入容器

kubectl exec -it miss-interface-unionpaycard-4.5.2.2-84f8c4b5f6-nhnp2 -- /bin/bash