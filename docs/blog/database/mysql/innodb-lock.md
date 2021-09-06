# MySQL InnoDB下的锁问题 <!-- {docsify-ignore-all} -->

## 背景知识

&nbsp; &nbsp; InnoDB相比较MyISAM一是支持事务，二是支持了行级锁，提到InnoDB锁问题就不得不提到事务，所以在这之前先了解下事务的一些知识

> 事务及其ACID属性

&nbsp; &nbsp; 事务是由一组SQL语句组成的逻辑处理单元，事务具有以下4个属性，通常简称为事务的ACID属性。

- 原子性（Atomicity）：事务开始后所有操作，要么全部做完，要么全部不做，不可能停滞在中间环节。事务执行过程中出错，会回滚到事务开始前的状态，所有的操作就像没有发生一样。也就是说事务是一个不可分割的整体，就像化学中学过的原子，是物质构成的基本单位。

- 一致性（Consistency）：事务开始前和结束后，数据库的完整性约束没有被破坏 。比如A向B转账，不可能A扣了钱，B却没收到。

- 隔离性（Isolation）：同一时间，只允许一个事务请求同一数据，不同的事务之间彼此没有任何干扰。比如A正在从一张银行卡中取钱，在A取钱的过程结束前，B不能向这张卡转账。

- 持久性（Durability）：事务完成后，事务对数据库的所有更新将被保存到数据库，不能回滚。

> 并发事务处理带来的问题

&nbsp; &nbsp; 相对于串行处理来说，并发事务处理能大大增加数据库资源的利用率，提高数据库系统的事务吞吐量，从而可以支持更多的用户。但是并发事务处理也会带来以下问题。

- 脏读：事务A读取了事务B更新的数据，然后B回滚操作，那么A读取到的数据是脏数据
- 不可重复读：事务 A 多次读取同一数据，事务 B 在事务A多次读取的过程中，对数据作了更新并提交，导致事务A多次读取同一数据时，结果不一致。
- 幻读：系统管理员A将数据库中所有学生的成绩从具体分数改为ABCDE等级，但是系统管理员B就在这个时候插入了一条具体分数的记录，当系统管理员A改结束后发现还有一条记录没有改过来，就好像发生了幻觉一样，这就叫幻读。

小结：不可重复读的和幻读很容易混淆，不可重复读侧重于修改，幻读侧重于新增或删除。解决不可重复读的问题只需锁住满足条件的行，解决幻读需要锁表

> 事务隔离级别

&nbsp; &nbsp; MySQL数据库实现事务隔离的方式，基本上可分为两种。

- 一种是在读取数据前，对其加锁，阻止其他事务对数据进行修改
- 不加锁，通过一定机制生成一个数据请求时间点的一致性数据快照，并用这个快照来提供一定级别（语句级或事务级）的一致性读取。从用户的角度来看，好像是数据库可以提供同意数据的多个版本，因此，这种技术叫做`数据多版本并发控制`简称MVCC。

四种事务隔离级别：

- READ UNCOMMITTED（读未提交）：事务A和B操作同一数据，事务A能够读到事务B未提交的数据，会产生幻读，不可重复读，脏读
- READ COMMITTED（读已提交）：事务A和B操作同一数据，事务A能够读到事务B更新的数据，会产生幻读和不可重复度
- REPEATABLE READ（可重复读）：事务A和事务B操作同一数据，事务A不能读到事务B已经插入的数据，会产生幻读
- SERIALIZABLE（串行化）：所有事务都必须保证串行执行，不会产生脏读，幻读，不可重复度


## 获取InnoDB行锁争用情况

可以通过检查InnoDB_row_lock状态变量来分析系统伤的行锁的争夺情况：

```sql
mysql> show status like 'innodb_row_lock%';
+-------------------------------+-------+
| Variable_name                 | Value |
+-------------------------------+-------+
| Innodb_row_lock_current_waits | 0     |
| Innodb_row_lock_time          | 0     |
| Innodb_row_lock_time_avg      | 0     |
| Innodb_row_lock_time_max      | 0     |
| Innodb_row_lock_waits         | 0     |
+-------------------------------+-------+
5 rows in set (0.04 sec)
```

如果锁争用比较严重，Innodb_row_lock_waits和Innodb_row_lock_time_avg的值比较高，可以通过查询information_schema数据库中相关的表来查看锁情况，或者通过设置InnodDB Monitors来进一步观察发生锁冲突的表、数据行等，并分析其原因。

（1）通过查询information_schema数据库中的innodb_locks表了解锁的等待情况：

```sql
mysql> use information_schema;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> select * from innodb_locks;
Empty set, 1 warning (0.01 sec)

mysql>
```

（2）通过设置InnoDB Monitors观察锁冲突情况：

```sql
mysql> create table innodb_monitor(a INT) ENGINE=INNODB;
Query OK, 0 rows affected (0.05 sec)
```

然后通过下面语句来进行查看：

```sql
mysql> show engine innodb status;
| Type   | Name | Status                                                                                                                                                                                                                                                                                           
| InnoDB |      |
...
------------
TRANSACTIONS
------------
Trx id counter 6076
Purge done for trx's n:o < 6071 undo n:o < 0 state: running but idle
History list length 0
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 421657844005624, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421657844004704, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 421657844006544, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
--------
FILE I/O
--------
...
--------------
ROW OPERATIONS
--------------
0 queries inside InnoDB, 0 queries in queue
0 read views open inside InnoDB
Process ID=1, Main thread ID=140182422546176, state: sleeping
Number of rows inserted 55250, updated 1240, deleted 376, read 22512
0.00 inserts/s, 0.00 updates/s, 0.00 deletes/s, 0.00 reads/s
----------------------------
END OF INNODB MONITOR OUTPUT
============================
1 row in set (0.01 sec)
```

监视器可以通过下列语句来停止：

```sql
mysql> drop table innodb_monitor;
Query OK, 0 rows affected (0.02 sec)
```

&nbsp; &nbsp; 设置监视器后，在show innodb status的显示内容中，会有详细的当前锁等待的信息，包括表名、锁类型、锁定记录的情况等，便于进一步分析和定位问题。

## InnoDB的行锁模式及加锁方法

&nbsp; &nbsp; InnoDB实现了两种类型的行锁

- 共享所（S）：允许一个事务去读一行，阻止其他事务获得相同数据集的排他锁。
- 排他锁（X）：允许获得排他锁的事务更新数据，阻止其他事务取得相同数据集的共享读锁和排他写锁。

&nbsp; &nbsp; 另外，为了允许行锁和表锁共存，实现多粒度锁机制，InnoDB还有两种内部使用的意向锁。

- 意向共享锁（IS）：事务打算给数据行加共享锁，事务在给一个数据行加共享锁前必须先获得该表的IS锁。
- 意向排他锁（IX）：事务打算给数据行加行排他锁，事务在给一个数据行排他锁前必须先获得该表的IX锁。

InnoDB行锁模式兼容性列表：

|  | X | IX | S | IS |
| --- | --- | --- | --- | --- |
| X | 冲突 | 冲突 | 冲突 | 冲突 |
| IX | 冲突 | 兼容 | 冲突 | 兼容 |
| S | 冲突 | 冲突 | 兼容 | 兼容 |
| IS | 冲突 | 兼容 | 兼容 | 兼容 |

&nbsp; &nbsp; 如果一个事务请求的锁模式与当前的锁兼容，InnoDB就将请求的锁收于授予该事务，否则该事务就要等待锁的释放。

&nbsp; &nbsp; 意向锁是InnoDB自动加的，对于update，delete和insert语句，InnoDB是会自动给涉及数据集加排他锁（X）；对于普通SELECT语句，InnoDB不会加任何锁；事务可以通过以下语句显示给记录集加共享锁或排他锁。

- 共享锁（S）：select * from table_name where ... lock in share mode;
- 排他锁（X）：select * from table_name where ... for update;

&nbsp; &nbsp; 用lock in share mode获得共享锁，主要用在需要数据依存关系时来确认某行记录是否存在，并确保没有人对这个记录进行update或delete。但是如果当前事务也需要对该记录进行更新操作，则很有可能造成死锁。

注：session1和session2是两个连接到MySQL的客户端，使用的数据库是从mysql官网下载的，下载地址：http://downloads.mysql.com/docs/sakila-db.zip 

> 下面是使用 lock in share mode加共享锁的例子：



session1:

```sql
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from actor where actor_id=178;
+----------+------------+-----------+---------------------+
| actor_id | first_name | last_name | last_update         |
+----------+------------+-----------+---------------------+
|      178 | LISA       | MONROE    | 2006-02-15 04:34:33 |
+----------+------------+-----------+---------------------+
1 row in set (0.00 sec)
```

session2:

```
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from actor where actor_id=178;
+----------+------------+-----------+---------------------+
| actor_id | first_name | last_name | last_update         |
+----------+------------+-----------+---------------------+
|      178 | LISA       | MONROE    | 2006-02-15 04:34:33 |
+----------+------------+-----------+---------------------+
1 row in set (0.00 sec)
```

&nbsp; &nbsp; session1对actor_id=178的记录加share mode的共享锁，session2也对actor_id=178，此时session1和session2能够加共享锁，如下：

```sql
session1
mysql> select * from actor where actor_id=178 lock in share mode;
+----------+------------+-----------+---------------------+
| actor_id | first_name | last_name | last_update         |
+----------+------------+-----------+---------------------+
|      178 | LISA       | MONROE    | 2006-02-15 04:34:33 |
+----------+------------+-----------+---------------------+
1 row in set (0.00 sec)

session2
mysql> select * from actor where actor_id=178 lock in share mode;
+----------+------------+-----------+---------------------+
| actor_id | first_name | last_name | last_update         |
+----------+------------+-----------+---------------------+
|      178 | LISA       | MONROE    | 2006-02-15 04:34:33 |
+----------+------------+-----------+---------------------+
1 row in set (0.00 sec)
```

&nbsp; &nbsp; 紧接着session1对178记录进行uodate，此时session1会等待锁，与此同时session2也对178记录更新，此时session2发生死锁，退出；

```sql
session1
mysql> update actor set last_name='monore t' where actor_id=178;
...等待

session2
mysql> mysql> update actor set last_name='monore t' where actor_id=178;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'mysql> update actor set last_name='monore t' where actor_id=178' at line 1
```

&nbsp; &nbsp; session2提交事务，session1获取到锁，update成功。

```sql
session2
mysql> commit;
Query OK, 0 rows affected (0.00 sec)

session1
Query OK, 1 row affected (49.29 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select * from actor where actor_id=178;
+----------+------------+-----------+---------------------+
| actor_id | first_name | last_name | last_update         |
+----------+------------+-----------+---------------------+
|      178 | LISA       | monore t  | 2021-09-02 12:47:50 |
+----------+------------+-----------+---------------------+
1 row in set (0.00 sec)
```

> 下面是使用for update加排他锁的例子：

&nbsp; &nbsp; session1对actor_id=178的行记录使用for update加排他锁，此时session2再次对178加排他锁是不会获取到锁的，会等待。

```sql
session1
mysql> start transaction;
Query OK, 0 rows affected (0.01 sec)

mysql> select * from actor where actor_id=178 for update;
+----------+------------+-----------+---------------------+
| actor_id | first_name | last_name | last_update         |
+----------+------------+-----------+---------------------+
|      178 | LISA       | monore t  | 2021-09-02 12:47:50 |
+----------+------------+-----------+---------------------+
1 row in set (0.00 sec)

session2
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from actor where actor_id=178 for update;
...等待
```

&nbsp; &nbsp; session1提交事务，session2获取到锁。

```sql
session1
mysql> commit;
Query OK, 0 rows affected (0.00 sec)

session2
mysql> select * from actor where actor_id=178 for update;
+----------+------------+-----------+---------------------+
| actor_id | first_name | last_name | last_update         |
+----------+------------+-----------+---------------------+
|      178 | LISA       | monore t  | 2021-09-02 12:47:50 |
+----------+------------+-----------+---------------------+
1 row in set (4.84 sec)
```

## InnoDB行锁的实现方式

&nbsp; &nbsp; InnoDB行锁是通过给索引伤的索引项加锁来实现的，如果没有索引，InnoDB将通过隐藏的聚集索引Row_id来对记录加锁，InnoDB行锁氛围3种情形。

- Record Lock：对索引项加锁。
- Gap lock：对索引项之间的“间隙”、第一条记录前的“间隙”或最后一条记录后的“间隙”加锁。
- Next-key lock：前两种的组合，对记录及其前面的间隙加锁 

&nbsp; &nbsp; InnoDB这种行锁实现特点意味着：如果不通过索引条件检索数据，那么InnoDB将对表中的所有记录加锁，实际效果跟锁表一样！在实际应用中，要特别注意InnoDB行锁这一特性，否则可能导致大量的锁冲突，从而影响并发性能。

> 在不通过索引条件查询时，InnoDB会锁定表中的所有记录。如下，payment表的amount字段没有索引。

&nbsp; &nbsp; session1加锁查询amount=8.99的数据，然后session2在加锁查询amount=3.99的数据，此时session2就会等待锁，session1 commit后session2获取到锁查询到数据。看起来session1只给amount=8.99的行加了锁，但是却出现了锁等待，原因就是在没有索引的情况下InnoDB对所有的记录加锁。

```sql
session1
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select payment_id, customer_id, staff_id, amount from payment where amount=8.99 for update;
+------------+-------------+----------+--------+
| payment_id | customer_id | staff_id | amount |
+------------+-------------+----------+--------+
|         62 |           3 |        1 |   8.99 |
|         81 |           3 |        1 |   8.99 |
|         83 |           3 |        2 |   8.99 |

...
+------------+-------------+----------+--------+

session2
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select payment_id, customer_id, staff_id, amount from payment where amount=3.99 for update;

...等待
```

&nbsp; &nbsp; 下面我们看一下通过索引条件加锁时的情况，例如session1加锁查询payment_id=62，session2加锁查询payment_id=81；此时使用了索引，加锁就只加在符合索引条件的记录上了，并没有出现等待锁情况。

```sql
session1
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select payment_id, customer_id, staff_id, amount from payment where payment_id=62 for update;
+------------+-------------+----------+--------+
| payment_id | customer_id | staff_id | amount |
+------------+-------------+----------+--------+
|         62 |           3 |        1 |   8.99 |
+------------+-------------+----------+--------+
1 row in set (0.00 sec)

session2
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select payment_id, customer_id, staff_id, amount from payment where payment_id=81 for update;
+------------+-------------+----------+--------+
| payment_id | customer_id | staff_id | amount |
+------------+-------------+----------+--------+
|         81 |           3 |        1 |   8.99 |
+------------+-------------+----------+--------+
1 row in set (0.00 sec)
```

> 由于MySQL的行锁是对索引加的锁，所以虽然访问了不同的记录，但是如果使用相同的索引键会出现冲突的。

&nbsp; &nbsp; 比如payment表staff_id有索引，amount没有索引，session1加锁查询staff_id=1 and amount=8.99的记录，session2加锁查询staff_id=1 and amount=3.99的记录，session2就会等待获取锁，虽然访问的是不同的行，因为锁是加在索引上，所以会产生锁冲突。session1 commit后session2获取锁成功。

```sql
session1
mysql> start transaction;
Query OK, 0 rows affected (0.01 sec)

mysql> select payment_id, customer_id, staff_id, amount from payment where staff_id=1 and amount=8.99 for update;
+------------+-------------+----------+--------+
| payment_id | customer_id | staff_id | amount |
+------------+-------------+----------+--------+
|         62 |           3 |        1 |   8.99 |
|         81 |           3 |        1 |   8.99 |
|        122 |           5 |        1 |   8.99 |
|        188 |           7 |        1 |   8.99 |
...
+------------+-------------+----------+--------+

session2
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select payment_id, customer_id, staff_id, amount from payment where staff_id=1 and amount=3.99 for update;
...等待
```

> 当表有多个索引的时候，不同事务可以使用不同的索引锁定不同的行，不论是使用主键索引、唯一索引或普通索引，InnoDB都会使用行锁来对数据加锁。

&nbsp; &nbsp; payment表的customer_id和staff_id是索引，session1加锁查询customer_id=3，session2加锁查询staff_id=1的行，customer_id=3的数据中包含staff_id=1的数据，session2会等待锁，因为session1锁了所有customer_id=3的行，包含staff_id=1的，所以session2或等待获取锁。

```sql
session1
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select payment_id, customer_id, staff_id, amount from payment where customer_id=3 for update;
+------------+-------------+----------+--------+
| payment_id | customer_id | staff_id | amount |
+------------+-------------+----------+--------+
|         60 |           3 |        1 |   1.99 |
|         61 |           3 |        1 |   2.99 |
|         62 |           3 |        1 |   8.99 |
|         63 |           3 |        1 |   6.99 |
|         64 |           3 |        2 |   6.99 |
|         65 |           3 |        1 |   2.99 |
|         66 |           3 |        1 |   4.99 |
|         67 |           3 |        1 |   4.99 |
|         68 |           3 |        1 |   5.99 |
|         69 |           3 |        2 |  10.99 |
|         70 |           3 |        2 |   7.99 |
|         71 |           3 |        2 |   6.99 |
|         72 |           3 |        1 |   4.99 |
|         73 |           3 |        2 |   4.99 |
|         74 |           3 |        1 |   2.99 |
|         75 |           3 |        1 |   1.99 |
|         76 |           3 |        2 |   3.99 |
|         77 |           3 |        1 |   2.99 |
|         78 |           3 |        2 |   4.99 |
|         79 |           3 |        2 |   5.99 |
|         80 |           3 |        2 |   4.99 |
|         81 |           3 |        1 |   8.99 |
|         82 |           3 |        2 |   2.99 |
|         83 |           3 |        2 |   8.99 |
|         84 |           3 |        2 |   0.99 |
|         85 |           3 |        1 |   2.99 |
+------------+-------------+----------+--------+
26 rows in set (0.00 sec)

session2
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select payment_id, customer_id, staff_id, amount from payment where staff_id=1 for update;

...等待
```

> 即便在条件中使用了索引字段，但是否使用索引来检索数据是有MySQL通过判断不同执行计划的代价来决定的，如果MySQL认为全表扫描效率更高，比如对一些很小的表就不会使用索引，这种情况InnoDB会对所有的行加锁 ，因此在分析锁冲突时别忘了分析sql执行计划。


## Next-Key锁

&nbsp; &nbsp; 当用范围条件而不是相等条件检索数据，并请求共享锁或排他锁时，InnoDB会给符合条件的数据行的索引项加锁，对于键值在条件范围内但并不存在的记录，叫做“间隙（GAP）”，InnoDB会对这个“间隙”加锁，这种锁机制就是所谓的Next-Key锁。

&nbsp; &nbsp; 举个例子，加锁查询payment_id>16048的数据，16049的记录会加锁，大于16049的记录（不存在）的“间隙”也会加锁。

```sql
session1
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select payment_id, customer_id, staff_id, amount from payment where payment_id>16048 for update;
+------------+-------------+----------+--------+
| payment_id | customer_id | staff_id | amount |
+------------+-------------+----------+--------+
|      16049 |         599 |        2 |   2.99 |
+------------+-------------+----------+--------+
1 row in set (0.00 sec)

session2
mysql> insert into payment (payment_id, customer_id, staff_id, amount) value (16050, 3, 2, 1.99);

... 等待
```

&nbsp; &nbsp; 还要特别说的是，InnoDB除了通过范围条件加锁时使用Next-Key锁外，如果使用相等条件请求一个不存在的记录加锁，InnoDB也会使用Next-Key。例如session1加锁请求payment_id=16051的记录（该记录不存在），session2 插入payment_id=16051的记录就会等待锁。

```sql
session1
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select payment_id, customer_id, staff_id, amount from payment where payment_id=16051 for update;
Empty set (0.01 sec)

session2
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into payment (payment_id, customer_id, staff_id, amount) value (16051, 3, 2, 1.99);

...等待
```

## 是什么时候用表锁

- 事务需要更新大部分或全部数据，表比较大，如果使用行锁，不仅这个事务执行效率低，而且可能造成其他事务长时间锁等待和锁冲突，这种情况下可以考虑使用表锁来提高事务的执行效率。
- 事务涉及到多个表，比较复杂，很可能引起死锁，造成大量事务回滚。这种情况也可以考虑一次性锁定事务涉及的表，从而避免死锁，减少数据库因事务回滚带来的开销。

当然这两种情况事务不能太多，否则，就应该考虑使用MyISAM表了。

## 死锁

&nbsp; &nbsp; 死锁示例，session1加锁查询payment表payment_id=15866记录，session2加锁查询actor表actor_id=200记录，之后session1加锁查询actor表actor_id=200，此时因为session2持有锁所以session1等待锁，然后session2加锁查询payment表payment_id=15866记录，这时InnoDB检测到了死锁，session2退出，session1查询到actor_id=200的记录。

```sql
session1
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from payment where payment_id=15866 for update;
+------------+-------------+----------+-----------+--------+---------------------+---------------------+
| payment_id | customer_id | staff_id | rental_id | amount | payment_date        | last_update         |
+------------+-------------+----------+-----------+--------+---------------------+---------------------+
|      15866 |         592 |        1 |     11410 |   8.99 | 2005-08-02 19:29:01 | 2006-02-15 22:23:29 |
+------------+-------------+----------+-----------+--------+---------------------+---------------------+
1 row in set (0.00 sec)

session2
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from actor where actor_id=200 for update;
+----------+------------+-----------+---------------------+
| actor_id | first_name | last_name | last_update         |
+----------+------------+-----------+---------------------+
|      200 | THORA      | TEMPLE    | 2006-02-15 04:34:33 |
+----------+------------+-----------+---------------------+
1 row in set (0.01 sec)

session1
mysql> select * from actor where actor_id=200 for update;

... 等待锁

session2，InnoDB检测到死锁，退出事务
mysql> select * from payment where payment_id=15866 for update;
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction


session2退出事务，session1获取到锁
mysql> select * from actor where actor_id=200 for update;

+----------+------------+-----------+---------------------+
| actor_id | first_name | last_name | last_update         |
+----------+------------+-----------+---------------------+
|      200 | THORA      | TEMPLE    | 2006-02-15 04:34:33 |
+----------+------------+-----------+---------------------+
1 row in set (6.53 sec)
```

&nbsp; &nbsp; 通过过例子可以看出，InnoDB一般能自动检测到死锁，并且使一个事务释放锁并回退，另一个事务获取到锁后可以继续正常执行，但是涉及外部锁以及表锁的情况下，InnoDB并不能完全自动检测到死锁，这就需要设置锁等待的超时时间innodb_lock_wait_timeout来解决。这个参数并不是只用来结局死锁问题，在并发比较高的情况下如果大量事务因无法立即获得所需的锁而挂起，会占用大量计算机资源，造成严重的性能问题，甚至拖垮数据库。通过设置合适的锁等待超市阈值可以避免这种情况发生。


&nbsp; &nbsp; 避免死锁的常用方法

- 在应用中，如果不同的程序会并发存取多个表，应尽量约定以相同的顺序来访问表，这样可以大大降低产生死锁的机会。
- 在程序以批量的方式处理数据的时候，如果实现对数据排序，保证每个线程按照固定的顺序来处理记录，也可以大大降低出现死锁的可能。
- 在事务中，如果要更新记录，应该直接申请足够级别的锁，即排他锁，不应该先申请共享锁再在更新时申请排他锁，防止其他事务在更新时获取到共享锁，造成锁冲突甚至是死锁。
- 在REPEATABLE-READ隔离级别下，如果两个线程同时对相同记录用select ... for update加排他锁，在没有符合记录的情况下，两个线程都会加锁成功。程序发现记录不存在，就试图插入一条新纪录，如果两个线程都这么做，就会出现死锁。
- 当隔离级别为READ COMMITED时，如果两个线程都先执行select ... for update，判断是否存在符合条件记录，如果没有，就插入记录。此时，只有一个线程能插入成功，另一个线程会出现锁等待，当第一个线程提交后，第二个线程因主键重复出错，虽然出错了，却会获得一个排他锁！这时如果第3个线程又来申请排他锁，也会出现死锁。


## 总结

- InnoDB的行锁是基于索引实现的，如果不通过索引加锁访问数据，InnoDB会对所有数据加锁。
- 在不同隔离级别下，InnoDB的锁机制和一致性读取策略不通。
- 介绍了Next-Key锁机制，以及InnoDB使用Next-Key锁的原因。
- 精心设计索引，并尽量使用索引访问数据，使加锁的粒度更小，从而减少锁冲突的机会。
- 选择合理的事务，小事务发生死锁的概率小。
- 尽量使用相等的条件访问数据，避免Next-Key锁对并发插入的影响。
- 避免死锁的一些常用方法
- 对于一些特定的事务，可以利用表锁来提高处理速度
- 不要申请超过实际需要的锁级别；除非必须，查询时不要显示加锁。