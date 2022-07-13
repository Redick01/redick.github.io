# Quarkus - 容错 <!-- {docsify-ignore-all} -->


微服务的分布式特性带来的挑战之一是与外部系统的通信本质上是不可靠的。这增加了对应用程序弹性的需求。为了简化制作更具弹性的应用程序，Quarkus 为SmallRye Fault Tolerance提供了MicroProfile Fault Tolerance规范的实现。

在本指南中，我们演示了 MicroProfile Fault Tolerance 注释（例如@Timeout、@Fallback和.@Retry@CircuitBreaker


## 创建项目

```shell
mvn io.quarkus.platform:quarkus-maven-plugin:2.10.2.Final:create \
    -DprojectGroupId=org.acme \
    -DprojectArtifactId=microprofile-fault-tolerance-quickstart \
    -Dextensions="smallrye-fault-tolerance,resteasy-reactive-jackson" \
    -DnoCode
```

如果你已经有`Quarkus`项目，那么添加如下的扩展即可：

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-smallrye-fault-tolerance</artifactId>
</dependency>
```

或

maven：

```powershell
./mvnw quarkus:add-extension -Dextensions="smallrye-fault-tolerance"
```

或

命令行：

```powershell
quarkus extension add 'smallrye-fault-tolerance'
```

## demo编码

### Fruit类

```java
public class Fruit {

    private String name;

    private String description;

    public Fruit() {}

    public Fruit(String name, String description) {
        this.name = name;
        this.description = description;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getDescription() {
        return description;
    }

    public void setDescription(String description) {
        this.description = description;
    }

    @Override
    public String toString() {
        return "Fruit{" +
                "name='" + name + '\'' +
                ", description='" + description + '\'' +
                '}';
    }
}
```

### 重试Retry

添加重试之前会随机的报错，在添加`@Retry`之后请求都会成功不会报错，实际上后台还是会报错，只不过系统自动重试了。

```java
@Path("/fruits")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@Slf4j
public class FruitResource {

    public List<Fruit> fruits = new ArrayList<>();

    private AtomicLong counter = new AtomicLong(0);

    public FruitResource() {
        fruits.add(new Fruit("Apple", "苹果"));
        fruits.add(new Fruit("Orange", "桔子"));
    }


    @GET
    @Path("retry")
    @Retry(maxRetries = 4)
    public List<Fruit> retry() {
        maybeFail();
        return fruits;
    }

    private void maybeFail() {
        if (new Random().nextBoolean()) {
            log.error("failure");
            throw new RuntimeException("Resource failure.");
        }
    }
}    
```

### 超时+降级

`randomDelay()`方法会随机睡0~500ms，当超过250ms时`/delay`接口报超时错误，在添加`@Fallback`注解后当产生超时后就会执行降级方法`fallback()`。

```java
@Path("/fruits")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@Slf4j
public class FruitResource {

    public List<Fruit> fruits = new ArrayList<>();

    private AtomicLong counter = new AtomicLong(0);

    public FruitResource() {
        fruits.add(new Fruit("Apple", "苹果"));
        fruits.add(new Fruit("Orange", "桔子"));
    }

    @GET
    @Path("/delay")
    @Timeout(250)
    @Fallback(fallbackMethod = "fallback")
    public List<Fruit> delay() throws InterruptedException {
        randomDelay();
        return fruits;
    }

    private void randomDelay() throws InterruptedException {
        Thread.sleep(new Random().nextInt(500));
    }

    private List<Fruit> fallback() {
        return Collections.singletonList(new Fruit("watermelon", "西瓜"));
    }
}
```

### 断路器

`invocationNumber % 4 > 1`会导致每隔两次请求就触发抛异常，抛出两次异常后，断路器打开5s，这里要注意容错相关的注解修饰的方法应该是`public`的。

```java
@Path("/fruits")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@Slf4j
public class FruitResource {

    public List<Fruit> fruits = new ArrayList<>();

    private AtomicLong counter = new AtomicLong(0);

    public FruitResource() {
        fruits.add(new Fruit("Apple", "苹果"));
        fruits.add(new Fruit("Orange", "桔子"));
    }

    @GET
    @Path("availability")
    public Response availability() {
        try {
            Integer i = getAvailability();
            return Response.ok(i).build();
        } catch (RuntimeException e) {
            String message = e.getClass().getSimpleName() + ": " + e.getMessage();
            return Response.status(Response.Status.INTERNAL_SERVER_ERROR)
                    .entity(message)
                    .type(MediaType.TEXT_PLAIN_TYPE)
                    .build();
        }
    }

    @CircuitBreaker(requestVolumeThreshold = 4)
    public Integer getAvailability() {
        maybeFailBreaker();
        return new Random().nextInt(30);
    }

    private void maybeFailBreaker() {
        final long invocationNumber = counter.getAndIncrement();
        // alternate 2 successful and 2 failing invocations
        if (invocationNumber % 4 > 1) {
            throw new RuntimeException("Service failed.");
        }
    }
}
```