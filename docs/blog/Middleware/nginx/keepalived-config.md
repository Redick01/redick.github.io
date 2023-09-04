## Nginx一主一从配置Keepalive <!-- {docsify-ignore-all} -->





### 机器规划

| 主机   | vip            | 内网ip         |
| ------ | -------------- | -------------- |
| Nginx1 | 192.168.xxx.xx | 192.168.xxx.ab |
| Nginx2 | 192.168.xxx.xx | 192.168.xxx.cd |



### 监控Nginx进程

1. Nginx监控脚本，监控nginx，如果nginx停了，那么杀掉keepalive进程

```bash
#!/bin/bash
A=`ps -C nginx --no-header | wc -l`
if [ $A -eq 0 ];then
    # 启动失败，将keepalived服务杀死。将vip漂移到其它备份节点
    systemctl stop keepalived
fi
```



### 安装配置Keepalive

1. 安装命令

```shell
yum -y install keepalived
```

2. 修改Keepalive配置

​    输入命令 vi /etc/keepalived/keepalived.conf，修改 keepalived 配置文件，需要修改几处内容，分别是设置vip（本例中为192.168.246.48）、修改云主机的优先级（主节点优先级高，建议主节点设置为100，备节点设置为50）、屏蔽 vrrp_strict、添加定时任务检查nginx进程脚本。

- 主节点配置

```shell
! Configuration File for keepalived

global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
   #vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

global_defs {
   router_id Nginx_01
}
vrrp_script check_nginx {
        script "/etc/keepalived/check_nginx.sh"
        # 每2秒检测一次nginx的运行状态
        interval 2
        # 失败一次，将自己的优先级-5
        weight -5
}
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
     # 虚拟IP
     192.168.xxx.xx
    }
    # 检查脚本
    track_script {
        check_nginx
    }
}

virtual_server 192.168.200.100 443 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.201.100 443 {
        weight 1
        SSL_GET {
            url {
              path /
              digest ff20ad2481f97b1754ef3e12ecd3a9cc
            }
            url {
              path /mrtg/
              digest 9b3a0c85a887a256d6939da88aabd8cd
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}

virtual_server 10.10.10.2 1358 {
    delay_loop 6
    lb_algo rr 
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    sorry_server 192.168.200.200 1358

    real_server 192.168.200.2 1358 {
        weight 1
        HTTP_GET {
            url { 
              path /testurl/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url { 
              path /testurl2/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url { 
              path /testurl3/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }

    real_server 192.168.200.3 1358 {
        weight 1
        HTTP_GET {
            url { 
              path /testurl/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334c
            }
            url { 
              path /testurl2/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334c
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}

virtual_server 10.10.10.3 1358 {
    delay_loop 3
    lb_algo rr 
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.200.4 1358 {
        weight 1
        HTTP_GET {
            url { 
              path /testurl/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url { 
              path /testurl2/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url { 
              path /testurl3/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }

    real_server 192.168.200.5 1358 {
        weight 1
        HTTP_GET {
            url { 
              path /testurl/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url { 
              path /testurl2/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url { 
              path /testurl3/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}
```

- 从节点配置

```shell
! Configuration File for keepalived

global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 192.168.200.1
   smtp_connect_timeout 30
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
   #vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

global_defs {
   router_id Nginx_02
}
vrrp_script check_nginx {
        # nginx进程检查脚本
        script "/etc/keepalived/check_nginx.sh"
        # 2s执行一次
        interval 2
        # 触发一次优先级降低5
        weight -5
}
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 50
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
     # 虚拟IP
     192.168.xxx.xx
    }
    # 检查脚本
    track_script {
        check_nginx
    }
}

virtual_server 192.168.200.100 443 {
    delay_loop 6
    lb_algo rr
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.201.100 443 {
        weight 1
        SSL_GET {
            url {
              path /
              digest ff20ad2481f97b1754ef3e12ecd3a9cc
            }
            url {
              path /mrtg/
              digest 9b3a0c85a887a256d6939da88aabd8cd
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}

virtual_server 10.10.10.2 1358 {
    delay_loop 6
    lb_algo rr 
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    sorry_server 192.168.200.200 1358

    real_server 192.168.200.2 1358 {
        weight 1
        HTTP_GET {
            url { 
              path /testurl/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url { 
              path /testurl2/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url { 
              path /testurl3/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }

    real_server 192.168.200.3 1358 {
        weight 1
        HTTP_GET {
            url { 
              path /testurl/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334c
            }
            url { 
              path /testurl2/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334c
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}

virtual_server 10.10.10.3 1358 {
    delay_loop 3
    lb_algo rr 
    lb_kind NAT
    persistence_timeout 50
    protocol TCP

    real_server 192.168.200.4 1358 {
        weight 1
        HTTP_GET {
            url { 
              path /testurl/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url { 
              path /testurl2/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url { 
              path /testurl3/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }

    real_server 192.168.200.5 1358 {
        weight 1
        HTTP_GET {
            url { 
              path /testurl/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url { 
              path /testurl2/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            url { 
              path /testurl3/test.jsp
              digest 640205b7b0fc66c1ea91c463fac6334d
            }
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
        }
    }
}
```

3. 主从启动 keepalived 服务，输入命令 systemctl start keepalived，注意要先启动Nginx。

### 验证高可用性

通过安装httpd服务验证可用性

1. 通过配置一个简单的HTTP服务，验证虚拟机高可用有效

2. VNC 登录主机1，输入命令安装HTTP服务：yum -y install httpd

3. 配置主节点简单页面：echo "Master NODE" > /var/www/html/index.html

4. 授予权限：chmod 644 /var/www/html/index.html

5. 启动主节点http服务：systemctl start httpd

6. VNC 登录主机2，输入命令安装HTTP服务：yum -y install httpd

7. 配置备节点简单页面：echo "Backup NODE" > /var/www/html/index.html

8. 授予权限：chmod 644 /var/www/html/index.html

9. 启动备节点 http 服务：systemctl start httpd

10. 通过备节点访问HA VIP（curl 192.168.xxx.xx），可以看到目前HAVIP是落在主节点上的

![image-20230830132535501](/Users/penghuiliu/geek_learn/redick.github.io/docs/_media/image/中间件/nginx/master.png)

11. 在主节点上停止Nginx进程

12. 继续通过备节点访问HA VIP（curl 192.168.xxx.xx），HA VIP 已经漂移到备节点上，证明高可用配置成功

![image-20230830132943944](/Users/penghuiliu/geek_learn/redick.github.io/docs/_media/image/中间件/nginx/salve.png)

