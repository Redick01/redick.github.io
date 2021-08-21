# MySQL-SQL优化分析 <!-- {docsify-ignore-all} -->


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
    - const/system：单表中最多有一个匹配行，查询起来非常迅速
    - NULL：MYSQL不用访问表或者索引就能直接得到结果，例如：select 1 from dual where 1 = 1;

- possible_keys：表示查询时可能走的索引
- key：查询时实际使用的索引
- key_len：使用到索引字段的长度
- rows：扫描行数
- Extra：执行情况的说明和描述，包含不适合在其他列中显示但是对执行计划非常重要的额外信息。

> show profiles和show profile分析SQL

MySQL从5.0.37版本开始对show profiles和show profile支持，通过`select @@have_profiling`可以查看当前MySQL是否支持profile。默认profile是关闭的可以通过`set profiling=1;`开启Session级别的profiling，通过profile，能够更清楚地了解SQL执行的过程。show profile的使用方式如下

1. 执行业务sql，如：select count(*) from payment;
2. show profiles; 该语句详细的列出sql的执行过程包含queryId，耗时，SQL过程
3. show profile for query #ID; 该语句详细列出show profiles;中某一个queryId中线程的每个状态和耗时
4. 在获取到最消耗时间的线程状态后，MySQL进一步支持选择all，cpu，block io，context swutch，page faults等明细类型来查看MySQL在使用什么资源上耗费了过高的时间，例如选择cpu：show profile cpu for query #ID

可以看到耗时主要发生在Sending data且耗时主要在CPU上，相同的SQL在MyISAM引擎下不会产生这么大耗时，因为MyISAM引擎缓存了表的行数，在统计的时候直接返回而InnoDB则需要扫描表。
```shell
mysql> select count(*) from payment;
+----------+
| count(*) |
+----------+
|    16049 |
+----------+
1 row in set (0.00 sec)

mysql> show profiles;
+----------+------------+------------------------------+
| Query_ID | Duration   | Query                        |
+----------+------------+------------------------------+
|        1 | 0.00572250 | select count(*) from payment |
|        2 | 0.00440850 | select count(*) from payment |
|        3 | 0.00435275 | select count(*) from payment |
+----------+------------+------------------------------+
3 rows in set, 1 warning (0.00 sec)

mysql> show profile for query 3;
+----------------------+----------+
| Status               | Duration |
+----------------------+----------+
| starting             | 0.000144 |
| checking permissions | 0.000012 |
| Opening tables       | 0.000016 |
| init                 | 0.000013 |
| System lock          | 0.000008 |
| optimizing           | 0.000005 |
| statistics           | 0.000017 |
| preparing            | 0.000011 |
| executing            | 0.000002 |
| Sending data         | 0.004019 |
| end                  | 0.000013 |
| query end            | 0.000010 |
| closing tables       | 0.000006 |
| freeing items        | 0.000053 |
| cleaning up          | 0.000026 |
+----------------------+----------+
15 rows in set, 1 warning (0.01 sec)

mysql> show profile cpu for query 3;
+----------------------+----------+----------+------------+
| Status               | Duration | CPU_user | CPU_system |
+----------------------+----------+----------+------------+
| starting             | 0.000144 | 0.000054 |   0.000029 |
| checking permissions | 0.000012 | 0.000006 |   0.000004 |
| Opening tables       | 0.000016 | 0.000011 |   0.000005 |
| init                 | 0.000013 | 0.000009 |   0.000005 |
| System lock          | 0.000008 | 0.000004 |   0.000003 |
| optimizing           | 0.000005 | 0.000003 |   0.000001 |
| statistics           | 0.000017 | 0.000011 |   0.000006 |
| preparing            | 0.000011 | 0.000007 |   0.000004 |
| executing            | 0.000002 | 0.000001 |   0.000000 |
| Sending data         | 0.004019 | 0.004022 |   0.000000 |
| end                  | 0.000013 | 0.000010 |   0.000000 |
| query end            | 0.000010 | 0.000010 |   0.000000 |
| closing tables       | 0.000006 | 0.000006 |   0.000000 |
| freeing items        | 0.000053 | 0.000053 |   0.000000 |
| cleaning up          | 0.000026 | 0.000025 |   0.000000 |
+----------------------+----------+----------+------------+
15 rows in set, 1 warning (0.01 sec)
```

> 通过trace分析优化器如何选择执行计划

MySQL 5.6提供了对SQL的跟踪trace，通过trace文件能够进一步了解为什么优化器选择A执行计划而不选择B执行计划

使用方式：首先打开trace，设置json格式，设置trace最大能够使用的内存大小，避免解析过程中因为默认内存过小而不能完整的显示。

```shell
mysql> SET OPTIMIZER_TRACE="enabled=on",END_MARKERS_IN_JSON=on;
Query OK, 0 rows affected (0.00 sec)

mysql> SET OPTIMIZER_TRACE_MAX_MEM_SIZE=1000000;
Query OK, 0 rows affected (0.00 sec)
```

业务sql如下：
```shell
mysql> select rental_id from rental where 1=1 and rental_date >= '2005-05-25 04:00:00' and rental_date <= '2005-05-25 05:00:00' and inventory_id=4466;
+-----------+
| rental_id |
+-----------+
|        39 |
+-----------+
1 row in set (0.00 sec)
```

最后，检查INFORMATION_SCHEMA.OPTIMIZER_TRACE就可以知道MySQL是如何执行SQL的：

```shell
mysql> SELECT * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE;
```

## 总结

经过上面的步骤基本上可以确认问题出现的原因了，此时就可以采取相应的措施进行优化以提高执行效率。