# Sharding-Proxy部署 <!-- {docsify-ignore-all} -->

## 单机版

### 下载

```powershell
# wget https://dlcdn.apache.org/shardingsphere/5.1.2/apache-shardingsphere-5.1.2-shardingsphere-proxy-bin.tar.g
```

### 数据库准备

&nbsp; &nbsp; 准备两个数据库，每个数据库创建`user`表，分表的形式创建，`sql`如下：

```sql
# CREATE DATABASE
CREATE DATABASE user_sharding_0;

CREATE DATABASE user_sharding_1;

# CREATE TABLE
use user_sharding_0;

CREATE TABLE `t_user_0` (
	`id` bigint (20) NOT NULL,
	`user_id` bigint (20) NOT NULL,
	`create_date` datetime DEFAULT NULL,
	PRIMARY KEY (`id`)) ENGINE = InnoDB DEFAULT CHARSET = latin1;

CREATE TABLE `t_user_1` (
	`id` bigint (20) NOT NULL,
	`user_id` bigint (20) NOT NULL,
	`create_date` datetime DEFAULT NULL,
	PRIMARY KEY (`id`)) ENGINE = InnoDB DEFAULT CHARSET = latin1;


use user_sharding_1;

CREATE TABLE `t_user_0` (
	`id` bigint (20) NOT NULL,
	`user_id` bigint (20) NOT NULL,
	`create_date` datetime DEFAULT NULL,
	PRIMARY KEY (`id`)) ENGINE = InnoDB DEFAULT CHARSET = latin1;


CREATE TABLE `t_user_1` (
	`id` bigint (20) NOT NULL,
	`user_id` bigint (20) NOT NULL,
	`create_date` datetime DEFAULT NULL,
	PRIMARY KEY (`id`)) ENGINE = InnoDB DEFAULT CHARSET = latin1;
```

## 配置修改

### 修改server.yml

默认的配置文件中的配置全都是注释的，打开注释，修改配置文件，将`mode.type`修改成`Standalone`，`repository.type`改为`File`，注册的类型也支持`H2`，具体的配置如下：

```yml
mode:
 type: Standalone # 单机模式
 repository:
   type: File
  #  props:
  #    jdbcUrl: jdbc:h2:~/test # 元数据持久化数据库连接 URL
  #    user: sha 
  #    password: admin
 overwrite: true # 是否覆盖已存在的元数据

rules: # 认证信息
 - !AUTHORITY
   users: # 初始化用户
     - root@%:root
     - sharding@:sharding
   provider:
     type: ALL_PERMITTED
 - !TRANSACTION
   defaultType: XA
   providerType: Atomikos
 - !SQL_PARSER
   sqlCommentParseEnabled: true
   sqlStatementCache:
     initialCapacity: 2000
     maximumSize: 65535
   parseTreeCache:
     initialCapacity: 128
     maximumSize: 1024

props: # 公用配置
 max-connections-size-per-query: 1
 kernel-executor-size: 16  # Infinite by default.
 proxy-frontend-flush-threshold: 128  # The default value is 128.
 proxy-opentracing-enabled: false
 proxy-hint-enabled: false
 sql-show: false
 check-table-metadata-enabled: false
 show-process-list-enabled: false
   # Proxy backend query fetch size. A larger value may increase the memory usage of ShardingSphere Proxy.
   # The default value is -1, which means set the minimum value for different JDBC drivers.
 proxy-backend-query-fetch-size: -1
 proxy-frontend-executor-size: 0 # Proxy frontend executor size. The default value is 0, which means let Netty decide.
   # Available options of proxy backend executor suitable: OLAP(default), OLTP. The OLTP option may reduce time cost of writing packets to client, but it may increase the latency of SQL execution
   # and block other clients if client connections are more than `proxy-frontend-executor-size`, especially executing slow SQL.
 proxy-backend-executor-suitable: OLAP
 proxy-frontend-max-connections: 0 # Less than or equal to 0 means no limitation.
 sql-federation-enabled: false
   # Available proxy backend driver type: JDBC (default), ExperimentalVertx
 proxy-backend-driver-type: JDBC
```

### 修改config-开头的配置文件

`config-`开头的配置文件是分库分表的规则的配置，如果只是简单的分库分表的功能只配置`config-sharding.yml`即可，这里我们只配置简单的分片规则，所以只修改`config-sharding.yml`，配置如下：

```yml
databaseName: sharding_db # sharding-proxy代理后伪装的数据库名

## sharding-proxy所代理的数据源配置
dataSources:
 ds_0:
   url: jdbc:mysql://127.0.0.1:3316/user_sharding_0?serverTimezone=UTC&useSSL=false
   username: root
   password: admin123
   connectionTimeoutMilliseconds: 30000
   idleTimeoutMilliseconds: 60000
   maxLifetimeMilliseconds: 1800000
   maxPoolSize: 50
   minPoolSize: 1
 ds_1:
   url: jdbc:mysql://127.0.0.1:3316/user_sharding_1?serverTimezone=UTC&useSSL=false
   username: root
   password: admin123
   connectionTimeoutMilliseconds: 30000
   idleTimeoutMilliseconds: 60000
   maxLifetimeMilliseconds: 1800000
   maxPoolSize: 50
   minPoolSize: 1
# 分库分表的规则配置
rules:
- !SHARDING
  tables:
    t_user:
      actualDataNodes: ds_${0..1}.t_user_${0..1}
      tableStrategy:
        standard:
          shardingColumn: user_id
          shardingAlgorithmName: t_user_inline
      keyGenerateStrategy:
        column: user_id
        keyGeneratorName: snowflake
  bindingTables:
    - t_user
  defaultDatabaseStrategy:
    standard:
      shardingColumn: user_id
      shardingAlgorithmName: database_inline
  defaultTableStrategy:
    none:

  shardingAlgorithms:
    database_inline:
      type: INLINE
      props:
        algorithm-expression: ds_${user_id % 2}
    t_user_inline:
      type: INLINE
      props:
        algorithm-expression: t_user_${user_id % 2}

  keyGenerators:
    snowflake:
      type: SNOWFLAKE
```

### 启动sharding-proxy

```powershell
➜  apache-shardingsphere-5.1.2-shardingsphere-proxy-bin sh bin/start.sh
```

### 测试

通过`mysql`命令登录`sharding-proxy`，示例命令如下：

因为我的mysql是docker安装的，所以这里通过-h指定宿主机的ip

```powershell
root@7890bc259b24:~# mysql -h192.168.58.45 -uroot -proot -P3307
```

可以看到，原来的`user_sharding_0`和`user_sharding_1`被伪装成了`sharding_db`

```powershell
mysql> show databases;
+--------------------+
| schema_name        |
+--------------------+
| sharding_db        |
| mysql              |
| information_schema |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.02 sec)
```

原来两个库中的分表被伪装成`t_user`了

```powershell
mysql> use sharding_db;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+-----------------------+------------+
| Tables_in_sharding_db | Table_type |
+-----------------------+------------+
| t_user                | BASE TABLE |
+-----------------------+------------+
1 row in set (0.01 sec)
```

查询分库分表中的数据：

```powershell
mysql> select * from t_user;
+----+---------+---------------------+
| id | user_id | create_date         |
+----+---------+---------------------+
|  2 |       2 | 2021-01-01 00:00:00 |
|  4 |       4 | 2022-01-01 00:00:00 |
|  6 |       6 | 2022-03-01 00:00:00 |
|  1 |       1 | 2021-01-01 00:00:00 |
|  3 |       3 | 2021-01-01 00:00:00 |
|  5 |       5 | 2022-02-01 00:00:00 |
+----+---------+---------------------+
6 rows in set (0.02 sec)
```