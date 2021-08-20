# MySQL-SQL优化 <!-- {docsify-ignore-all} -->


## 优化sql语句的一般步骤

> show status命令

show status命令能统计SQL的执行频率

语法：
```sql
-- 显示当前session中所有统计参数的值
show status like 'com_%';
```
show status可以指定参数用于统计特定SQL操作的执行频率，如：Com_XXX的方式，XXX可以是select，delete，update，insert等；

show status也可以指定统计存储引擎，如：Innodb_row_read执行select查询返回的行数，Innodb_row_inserted执行insert插入的行数，Innodb_row_updated执行update更新的行数，Innodb_row_deleted执行delete删除的行数。通过这几个参数，可以容易的了解当前数据库的应用是以插入更新为主还是以查询操作为主，以及各种类型的sql执行的大致比例，对于更新操作的计数不论提交还是婚回滚次数都会累加。

对于事务类型的应用，通过Com_commit和Com_rollback可以了解事务提交和回滚的情况，对于回滚操作非常频繁的数据库，可能意味着应用编写存在问题。

以下接个参数可以了解数据库的基本情况：

Connections：试图连接Mysql服务器的次数
Uptime：服务器工作时间
Slow_queries：慢查询次数

> 定位执行效率低的SQL语句

- 通过慢查询日志：--log-slow-queries[=file_name]，启动时，mysqld写一个包含所有执行时间超过long_query_time秒的SQL语句的日志文件
- 使用show processlist命令查看当前mysql在进行的线程，包含线程的状态，是否缩表等

> explain分析低效SQL的执行计划

- select_type：表示SELECT的类型，SIMPLE简单表（不使用表连接或子查询），PRIMARY主查询，UNION（UNION重的第二个或者后面的查询语句），SYBQUERY子查询重的第一个SELECT等。
- table：输出结果集的表
- type：表示MySQL在表中找到所需行的方式，或者叫访问类型分别是ALL->index->range->eq_ref->const,system->NULL，从左至右，性能由最差到最好

    - ALL：全表扫描
    - index：索引全扫描，MySQL遍历整个索引来查询匹配的行
    - range：索引范围扫描，常见的<,<=,>,>=,between等
    - ref：使用非唯一索引或唯一索引的前缀扫描，返回匹配某个单独值的记录行
    - eq_ref：与ref类似，区别就是使用的索引是唯一索引
    - const/system：