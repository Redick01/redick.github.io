# JMH - Java微基准测试工具套件 <!-- {docsify-ignore-all} -->

## JMH简介

&nbsp; &nbsp; JMH 是 OpenJDK 团队开发的一款基准测试工具，一般用于代码的性能调优，精度甚至可以达到纳秒级别，适用于 java 以及其他基于 JVM 的语言。和 Apache JMeter 不同，`JMH 测试的对象可以是任一方法，颗粒度更小，而不仅限于rest api`。

JMH 比较典型的应用场景如下：

1. 想准确地知道某个方法需要执行多长时间，以及执行时间和输入之间的相关性
2. 对比接口不同实现在给定条件下的吞吐量
3. 查看多少百分比的请求在多长时间内完成

## JMH基础及常用注解说明

&nbsp; &nbsp; JMH主要是通过注解的形式编写测试单元，告诉JMH如何测试，JMH自动生成测试代码，所以在使用JMH进行微基准测试时一定要先了对JMH注解有一定了解，下面就介绍下JMH的注解。


### @Benchmark

&nbsp; &nbsp; `@Benchmark`用于告诉`JMH`哪些方法需要进行测试，只能注解在方法上，有点类似`junit`的`@Test`。在测试项目进行`package`时，`JMH`会针对注解了`@Benchmark`的方法生成`Benchmark`方法代码。通常情况下，每个`Benchmark`方法都运行在独立的进程中，互不干涉。

### @BenchmarkMode

`@BenchmarkMode`用于指定当前`Benchmark`方法使用哪种模式测试。`JMH`提供了4种不同的模式，用于输出不同的结果指标，如下：

1. Throughput：吞吐量，ops/time。单位时间内执行操作的平均次数
2. AverageTime：每次操作所需时间，time/op。执行每次操作所需的平均时间
3. SampleTime：同 AverageTime，区别在于 SampleTime会随机取样，最后输出取样结果的分布
4. SingleShotTime：同 AverageTime。区别在于 SingleShotTime 只执行一次操作。这种模式的结果存在较大随机性。
5. All：上边所有的都执行一遍

@BenchmarkMode支持数组，可以指定多种模式也可以配置All，所有模式都执行一遍

### @Warmup和@Measurement

&nbsp; &nbsp; `@Warmup`和`@Measurement`分别用于配置预热迭代和测试迭代。其中，`iterations`用于指定迭代次数，`time`和`timeUnit`用于每个迭代的时间，`batchSize`表示执行多少次`Benchmark`方法为一个`invocation`。

### @State

&nbsp; &nbsp; 通过 State 可以指定一个对象的作用范围，JMH 根据 scope 来进行实例化和共享操作。@State 可以被继承使用，如果父类定义了该注解，子类则无需定义。由于 JMH 允许多线程同时执行测试，不同的选项含义如下：

1. Scope.Benchmark：所有测试线程共享一个实例，测试有状态实例在多线程共享下的性能
2. Scope.Group：同一个线程在同一个 group 里共享实例
3. Scope.Thread：默认的 State，每个测试线程分配一个实例

### @Setup 和 @TearDown

这两个注解只能定义在注解了 State 里，其中，@Setup类似于 junit 的@Before，而@TearDown类似于 junit 的@After。

```java
@State(Scope.Thread)
public class JMHSample_05_StateFixtures {

    double x;

    @Setup(Level.Iteration)
    public void prepare() {
        System.err.println("init............");
        x = Math.PI;
    }

    @TearDown(Level.Iteration)
    public void check() {
        System.err.println("destroy............");
        assert x > Math.PI : "Nothing changed?";
    }

    @Benchmark
    public void measureRight() {
        x++;
    }
}
```

这两个注解注释的方法的调用时机，主要受`Level`的控制，JMH提供了三种`Level`，如下：

1. Trial：Benchmark 开始前或结束后执行，如下。Level 为 Benchmark 的 Setup 和 TearDown 方法的开销不会计入到最终结果。
2. Iteration：Benchmark 里每个 Iteration 开始前或结束后执行，如下。Level 为 Iteration 的 Setup 和 TearDown 方法的开销不会计入到最终结果。
3. Invocation：Iteration 里每次方法调用开始前或结束后执行，如下。`Level 为 Invocation 的 Setup 和 TearDown 方法的开销将计入到最终结果`。

### @OutputTimeUnit

为统计结果的时间单位，可用于类或者方法注解


### @Threads

每个进程中的测试线程，可用于类或者方法上。

### @Fork

进行 fork 的次数，可用于类或者方法上。如果 fork 数是 2 的话，则 JMH 会 fork 出两个进程来进行测试。


## JMH 陷阱

在使用 JMH 的过程中，一定要避免一些陷阱。

比如 JIT 优化中的死码消除，比如以下代码：

```java
@Benchmark
public void testStringAdd(Blackhole blackhole) {
    String a = "";
    for (int i = 0; i < length; i++) {
        a += i;
    }
}
```

JVM 可能会认为变量 a 从来没有使用过，从而进行优化把整个方法内部代码移除掉，这就会影响测试结果。JMH 提供了两种方式避免这种问题，一种是将这个变量作为方法返回值 return a，一种是通过 Blackhole 的 consume 来避免 JIT 的优化消除。其他陷阱还有常量折叠与常量传播、永远不要在测试中写循环、使用 Fork 隔离多个测试方法、方法内联、伪共享与缓存行、分支预测、多线程测试等

## IDEA插件

`JMH plugin`插件可以让我们像使用Junit一样使用JMH。

这个插件可以让我们能够以 JUnit 相同的方式使用 JMH，主要功能如下：自动生成带有 @Benchmark 的方法像 JUnit 一样，运行单独的 Benchmark 方法运行类中所有的 Benchmark 方法比如可以通过右键点击 Generate...，选择操作 Generate JMH benchmark 就可以生成一个带有 @Benchmark 的方法。还有将光标移动到方法声明并调用 Run 操作就运行一个单独的 Benchmark 方法。将光标移到类名所在行，右键点击 Run 运行，该类下的所有被 @Benchmark 注解的方法都会被执行。

## 集成使用

### maven依赖和插件配置

```xml
        <!-- JMH的核心包 https://mvnrepository.com/artifact/org.openjdk.jmh/jmh-core -->
        <dependency>
            <groupId>org.openjdk.jmh</groupId>
            <artifactId>jmh-core</artifactId>
            <version>1.21</version>
        </dependency>

        <!-- JMH依赖注解,需要注解处理包 https://mvnrepository.com/artifact/org.openjdk.jmh/jmh-generator-annprocess -->
        <dependency>
            <groupId>org.openjdk.jmh</groupId>
            <artifactId>jmh-generator-annprocess</artifactId>
            <version>1.21</version>
            <scope>test</scope>
        </dependency>
```

### 编写基准测试

```java
@BenchmarkMode(Mode.AverageTime)
@Warmup(iterations = 3, time = 1)
@Measurement(iterations = 5, time = 5)
@Threads(4)
@Fork(1)
@State(value = Scope.Benchmark)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
public class StringConnectTest {

    @Param(value = {"10", "50", "100"})
    private int length;

    @Benchmark
    public void testStringAdd(Blackhole blackhole) {
        String a = "";
        for (int i = 0; i < length; i++) {
            a += i;
        }
        blackhole.consume(a);
    }

    @Benchmark
    public void testStringBuilderAdd(Blackhole blackhole) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < length; i++) {
            sb.append(i);
        }
        blackhole.consume(sb.toString());
    }

    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(StringConnectTest.class.getSimpleName())
                .result("result.json")
                .resultFormat(ResultFormatType.JSON).build();
        new Runner(opt).run();
    }
}
```

### 生成 jar 包执行

JMH 官方提供了生成 jar 包的方式来执行，我们需要在 maven 里增加一个 plugin，具体配置如下：

```xml
<plugins>
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-shade-plugin</artifactId>
        <version>2.4.1</version>
        <executions>
            <execution>
                <phase>package</phase>
                <goals>
                    <goal>shade</goal>
                </goals>
                <configuration>
                    <finalName>jmh-demo</finalName>
                    <transformers>
                        <transformer
                                implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                            <mainClass>org.openjdk.jmh.Main</mainClass>
                        </transformer>
                    </transformers>
                </configuration>
            </execution>
        </executions>
    </plugin>
</plugins>
```

运行基准测试

```powershell
mvn clean install
java -jar target/jmh-demo.jar StringConnectTest
```

## 可视化

- JMH Visual Chart：http://deepoove.com/jmh-visual-chart/
- JMH Visualizer：https://jmh.morethan.io/