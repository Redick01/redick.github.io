# Spring Cloud Alibaba-数据库操作 <!-- {docsify-ignore-all} -->



​    本篇开始介绍数据库操作的相关内容，技术选型选用`Mybatis-Plus`作为`ORM`框架，本篇会直接从动态数据源和读写分离开始介绍，直接跳过快速接入，如想了解单数据源快速接入，可自行网上搜索资料或`Mybatis-Plus`官网。



## pom依赖

```xml
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>${druid.version}</version>
        </dependency>

        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>${mybatis.plus.version}</version>
        </dependency>

        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>dynamic-datasource-spring-boot-starter</artifactId>
            <version>${dynamic.datasource.version}</version>
        </dependency>
```



## 动态数据源



#### application.yml配置

```yaml
spring:
  application:
    name: account-svc
  profiles:
    active: dev
  datasource:
    dynamic:
      #设置默认的数据源或者数据源组,默认值即为master
      primary: master
      #严格匹配数据源,默认false. true未匹配到指定数据源时抛异常,false使用默认数据源
      strict: false 
      datasource:
        master:
          url: jdbc:mysql://127.0.0.1:3316/ruuby-stock?useUnicode=true&characterEncoding=UTF8&statementInterceptors=com.redick.support.mysql.Mysql5StatementInterceptor
          username: root
          password: admin123
          driver-class-name: com.mysql.jdbc.Driver
          type: com.alibaba.druid.pool.DruidDataSource
          initialSize: 5
          minIdle: 5
          maxActive: 20
        slave1:
          url: jdbc:mysql://127.0.0.1:3326/ruuby-stock?useUnicode=true&characterEncoding=UTF8&statementInterceptors=com.redick.support.mysql.Mysql5StatementInterceptor
          username: root
          password: admin123
          driver-class-name: com.mysql.jdbc.Driver
          type: com.alibaba.druid.pool.DruidDataSource
          initialSize: 5
          minIdle: 5
          maxActive: 20
        slave2:
          url: jdbc:mysql://127.0.0.1:3326/ruuby-stock?useUnicode=true&characterEncoding=UTF8&statementInterceptors=com.redick.support.mysql.Mysql5StatementInterceptor
          username: root
          password: admin123
          driver-class-name: com.mysql.jdbc.Driver
          type: com.alibaba.druid.pool.DruidDataSource
          initialSize: 5
          minIdle: 5
          maxActive: 20
```

#### 数据源切换

​    使用`@DS`注解切换数据源，如：`@DS("master")`，伪代码如下：

```java
@Service
@DS("master")
public class StockServiceImpl extends ServiceImpl<StockMapper, Stock> implements StockService {

}
```

详细使用方式参考[Mybatis-Plus官方文档](https://www.baomidou.com/pages/a61e1b/#dynamic-datasource)