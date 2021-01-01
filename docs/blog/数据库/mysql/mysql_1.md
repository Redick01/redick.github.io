# Mysql集群与高可用

## Docker搭建最基本的Mysql主从

### 1.安装Mysql
    docker pull mysql:5.7, 拉取mysql5.7镜像

### 2.编写docker-compose文件（docker-compose.yaml）
    version: '2'
    networks:
    byfn:
    services:
    master:
        image: 'mysql:5.7'
        restart: always
        container_name: mysql-master
        environment:
        MYSQL_USER: test
        MYSQL_PASSWORD: root
        MYSQL_ROOT_PASSWORD: root
        ports:
        - '3316:3306'
        volumes:
        - "./master/my.cnf:/etc/mysql/my.cnf"
        networks:
        - byfn
    slave:
        image: 'mysql:5.7'
        restart: always
        container_name: mysql-slave
        environment:
        MYSQL_USER: test
        MYSQL_PASSWORD: root
        MYSQL_ROOT_PASSWORD: root
        ports:
        - '3326:3306'
        volumes:
        - "./slave/my.cnf:/etc/mysql/my.cnf"
        networks:
        - byfn
  
### 3.在docker-compose.yaml当前目录下，建立两个文件夹，master和slave，里面分别写入文件my.cnf
    master/my.cnf
    [mysqld]
    server-id=1
    max_allowed_packet=200M
    wait_timeout=1814400
    log-bin=/var/lib/mysql/mysql-bin

    slave/my.cnf
    [mysqld]
    server-id=2
    wait_timeout=1814400
    max_allowed_packet = 200M

### 4.然后在当前docker-compose.yaml 文件目录下执行
    docker-compose -f docker-compse.yaml up -d

### 5.启动之后进入master容器
    docker exec -it mysql-master /bin/bash
    mysql -uroot -proot
    进入 mysql 终端之后
    mysql> create user 'repl'@'%' identified by 'root';
    mysql> GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'%'; 
    mysql> flush privileges;
    mysql> show master status;
    +------------------+----------+--------------+------------------+-------------------+
    | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
    +------------------+----------+--------------+------------------+-------------------+
    | mysql-bin.000010 |      154 |              |                  |                   |
    +------------------+----------+--------------+------------------+-------------------+
    1 row in set (0.00 sec)

### 6.另起一个终端进入slave容器
    docker exec -it mysql-slave /bin/bash
    mysql -uroot -proot
    进入 mysql 终端之后
    mysql> CHANGE MASTER TO MASTER_HOST='mysql-master', MASTER_PORT=3306,  MASTER_USER='repl', MASTER_PASSWORD='root', MASTER_LOG_FILE='mysql-bin.000010', MASTER_LOG_POS=154;
    mysql> start slave;

    show slave status\G
    *************************** 1. row ***************************
                Slave_IO_State: Waiting for master to send event
                    Master_Host: mysql-master
                    Master_User: repl
                    Master_Port: 3306
                    Connect_Retry: 60
                Master_Log_File: mysql-bin.000004
            Read_Master_Log_Pos: 154
                Relay_Log_File: 295f6f6a4e34-relay-bin.000005
                    Relay_Log_Pos: 367
            Relay_Master_Log_File: mysql-bin.000004
                Slave_IO_Running: Yes
                Slave_SQL_Running: Yes
                Replicate_Do_DB: 
            Replicate_Ignore_DB: 
            Replicate_Do_Table: 
        Replicate_Ignore_Table: 
        Replicate_Wild_Do_Table: 
    Replicate_Wild_Ignore_Table: 
                    Last_Errno: 0
                    Last_Error: 
                    Skip_Counter: 0
            Exec_Master_Log_Pos: 154
                Relay_Log_Space: 747
                Until_Condition: None
                Until_Log_File: 
                    Until_Log_Pos: 0
            Master_SSL_Allowed: No
            Master_SSL_CA_File: 
            Master_SSL_CA_Path: 
                Master_SSL_Cert: 
                Master_SSL_Cipher: 
                Master_SSL_Key: 
            Seconds_Behind_Master: 0
    Master_SSL_Verify_Server_Cert: No
                    Last_IO_Errno: 0
                    Last_IO_Error: 
                Last_SQL_Errno: 0
                Last_SQL_Error: 
    Replicate_Ignore_Server_Ids: 
                Master_Server_Id: 1
                    Master_UUID: 4d83248d-34a8-11eb-beb9-0242ac120003
                Master_Info_File: /var/lib/mysql/master.info
                        SQL_Delay: 0
            SQL_Remaining_Delay: NULL
        Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
            Master_Retry_Count: 86400
                    Master_Bind: 
        Last_IO_Error_Timestamp: 
        Last_SQL_Error_Timestamp: 
                Master_SSL_Crl: 
            Master_SSL_Crlpath: 
            Retrieved_Gtid_Set: 
                Executed_Gtid_Set: 
                    Auto_Position: 0
            Replicate_Rewrite_DB: 
                    Channel_Name: 
            Master_TLS_Version: 
    1 row in set (0.01 sec)
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
同步成功

### 7.mysql客户端连接两个mysql，在master上创建数据库、表，添加数据，查看slave
    master
    mysql> select * from user;
    +----+------+------+------+
    | id | name | sex  | age  |
    +----+------+------+------+
    |  1 | hhh  | 1    |   11 |
    |  2 | qqq  | 2    |   22 |
    |  3 | ssss | 2    |   22 |
    +----+------+------+------+
    3 rows in set (0.00 sec)

    slave
    mysql> select * from user;
    +----+------+------+------+
    | id | name | sex  | age  |
    +----+------+------+------+
    |  2 | qqq  | 2    |   22 |
    |  3 | ssss | 2    |   22 |
    +----+------+------+------+
    2 rows in set (0.01 sec)

    id=1是配置主从同步前的数据
    id=2和3的是配置主从后的数据

## 高可用MGR

## 高可用MHA


## 高可用Cluster