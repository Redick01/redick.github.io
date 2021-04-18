# ActiveMQ持久化方式

## 本文参考

官方文档：https://activemq.apache.org/persistence

博客：https://blog.csdn.net/weixin_42796626/article/details/98206384


## 持久化方式支持

目前最新版本支持的消息持久化方式有以下几种，其他版本的持久化支持情况请参考官方文档

- kahaDB文件持久化
- jdbc持久化
- levelDB存储
- levelDB主从复制
- AMQ持久化；（不推荐，可以用kahaDB替代）
- Memory内存持久化；（不推荐，容易丢失数据）

## 持久化配置及特点

如果没有特殊情况，activeMQ的持久化配置文件为/config/activemq.xml

### kahaDB文件持久化

从activeMQ 5.3版本开始官方建议使用KahaDB，它提供了比AMQ消息存储更高的可伸缩性和可恢复性。AMQ消息存储虽然比KahaDB快，但扩展性不如KahaDB，恢复时间也更长。

Kaha是默认的持久化策略，所有消息顺序添加到一个日志文件中，同时另外有一个索引文件记录指向这些日志的存储地址，还有一个事务日志用于消息回复操作。是一个专门针对消息持久化的解决方案,它对典型的消息使用模式进行了优化。

在data/kahadb这个目录下，会生成四个文件，来完成消息持久化
1. db.data 它是消息的索引文件，本质上是B-Tree（B树），使用B-Tree作为索引指向db-*.log里面存储的消息
2. db.redo 用来进行消息恢复
3. db-*.log 存储消息内容。新的数据以APPEND的方式追加到日志文件末尾。属于顺序写入，因此消息存储是比较 快的。默认是32M，达到阀值会自动递增
4. lock文件 锁，写入当前获得kahadb读写权限的broker ，用于在集群环境下的竞争处理
 

```
<persistenceAdapter>
    <kahaDB directory="${activemq.data}/kahadb" journalMaxFileLength="16mb"/>
</persistenceAdapter>
```

更多参数配置参考官网：https://activemq.apache.org/kahadb

### JDBC持久化

jdbc持久化需要配置一个持久化适配器，该适配器指向一个数据源的bean。特别注意，<bean>应该在<beans>的根元素配置，而不是跟<persistenceAdapter>配置在一起。官方建议在长期持久化的场景中可以使用JDBC和高性能日志（High performance journal）的组合，单独只使用JDBC也可以但是速度会很慢。

在消息消费者能跟上生产者的速度时，journal文件能大大减少需要写入到DB中的消息。举个例子：生产者产生了10000个消息，这10000个消息会保存到journal文件中，但是消费者的速度很快，在journal文件还未同步到DB之前，以消费了9900个消息。那么后面就只需要写入100个消息到DB了。如果消费者不能跟上生产者的速度，journal文件可以使消息以批量的方式写入DB中，JDBC驱动进行DB写入的优化。从而提升了性能。另外，journal文件支持JMS事务的一致性。

官方默认支持使用Apache Derby作为默认数据库，该数据库易于嵌入，也支持其他的SQL数据库，如：MySQL，Oracle，只需在Xml Configuration中重新配置JDBC配置即可。

- **JDBC+高性能日志配置样例**

以下配置来自于官网
```
<beans xmlns="http://www.springframework.org/schema/beans" 
       xmlns:amq="http://activemq.apache.org/schema/core" 
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" 
       xsi:schemaLocation="http://www.springframework.org/schema/beans 
                           http://www.springframework.org/schema/beans/spring-beans-2.0.xsd 
                           http://activemq.apache.org/schema/core 
                           http://activemq.apache.org/schema/core/activemq-core.xsd"> 
  <bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer"/> 
  <broker useJmx="true" xmlns="http://activemq.apache.org/schema/core"> 
    <networkConnectors> 
      <!-- <networkConnector uri="multicast://default?initialReconnectDelay=100" /> <networkConnector uri="static://(tcp://localhost:61616)" /> --> 
    </networkConnectors> 
    <persistenceFactory>
      <journalPersistenceAdapterFactory journalLogFiles="5" dataDirectory="${basedir}/target" /> 
      <!-- To use a different dataSource, use the following syntax : --> 
      <!-- <journalPersistenceAdapterFactory journalLogFiles="5" dataDirectory="${basedir}/activemq-data" dataSource="#mysql-ds"/> --> 
    </persistenceFactory> 
    <transportConnectors> 
      <transportConnector uri="tcp://localhost:61636" /> 
    </transportConnectors> 
  </broker> 
  <!-- MySql DataSource Sample Setup --> 
  <!-- 
  <bean id="mysql-ds" class="org.apache.commons.dbcp2.BasicDataSource" destroy-method="close"> 
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/> 
    <property name="url" value="jdbc:mysql://localhost/activemq?relaxAutoCommit=true"/> 
    <property name="username" value="activemq"/> 
    <property name="password" value="activemq"/> 
    <property name="poolPreparedStatements" value="true"/> 
  </bean> 
  --> 
</beans>
```

- **JDBC配置样例**

```
<!--在beans元素下配置一个数据源 -->
<bean id="mysqlDataSource" class="org.apache.commons.dbcp2.BasicDataSource" destroy-method="close"> 
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>      
    <property name="url" value="jdbc:mysql://localhost:3306/test"/>      
    <property name="username" value="root"/>     
     <property name="password" value="1234"/>   
</bean>
 
<!-- 使用jdbc存储 -->
<persistenceAdapter>
   <jdbcPersistenceAdapter dataSource="#mysqlDataSource" createTablesOnStartup="true" /> 
</persistenceAdapter>
```

### levelDB存储和复制的LevelDB存储

两种持久化方式官网已经提示被弃用，官方不支持或建议使用，推荐使用kahaDB，这里不做详细配置说明，值得一提的是，公司曾经用过复制的LevelDB存储

### Memory 消息存储

基于内存的消息存储，消息存储在内存总，说白了就是没有进行持久化，直接存到内存中，配置是在 <broker>标签加上`persistent="false"`