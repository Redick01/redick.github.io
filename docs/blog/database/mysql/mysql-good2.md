# MySQL-SQL语句优化 <!-- {docsify-ignore-all} -->


## 大批量插入数据

- **MyISAM引擎**

&nbsp; &nbsp; 通过load命令可以导入数据，适当的设置可以提高load命令的导入速度，DSIABLE KEYS和ENABLE KEYS 用来关闭和打开MyISAM表非唯一索引的更新，导入大量数据到非空的表关闭非唯一索引可以提高load的效率，再load完成后打开非唯一索引即可。对于一个空的表导入大量数据可以不必有此操作因为默认是先导入数据然后才创建索引。

- **InnoDB** 

    - 导入数据按照主键顺序排列，可以有效地提高导入数据效率
    - 在导入数据前执行SET UNIQUE_CHECK=0，关闭唯一性校验，导入结束后再SET UNIQUE_CHECK=1恢复唯一性校验，可以提高导入效率
    - 如果应用使用自动提交的方式，建议子啊导入前执行SET AUTOCOMMIT=0，关闭自动提交，导入结束后再执行SET AUTOCOMMOT=1，打开自动提交，也可以提高导入效率。

## 优化INSERT语句

- insert多条数据时尽量使用多个值表的insert语句，这种方式将大大所见客户端与数据库之间的连接、关闭等消耗，使得效率比分开执行的单个insert语句快
- 如果从不通的客户端插入很多航，可以通过使用insert delayed语句得到更高的速度，delayed的含义是让insert语句马上执行，数据被放在内存的队列中，并没有真正写入磁盘，这笔每条语句分别插入要快很多。
- 将索引文件和数据文件分在不同的磁盘上存放（利用建表中的选项）
- 如果进行批量插入，可以通过增加bulk_insert_buffer_size变量值的方法来提高速度，仅针对MyISAM引擎
- 当从一个文本文件装载一个表时，使用LOAD DATA INFILE，这通常比使用很多的INSERT语句快20倍。

## 优化ORDER BY语句

> MySQL两种排序方式

1. 通过有序索引顺序扫描直接返回有序数据，这种方式在使用explain分析查询的时候显示未Using Index，不需要额外的排序，操作效率较高。
2. 通过对返回的数据进行排序，也就是常说的Filesort排序，所有不是通过索引直接返回排序结果的排序都叫Filesort排序。

> 优化

&nbsp; &nbsp; 尽量减少额外的排序，通过索引直接返回有序数据。where条件和order by使用相同的索引，并且order by的顺序和索引顺序相同，并且order by的字段都是圣墟或者都是降序排序，否则需要额外的排序操作，这样就会出现Filesort。

> Filesort优化

- 两次扫描算法
- 一次扫描算法

## 优化GROUP BY语句

&nbsp; &nbsp; group by默认情况下对其分组条件字段进行了排序，如果想避免排序带来的性能消耗，可以指定 ORDER BY NULL禁止排序

## 优化嵌套查询

&nbsp; &nbsp; JOIN代替

## MySQL优化OR条件

&nbsp; &nbsp; 对于or的查询子句，如果要利用索引，则or之间的每个条件列都必须用到索引，如果没有索引，应该考虑建立索引，就是说每个条件列单独拎出来都要能走索引，比如列a和列b都是索引，那么条件a or b是走索引的，通过执行计划可以看出结果进行了union操作，如果a和b是联合索引那么a or b不走索引。

## 优化分页查询

&nbsp; &nbsp; 一般分页查询时，通过创建覆盖索引能够比较好滴提高性能。一个常见又非常头疼的分页场景时“limit 1000,20”，此时MySQL排序出前1020条记录后仅仅需要返回第1001到1020条记录，前1000条记录都会被抛弃，查询和排序的代价非常高。

> 优化思路一

&nbsp; &nbsp; 按照索引分页后回表方式改写SQL，这种方式能让MySQL扫描尽可能少的页面来提高分页效率。

> 优化思路二

&nbsp; &nbsp; 把LIMIT查询转换成某个位置的查询，例如，假设每页10条记录，查询表a中按照a_id字段逆序排序的第42页记录，能够看到执行计划走了全表扫描

```sql
explain select * from a order by a_id desc limit 410, 10;
```
&nbsp; &nbsp; 和开发人员协商，翻页的过程通过增加一个参数last_page_record，用来记录上一页在后一行的a_id，例如第41页最后一行的a_id=101，那么翻页到42页是，可以根据41页的最后一条记录在向后查sql可以改为：

```sql
explain select * from a where a_id a_id < 101 order by a_id limit 10;
```
这时，就把limit m, n转化成了limit n的查询，这种情况适合有排序的特定场景，能够减轻分页的压力；如果排序出现大量的重复值，那么分页结果会导致丢失记录，不适合这种方式优化。

## 使用SQL提示

&nbsp; &nbsp; SQL提示是优化数据库的一个重要手段，简单来说就是在SQL语句中加入一些人为的提示来达到优化操作的目的，下面是一些MySQL中常用的SQL提示。

> USER INDEX

在查询语句中表名的后面，添加USE INDEX来提供希望MySQL去参考的索引列表，就可以让MySQL不再考虑其他可用的索引。

```sql
explain select count(*) from a use index (index_1);
```

> IGNORE INDEX

如果用户知识单纯的想让MySQL忽略一个或者多个索引，则可以使用IGNORE INDEX，同样的，看一下忽略索引的用法

```sql
explain select count(*) from a ignore index (index_1);
```

> FORCE INDEX

&nbsp; &nbsp; 为强制MySQL使用一个特定的索引，可在查询中使用FORCE INDEX，例如，当不强制使用索引的时候，对于查询表a条件a_id>1时，因为大部分数据的a_id都大于1，因此MySQL会默认进行全表扫描，这种情况下使用use index发现MySQL还是选择了全表扫描，但是，当使用FORCE INDEX进行提示时，即便是使用索引的效率不是最高，MySQL还是选择使用索引，这是MySQL留给用户的一个自行选择执行计划的权利。



## 参考

《深入浅出MySQL》