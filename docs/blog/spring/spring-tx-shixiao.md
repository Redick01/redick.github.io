# Spring事务失效 <!-- {docsify-ignore-all} -->
 
## 前言

&nbsp; &nbsp; 在使用Spring的事务管理时，有时候就会莫名其妙的发现事务没生效，其实并非Spring事务管理本身的问题，而是开发人员在使用时使用方式不对，Spring事务管理的底层机制没搞懂，又或者数据库层的问题导致，今天总结下Spring事务是失效的集中场景。


## 常见的Spring事务失效原因

- **没有被Spring管理**

&nbsp; &nbsp; 不是由Spring管理的Bean，例如，一个类没有在xml中声明或者没有`@Service,@RestController,@Component`等注解的方式声明由Spring管理，但在类的方法中使用`@Transactional`注解声明事务，这种情况事务是不会生效的，也就是说Spring管理的事务只能是在Spring容器管理的Bean中声明才会生效。如下代码，testA方法虽然声明了事务，但是由于`@Service`注解被注释了，所以事务不会生效。

```
//@Service
public class TransactionService {


    private final AccountMapper accountMapper;

    private Account account = null;

    @Autowired
    public TransactionService(AccountMapper accountMapper) {
        this.accountMapper = accountMapper;
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

    public void insertB() {
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

- **发生自调用**

先上代码，required方法没有加注解声明事务，required方法调用了testA方法，testA方法声明了事务，但是这里事务不会生效，其原因是因为Spring事务管理是通过代理类实现的，然而这种自调用相当于this.testA()，是通过对象调用，无法走到事务的切面中，所以事务就不生效了。

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

    @Transactional
    public void testA() {
        insertB();
        account.getBalance();
        insertB();
    }
}
```

再看下面代码，这时required()方法声明了事务，这时testA方法声明了`REQUIRES_NEW`传播机制的事务，testA方法的事务会生效吗？答案是不会生效，一样的问题产生了自调用，并没有经过Spring的代理，所以事务不生效。

```
@Service
public class TransactionService {


    private final AccountMapper accountMapper;

    private Account account = null;

    @Autowired
    public TransactionService(AccountMapper accountMapper) {
        this.accountMapper = accountMapper;
    }
    
    @Transactional
    public void required() {
        insertA();
        testA();
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void testA() {
        insertB();
        account.getBalance();
        insertB();
    }
}
```

- **方法不是public的**

来自 Spring 官方文档：

    When using proxies, you should apply the @Transactional annotation only to methods with public visibility. If you do annotate protected, private or package-visible methods with the @Transactional annotation, no error is raised, but the annotated method does not exhibit the configured transactional settings. Consider the use of AspectJ (see below) if you need to annotate non-public methods.

大概意思就是 @Transactional 只能用于 public 的方法上，否则事务不会失效，如果要用在非 public 方法上，可以开启 AspectJ 代理模式。

- **数据源未配置事务管理器**

    数据源没有配置事务管理器，声明了事务也是白搭，下面是配置事务管理器的代码

```
    @Bean(name = "transactionManager")
    public DataSourceTransactionManager sentinelTransactionManager(@Qualifier("datasource") DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
```

- **数据库引擎不支持事务**

    以MySQL为例，其MyISAM引擎是不支持事务操作的，InnoDB才是支持事务的引擎，一般要支持事务都会使用InnoDB。如果数据库引擎不支持事务，那也是白搭。

- **事务传播机制设置以不支持事务运行**

    Propagation.NOT_SUPPORTED： 表示不以事务运行，当前若存在事务则挂起。

- **异常被catch掉了**

    如下面的代码，如果insertB方法中，插入数据库后产生异常，并不会回滚，因为异常被catch调了，事务失效。

```
    @Transactional
    public void testA() {
        try {
            insertB();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

- **异常类型错误**

    事务默认回滚的异常是RuntimeException，如果想触发其他异常的回滚，需要在注解上配置，例如：

```
@Transactional(rollbackFor = Exception.class)
```

## 总结

    本文总结了八种事务失效的场景，其实发生最多就是自身调用、异常被吃、异常抛出类型不对这三个了。