# Spring事务管理传播机制 <!-- {docsify-ignore-all} -->

- 什么是事务传播机制
- Spring事务传播的机制

参考：https://zhuanlan.zhihu.com/p/148504094

## 什么是事务传播机制

&nbsp; &nbsp; 简单的理解就是多个事务方法相互调用时,事务如何在这些方法间传播。

&nbsp; &nbsp; 举个栗子，方法A是一个事务的方法，方法A执行过程中调用了方法B，那么方法B有无事务以及方法B对事务的要求不同都会对方法A的事务具体执行造成影响，同时方法A的事务对方法B的事务执行也有影响，这种影响具体是什么就由两个方法所定义的事务传播类型所决定。

## Spring事务传播的机制

&nbsp; &nbsp; 在Spring中对于事务的传播行为定义了七种类型分别是：REQUIRED、SUPPORTS、MANDATORY、REQUIRES_NEW、NOT_SUPPORTED、NEVER、NESTED。在Spring源码中这七种类型被定义为了枚举。源码在org.springframework.transaction.annotation包下的Propagation，源码中注释很多，对传播行为的七种类型的不同含义都有解释，后文中锤子我也会给大家分析，我在这里就不贴所有的源码，只把这个类上的注解贴一下，翻译一下就是：表示与TransactionDefinition接口相对应的用于@Transactional注解的事务传播行为的枚举。也就是说枚举类Propagation是为了结合@Transactional注解使用而设计的，这个枚举里面定义的事务传播行为类型与TransactionDefinition中定义的事务传播行为类型是对应的，所以在使用@Transactional注解时我们就要使用Propagation枚举类来指定传播行为类型，而不直接使用TransactionDefinition接口里定义的属性。在TransactionDefinition接口中定义了Spring事务的一些属性，不仅包括事务传播特性类型，还包括了事务的隔离级别类型（事务的隔离级别后面文章会详细讲解），更多详细信息，大家可以打开源码自己翻译一下里面的注释


- REQUIRED：Spring默认的事务传播类型，如果当前没有事务，则自己新建一个事务，如果当前存在事务，则加入这个事务。
- SUPPORTS：当前存在事务，则加入当前事务，如果当前没有事务，就以非事务方法执行。
- MANDATORY：当前存在事务，则加入当前事务，如果当前事务不存在，则抛出异常。
- REQUIRES_NEW：创建一个新事务，如果存在当前事务，则挂起该事务。
- NOT_SUPPORTED：始终以非事务方式执行,如果当前存在事务，则挂起当前事务，可以理解为设置事务传播类型为NOT_SUPPORTED的方法，在执行时，不论当前是否存在事务，都会以非事务的方式运行。
- NEVER：不使用事务，如果当前事务存在，则抛出异常
- NESTED：如果当前事务存在，则在嵌套事务中执行，否则REQUIRED的操作一样（开启一个事务）

注意：**Spring中事务的默认实现使用的是AOP，也就是代理的方式，如果大家在使用代码测试时，同一个Service类中的方法相互调用需要使用注入的对象来调用，不要直接使用this.方法名来调用，this.方法名调用是对象内部方法调用，不会通过Spring代理，也就是事务不会起作用**

### Spring事务验证示例

&nbsp; &nbsp; 写一个springboot+mybatis的demo用于验证Spring事务传播，下面是核心的代码，在后续的七种传播机制的验证时就只展示改动部分了，首先看下面的代码没有使用spring事务，这里我自己制造了一个空指针异常，无事务测试中只调用`required`方法。验证结果是insert两条数据，空指针异常后边的代码没有执行，没有insert。

```
@Service
public class TransactionService {


    private final AccountMapper accountMapper;

    private Account account = null;

    @Autowired
    public TransactionService(AccountMapper accountMapper) {
        this.accountMapper = accountMapper;
    }

    public void required() {
        insertA();
        testA();
    }

    public void testA() {
        insertB();
        account.getBalance();
        insertB();
    }

    private void insertA() {
        Account account = new Account();
        account.setBalance(new Long(100));
        account.setCreateTime(new Date());
        account.setFreezeAmount(new Long(0));
        account.setUpdateTime(new Date());
        account.setUserId("123");
        accountMapper.insertSelective(account);
    }

    private void insertB() {
        Account account = new Account();
        account.setBalance(new Long(200));
        account.setCreateTime(new Date());
        account.setFreezeAmount(new Long(0));
        account.setUpdateTime(new Date());
        account.setUserId("456");
        accountMapper.insertSelective(account);
    }
}
```

数据库执行结果

```
mysql> select * from account;
+----+---------+---------+---------------+---------------------+---------------------+
| id | user_id | balance | freeze_amount | create_time         | update_time         |
+----+---------+---------+---------------+---------------------+---------------------+
| 23 | 123     |     100 |             0 | 2021-05-30 18:16:45 | 2021-05-30 18:16:45 |
| 24 | 456     |     200 |             0 | 2021-05-30 18:16:45 | 2021-05-30 18:16:45 |
+----+---------+---------+---------------+---------------------+---------------------+
2 rows in set (0.00 sec)
```

#### REQUIRED

&nbsp; &nbsp; `REQUIRED`该机制是Spring默认的事务传播机制，如果没有事务就会创建一个事务，如果存在事务则加入当前事务，下面是改动的代码，验证的结果是不会insert任何的数据，因为增加了事务`@Transactional`，testA方法产生的异常也会上抛到required方法中，在required方法中并没有catch异常，所以这里会回滚不会insert任何数据。如果这里catch住required方法中的`testA();`会怎么样？这时就会insert两条数据，分别是`insertA();`和testA()方法中的第一个`insertB();`插入的数据，因为事务传播时`REQUIRED`，所以testA()是加入到了required()的事务中，required()方法catch住了异常，因此会commit，但是testA()没有任何异常处理，所以异常后的代码不会执行，第二个insert也就不会执行了，所以结果就是insert两条数据。

```
    @Transactional(propagation = Propagation.REQUIRED)
    public void required() {
        insertA();
        testA();
    }

    @Transactional(propagation = Propagation.REQUIRED)
    public void testA() {
        insertB();
        account.getBalance();
        insertB();
    }   
```

数据库执行结果，没有数据新增，还是刚才那两条

```
mysql> select * from account;
+----+---------+---------+---------------+---------------------+---------------------+
| id | user_id | balance | freeze_amount | create_time         | update_time         |
+----+---------+---------+---------------+---------------------+---------------------+
| 23 | 123     |     100 |             0 | 2021-05-30 18:16:45 | 2021-05-30 18:16:45 |
| 24 | 456     |     200 |             0 | 2021-05-30 18:16:45 | 2021-05-30 18:16:45 |
+----+---------+---------+---------------+---------------------+---------------------+
2 rows in set (0.00 sec)
```

&nbsp; &nbsp; 如果只有testA声明事务呢，如下代码展示的，测试的结果是insertA方法插入数据成功，testA方法没有insert成功。

注意：如果`transactionService`中的一个没有声明事务的方法调用了一个声明事务的方法（自身调用），事务就会失效，因为自身调用并不是经过Spring代理的类，所以这时候事务失效了，Spring事务管理器管理的是Spring代理的类，这只是事务失效的一种原因。

```
    // 测试的Controller
    @GetMapping("/trans/transTest")
    public String testTransaction() {
        transactionService.insertA();
        transactionService.testA();
        return "test";
    }

    public void insertA() {
        Account account = new Account();
        account.setBalance(new Long(100));
        account.setCreateTime(new Date());
        account.setFreezeAmount(new Long(0));
        account.setUpdateTime(new Date());
        account.setUserId("123");
        accountMapper.insertSelective(account);
    }

    @Transactional(propagation = Propagation.REQUIRED)
    public void testA() {
        insertB();
        account.getBalance();
        insertB();
    }
```

数据库执行结果，insertA插入数据成功，testA并没插入数据

```
mysql> select * from account;
+----+---------+---------+---------------+---------------------+---------------------+
| id | user_id | balance | freeze_amount | create_time         | update_time         |
+----+---------+---------+---------------+---------------------+---------------------+
| 23 | 123     |     100 |             0 | 2021-05-30 18:16:45 | 2021-05-30 18:16:45 |
| 24 | 456     |     200 |             0 | 2021-05-30 18:16:45 | 2021-05-30 18:16:45 |
| 31 | 123     |     100 |             0 | 2021-05-30 18:31:25 | 2021-05-30 18:31:25 |
+----+---------+---------+---------------+---------------------+---------------------+
3 rows in set (0.00 sec)
```

#### SUPPORTS

**当前存在事务，则加入当前事务，如果当前没有事务，就以非事务方法执行**

测试代码如下，测试的结果是testA方法的第一个insertB插入数据成功，testA以非事务的形式执行，`account.getBalance();`抛出异常后第二个insertB和insertA都没有执行。

```
    @GetMapping("/trans/transTest")
    public String testTransaction() {
        transactionService.testA();
        transactionService.insertA();
        return "test";
    }

    @Transactional(propagation = Propagation.SUPPORTS)
    public void testA() {
        insertB();
        account.getBalance();
        insertB();
    }

    public void insertA() {
        Account account = new Account();
        account.setBalance(new Long(100));
        account.setCreateTime(new Date());
        account.setFreezeAmount(new Long(0));
        account.setUpdateTime(new Date());
        account.setUserId("123");
        accountMapper.insertSelective(account);
    }

    public void insertB() {
        Account account = new Account();
        account.setBalance(new Long(200));
        account.setCreateTime(new Date());
        account.setFreezeAmount(new Long(0));
        account.setUpdateTime(new Date());
        account.setUserId("456");
        accountMapper.insertSelective(account);
    }
```

数据库执行结果

```
mysql> select * from account;
+----+---------+---------+---------------+---------------------+---------------------+
| id | user_id | balance | freeze_amount | create_time         | update_time         |
+----+---------+---------+---------------+---------------------+---------------------+
| 41 | 456     |     200 |             0 | 2021-05-30 19:25:19 | 2021-05-30 19:25:19 |
+----+---------+---------+---------------+---------------------+---------------------+
1 row in set (0.00 sec)
```

#### MANDATORY

**当前存在事务，则加入当前事务，如果当前事务不存在，则抛出异常。**

测试代码如下，与`SUPPORTS`的代码一样只是讲传播机制改为`MANDATORY`，代码的执行结果是抛出了异常，因为我只在testA方法上声明了事务，所以不存事务的情况下`MANDATORY`机制就抛出了异常。

```
    @Transactional(propagation = Propagation.MANDATORY)
    public void testA() {
        insertB();
        account.getBalance();
        insertB();
    }
```
抛出异常堆栈

```
org.springframework.transaction.IllegalTransactionStateException: No existing transaction found for transaction marked with propagation 'mandatory'
```

调整测试代码，将required方法声明事务，事务传播机制为`REQUIRED`，代码如下，执行结果是不会插入任何数据，因为testA方法加到了required方法的事务中，` account.getBalance();`抛出空指针异常执行了事务的回滚。

```
    @Transactional(propagation = Propagation.REQUIRED)
    public void required() {
        insertA();
        testA();
    }

    @Transactional(propagation = Propagation.MANDATORY)
    public void testA() {
        insertB();
        account.getBalance();
        insertB();
    }
```

#### REQUIRES_NEW

**创建一个新事务，如果存在当前事务，则挂起该事务。**

测试代码如下，testA方法声明事务传播机制为`REQUIRES_NEW`，改动required方法，加入`account.getBalance();`，这里验证testA方法的事务是新开启的事务而不是加入到required方法的事务，测试结果是testA方法两条insert执行成功，required方法的insertA没有插入数据发生了回滚，与之相对的就是`REQUIRED`测试的例子，这里如果testA方法的事务传播机制是`REQUIRED`，那么就都不会插入数据成功，因为testA加入到了required方法的事务，并没有开启新的事务。

```
    @GetMapping("/trans/transTest")
    @Transactional(propagation = Propagation.REQUIRED)
    public String testTransaction() {
        transactionService.insertA();
        transactionService.testA();
        account.getBalance();
        return "test";
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void testA() {
        insertB();
        insertB();
    }

    public void insertA() {
        Account account = new Account();
        account.setBalance(new Long(100));
        account.setCreateTime(new Date());
        account.setFreezeAmount(new Long(0));
        account.setUpdateTime(new Date());
        account.setUserId("123");
        accountMapper.insertSelective(account);
    }
```

测试结果， testA()开启新的事务，插入了两条user_id为456的数据，结果如下：

```
mysql> select * from account;
+----+---------+---------+---------------+---------------------+---------------------+
| id | user_id | balance | freeze_amount | create_time         | update_time         |
+----+---------+---------+---------------+---------------------+---------------------+
| 67 | 456     |     200 |             0 | 2021-05-30 19:54:26 | 2021-05-30 19:54:26 |
| 68 | 456     |     200 |             0 | 2021-05-30 19:54:26 | 2021-05-30 19:54:26 |
+----+---------+---------+---------------+---------------------+---------------------+
2 rows in set (0.00 sec)
```

#### NOT_SUPPORTED 

**始终以非事务方式执行,如果当前存在事务，则挂起当前事务**

代码如下，testTransaction()方法声明了`REQUIRED`的事务，testA()方法声明了`NOT_SUPPORTED`的事务，受到异常影响insertA进行了回滚，testA是非事务的方式，所以testA方法第一个isnertB执行成功，插入一条数据。

```
    @GetMapping("/trans/transTest")
    @Transactional(propagation = Propagation.REQUIRED)
    public String testTransaction() {
        transactionService.insertA();
        transactionService.testA();
        return "test";
    }

    @Transactional(propagation = Propagation.NOT_SUPPORTED)
    public void testA() {
        insertB();
        account.getBalance();
        insertB();
    }

    public void insertA() {
        Account account = new Account();
        account.setBalance(new Long(100));
        account.setCreateTime(new Date());
        account.setFreezeAmount(new Long(0));
        account.setUpdateTime(new Date());
        account.setUserId("123");
        accountMapper.insertSelective(account);
    }
```

执行结果，插入了一条user_id为456的数据

```
mysql> select * from account;
+----+---------+---------+---------------+---------------------+---------------------+
| id | user_id | balance | freeze_amount | create_time         | update_time         |
+----+---------+---------+---------------+---------------------+---------------------+
| 75 | 456     |     200 |             0 | 2021-05-30 20:03:23 | 2021-05-30 20:03:23 |
+----+---------+---------+---------------+---------------------+---------------------+
1 row in set (0.01 sec)
```

#### NEVER

**不使用事务，如果当前事务存在，则抛出异常**

测试代码如下，testA方法声明事务传播方式为`NEVER`，执行测试，结果抛出异常。

```
    @GetMapping("/trans/transTest")
    @Transactional(propagation = Propagation.REQUIRED)
    public String testTransaction() {
        transactionService.insertA();
        transactionService.testA();
        return "test";
    }

    @Transactional(propagation = Propagation.NEVER)
    public void testA() {
        insertB();
        account.getBalance();
        insertB();
    }
```

异常堆栈

```
org.springframework.transaction.IllegalTransactionStateException: Existing transaction found for transaction marked with propagation 'never'
```

#### NESTED

**如果当前事务存在，则在嵌套事务中执行，否则REQUIRED的操作一样（开启一个事务）**

- 与REQUIRES_NEW的区别

REQUIRES_NEW是新建一个事务并且新开启的这个事务与原有事务无关，而NESTED则是当前存在事务时（我们把当前事务称之为父事务）会开启一个嵌套事务（称之为一个子事务）。在NESTED情况下父事务回滚时，子事务也会回滚，而在REQUIRES_NEW情况下，原有事务回滚，不会影响新开启的事务。

- 与REQUIRED的区别

REQUIRED情况下，调用方存在事务时，则被调用方和调用方使用同一事务，那么被调用方出现异常时，由于共用一个事务，所以无论调用方是否catch其异常，事务都会回滚；而在NESTED情况下，被调用方发生异常时，调用方可以catch其异常，这样只有子事务回滚，父事务不受影响


1. **示例1，父事务抛出异常，子事务也进行回滚**

insertA和testA方法中的插入数据都不会成功，因为父事务中抛出了异常导致了回滚，子事务也受到了影响也会回滚

```
    @GetMapping("/trans/transTest")
    @Transactional(propagation = Propagation.REQUIRED)
    public String testTransaction() {
        transactionService.insertA();
        transactionService.testA();
        account.getBalance();
        return "test";
    }

    @Transactional(propagation = Propagation.NESTED)
    public void testA() {
        insertB();
        insertB();
    }
```

1. **示例2，父事务抛出异常，子事务也进行回滚**

testA方法声明了`NESTED`的事务，testA发生的异常被catch住了，所以子事务并没有影响父事务

```
    @GetMapping("/trans/transTest")
    @Transactional(propagation = Propagation.REQUIRED)
    public String testTransaction() {
        transactionService.insertA();
        try {
            transactionService.testA();
        } catch (Exception e) {
            e.printStackTrace();
        }
        transactionService.insertA();
        return "test";
    }

    @Transactional(propagation = Propagation.NESTED)
    public void testA() {
        insertB();
        account.getBalance();
        insertB();
    }

    public void insertA() {
        Account account = new Account();
        account.setBalance(new Long(100));
        account.setCreateTime(new Date());
        account.setFreezeAmount(new Long(0));
        account.setUpdateTime(new Date());
        account.setUserId("123");
        accountMapper.insertSelective(account);
    }
```

测试结果，插入了两条id为123的数据

```
mysql> select * from account;
+----+---------+---------+---------------+---------------------+---------------------+
| id | user_id | balance | freeze_amount | create_time         | update_time         |
+----+---------+---------+---------------+---------------------+---------------------+
| 80 | 123     |     100 |             0 | 2021-05-30 20:22:57 | 2021-05-30 20:22:57 |
| 82 | 123     |     100 |             0 | 2021-05-30 20:22:57 | 2021-05-30 20:22:57 |
+----+---------+---------+---------------+---------------------+---------------------+
2 rows in set (0.00 sec)
```