# MySQL-SQL语句优化 <!-- {docsify-ignore-all} -->


## 大批量插入数据

- **MyISAM引擎**

&nbsp; &nbsp; 通过load命令可以导入数据，适当的设置可以提高load命令的导入速度，DSIABLE KEYS和ENABLE KEYS 用来关闭和打开MyISAM表非唯一索引的更新，导入大量数据到非空的表关闭非唯一索引可以提高load的效率，再load完成后打开非唯一索引即可。对于一个空的表导入大量数据可以不必有此操作因为默认是先导入数据然后才创建索引。

- **InnoDB** 

    - 导入数据按照主键顺序排列，可以有效地提高导入数据效率
    - 在导入数据前执行SET UNIQUE_CHECK=0，关闭唯一性校验，导入结束后再SET UNIQUE_CHECK=1恢复唯一性校验，可以提高导入效率
    - 如果应用使用自动提交的方式，建议子啊导入前执行SET AUTOCOMMIT=0，关闭自动提交，导入结束后再执行SET AUTOCOMMOT=1，打开自动提交，也可以提高导入效率。

## 优化INSERT语句

- insert多条数据时尽量使用多个值表的insert语句，这种方式将大大所见客户端与数据库之间的连接