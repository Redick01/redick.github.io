# Quarkus - MySQL Hibernate ORM + 多数据源 <!-- {docsify-ignore-all} -->


## 集成Hibernate ORM

### POM

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>quarkus-project-parent</artifactId>
        <groupId>io.redick.quarkus</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>quarkus-project-hibernate</artifactId>

    <properties>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>${quarkus.platform.group-id}</groupId>
                <artifactId>${quarkus.platform.artifact-id}</artifactId>
                <version>${quarkus.platform.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-resteasy-reactive</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-resteasy-reactive-jackson</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkiverse.loggingmanager</groupId>
            <artifactId>quarkus-logging-manager</artifactId>
            <version>2.1.3</version>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-logging-json</artifactId>
        </dependency>
        <!-- Hibernate ORM specific dependencies -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-hibernate-orm</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-jdbc-mysql</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-smallrye-openapi</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.rest-assured</groupId>
            <artifactId>rest-assured</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>io.redick.quarkus</groupId>
            <artifactId>quarkus-project-common</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>${quarkus.platform.group-id}</groupId>
                <artifactId>quarkus-maven-plugin</artifactId>
                <version>${quarkus.platform.version}</version>
                <extensions>true</extensions>
                <executions>
                    <execution>
                        <goals>
                            <goal>build</goal>
                            <goal>generate-code</goal>
                            <goal>generate-code-tests</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>${compiler-plugin.version}</version>
                <configuration>
                    <compilerArgs>
                        <arg>-parameters</arg>
                    </compilerArgs>
                </configuration>
            </plugin>
            <plugin>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>${surefire-plugin.version}</version>
                <configuration>
                    <systemPropertyVariables>
                        <java.util.logging.manager>org.jboss.logmanager.LogManager</java.util.logging.manager>
                        <maven.home>${maven.home}</maven.home>
                    </systemPropertyVariables>
                </configuration>
            </plugin>
            <plugin>
                <artifactId>maven-failsafe-plugin</artifactId>
                <version>${surefire-plugin.version}</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>integration-test</goal>
                            <goal>verify</goal>
                        </goals>
                        <configuration>
                            <systemPropertyVariables>
                                <native.image.path>${project.build.directory}/${project.build.finalName}-runner
                                </native.image.path>
                                <java.util.logging.manager>org.jboss.logmanager.LogManager</java.util.logging.manager>
                                <maven.home>${maven.home}</maven.home>
                            </systemPropertyVariables>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
    <profiles>
        <profile>
            <id>native</id>
            <activation>
                <property>
                    <name>native</name>
                </property>
            </activation>
            <properties>
                <skipITs>false</skipITs>
                <quarkus.package.type>native</quarkus.package.type>
            </properties>
        </profile>
    </profiles>
</project>
```

### application.properties配置

支持了多数据源配置

```properties
quarkus.http.port=8093
quarkus.application.name=quarkus-hibernate-demo
quarkus.application.version=1.0

quarkus.log.level=INFO
quarkus.log.min-level=DEBUG

# 数据库1
quarkus.datasource."bank01".db-kind=mysql
quarkus.datasource."bank01".jdbc.url=jdbc:mysql://localhost:3316/tx-tcc-bank01
quarkus.datasource."bank01".username=root
quarkus.datasource."bank01".password=admin123
quarkus.datasource."bank01".jdbc.max-size=8
quarkus.datasource."bank01".jdbc.min-size=2

quarkus.hibernate-orm."bank01".log.sql=true
quarkus.hibernate-orm."bank01".datasource=bank01
quarkus.hibernate-orm."bank01".packages=io.redick.quarkus.hibernate.db.bank

# 数据库2
quarkus.datasource.stock.db-kind=mysql
quarkus.datasource.stock.jdbc.url=jdbc:mysql://localhost:3316/tx-stock
quarkus.datasource.stock.username=root
quarkus.datasource.stock.password=admin123
quarkus.datasource.stock.jdbc.max-size=8
quarkus.datasource.stock.jdbc.min-size=2

quarkus.hibernate-orm.stock.log.sql=true
quarkus.hibernate-orm.stock.datasource=stock
quarkus.hibernate-orm.stock.packages=io.redick.quarkus.hibernate.db.stock
```

### ORM映射关系编码

创建`Stock`类对应数据库2的stock表

```java
@Entity
@Table(name = "stock")
@NamedQuery(name = "Stock.findAll",
        query = "SELECT f FROM Stock f",
        hints = @QueryHint(name = "org.hibernate.cacheable", value = "true") )
@Cacheable
@PersistenceUnit(name = "stock")
public class Stock {

    @Id
    @Column(name = "id", length = 20, unique = true)
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "product_id", length = 11)
    private Long productId;

    @Column(name = "total_count", length = 11)
    private Integer totalCount;

    @Column(name = "create_time")
    private Date createTime;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public Long getProductId() {
        return productId;
    }

    public void setProductId(Long productId) {
        this.productId = productId;
    }

    public Integer getTotalCount() {
        return totalCount;
    }

    public void setTotalCount(Integer totalCount) {
        this.totalCount = totalCount;
    }

    public Date getCreateTime() {
        return createTime;
    }

    public void setCreateTime(Date createTime) {
        this.createTime = createTime;
    }

    @Override
    public String toString() {
        return "Stock{" +
                "id=" + id +
                ", productId=" + productId +
                ", totalCount=" + totalCount +
                ", createTime=" + createTime +
                '}';
    }
}
```

`StockService`操作数据库的Service

```java
@ApplicationScoped
public class StockService {

    @Inject
    @Named("stock")
    EntityManager manager;

    @Transactional(rollbackOn = Exception.class)
    public void createUserCount() {
        Stock stock = new Stock();
        stock.setCreateTime(new Date());
        stock.setTotalCount(10000);
        stock.setProductId(2L);
        manager.persist(stock);
    }

    public List<Stock> findAll() {
        return manager.createNamedQuery("Stock.findAll", Stock.class).getResultList();
    }
}
```

创建`UserAccount`类对应数据库1的user_account表

```java
@Entity
@Table(name = "user_account")
@NamedQuery(name = "UserAccount.findAll",
        query = "SELECT f FROM UserAccount f",
        hints = @QueryHint(name = "org.hibernate.cacheable", value = "true") )
@Cacheable
@PersistenceUnit(name = "bank01")
public class UserAccount {

    @Id
    @Column(name = "id", length = 11, unique = true)
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    @Column(name = "account_no", length = 64, unique = true)
    private String accountNo;

    @Column(name = "account_name", length = 50)
    private String accountName;

    @Column(name = "account_balance", length = 10)
    private Double accountBalance;

    @Column(name = "transform_balance", length = 10)
    private Double transformBalance;

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public String getAccountNo() {
        return accountNo;
    }

    public void setAccountNo(String accountNo) {
        this.accountNo = accountNo;
    }

    public String getAccountName() {
        return accountName;
    }

    public void setAccountName(String accountName) {
        this.accountName = accountName;
    }

    public Double getAccountBalance() {
        return accountBalance;
    }

    public void setAccountBalance(Double accountBalance) {
        this.accountBalance = accountBalance;
    }

    public Double getTransformBalance() {
        return transformBalance;
    }

    public void setTransformBalance(Double transformBalance) {
        this.transformBalance = transformBalance;
    }

    @Override
    public String toString() {
        return "UserCount{" +
                "id=" + id +
                ", accountNo='" + accountNo + '\'' +
                ", accountName='" + accountName + '\'' +
                ", accountBalance='" + accountBalance + '\'' +
                ", transformBalance='" + transformBalance + '\'' +
                '}';
    }
}
```

`UserAccountService`操作数据库的Service

```java
@Singleton
public class UserAccountService {

    @Inject
    @Named("bank01")
    EntityManager manager;

    @Transactional(rollbackOn = Exception.class)
    public void createUserCount() {
        UserAccount userCount = new UserAccount();
        userCount.setAccountName("刘p辉");
        userCount.setAccountNo("1003");
        userCount.setTransformBalance(100.0);
        userCount.setAccountBalance(1000.0);
        manager.persist(userCount);
    }

    public List<UserAccount> getUserCount() {
        return manager.createNamedQuery("UserAccount.findAll", UserAccount.class).getResultList();
    }
}
```

### HTTP接口Resource的编写

- StockResource

```java
@Path("/stock")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class StockResource {

    private static final Logger log = LoggerFactory.getLogger(UserAccountResource.class);

    @Inject
    StockService stockService;

    @GET
    @Path("/getAll")
    @Logged
    public Response getAll() {
        HttpResponseDTO<Object> responseDTO = HttpResponseDTO
                .builder()
                .resCode("0000")
                .resMessage("成功")
                .resData(stockService.findAll())
                .build();
        return Response.ok(responseDTO).build();
    }

    @POST
    @Path("/create")
    @Logged
    public Response createUserCount() {
        stockService.createUserCount();
        HttpResponseDTO<Object> responseDTO = HttpResponseDTO
                .builder()
                .resCode("0000")
                .resMessage("成功")
                .resData(stockService.findAll())
                .build();
        return Response.ok(responseDTO).build();
    }
}
```

- UserAccountResource

```java
@Path("/account")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class UserAccountResource {

    private static final Logger log = LoggerFactory.getLogger(UserAccountResource.class);

    @Inject
    UserAccountService userCountService;

    @GET
    @Path("/getAll")
    @Logged
    public Response order() {

        HttpResponseDTO<Object> responseDTO = HttpResponseDTO
                .builder()
                .resCode("0000")
                .resMessage("成功")
                .resData(userCountService.getUserCount())
                .build();
        return Response.ok(responseDTO).build();
    }

    @POST
    @Path("/create")
    @Logged
    public Response createUserCount() {
        userCountService.createUserCount();
        HttpResponseDTO<Object> responseDTO = HttpResponseDTO
                .builder()
                .resCode("0000")
                .resMessage("成功")
                .resData(userCountService.getUserCount())
                .build();
        return Response.ok(responseDTO).build();
    }
}
```


[详细参考官网](https://cn.quarkus.io/guides/hibernate-orm)