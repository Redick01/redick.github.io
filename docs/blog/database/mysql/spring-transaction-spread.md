# Spring事务管理传播机制

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

#### REQUIRED
