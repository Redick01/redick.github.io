

## Prometheus配置文件


## Grafana修改密码

找到 "grafana.db" 文件

执行sqlite3 grafana.db 

```shell
# 查看数据库中的表
sqlite> .tables
alert                     dashboard_tag             server_lock             
alert_notification        dashboard_version         session                 
alert_notification_state  data_source               star                    
alert_rule_tag            login_attempt             tag                     
annotation                migration_log             team                    
annotation_tag            org                       team_member             
api_key                   org_user                  temp_user               
cache_data                playlist                  test_data               
dashboard                 playlist_item             user                    
dashboard_acl             plugin_setting            user_auth               
dashboard_provisioning    preferences               user_auth_token         
dashboard_snapshot        quota

# 查看user表中的内容                   
sqlite> select * from user;
1|0|admin|admin@localhost||a00f4ed0f73a671bb3e0f61ea64f3ba31278e614b7873f82e2d2f89fc3b39f0b776a8d10a6a4667b72de3c08f50f33711487|VGdjDikpzv|AmtsbvblVi||1|1|0||2019-12-10 05:45:17|2020-02-26 01:59:34|0|2020-02-26 01:59:34|0

# 重置admin的密码为admin
sqlite> update user set password = '59acf18b94d7eb0694c61e60ce44c110c7a683ac6a8f09580d626f90f4a242000746579358d77dd9e570e83fa24faa88a8a6', salt = 'F3FAxVm33R' where login = 'admin';

# 退出数据库
sqlite> .exit
```