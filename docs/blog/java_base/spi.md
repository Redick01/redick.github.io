# Java SPI机制详解 <!-- {docsify-ignore-all} -->

## SPI介绍

&nbsp; &nbsp; SPI ，全称为 Service Provider Interface，是一种服务发现机制，是Java提供的一套用来被第三方实现或者扩展的接口，它可以用来启用框架扩展和替换组件。 SPI的作用就是为这些被扩展的API寻找服务实现。<<高可用可伸缩微服务架构>> 第3章 Apache Dubbo 框架的原理与实现 中有这样的一句定义.SPI是 JDK 内置的一种服务提供发现功能, 一种动态替换发现的机制. 举个例子, 要想在运行时动态地给一个接口添加实现, 只需要添加一个实现即可.

## Java SPI的实现规范

![avatar](../../../_media/image/structure/javaspi.jpg) 


&nbsp; &nbsp; 也就是说在我们代码中的实现里, 无需去写入一个 Factory 工厂, 用 MAP 去包装一些子类, 最终返回的类型是父接口. 只需要定义好资源文件, 让父接口与它的子类在文件中写明, 即可通过设置好的方式拿到所有定义的子类对象:

```java
ServiceLoader<Interface> loaders = ServiceLoader.load(Interface.class)
for(Interface interface : loaders){
	System.out.println(interface.toString());
}
```
这种方式相比与普通的工厂模式, 肯定是更符合开闭原则, 新加入一个子类不用去修改工厂方法, 而是编辑资源文件.

- **按照Java SPI规范实现SPI**

编写SPI接口和实现类
```java
public interface Fruit {

    /**
     * name
     * @return name
     */
    String name();
}

public class Apple implements Fruit {

    @Override
    public String name() {
        return "Apple";
    }
}

public class Orange implements Fruit {

    @Override
    public String name() {
        return "Orange";
    }
}
```

`resource`目录下创`META-INF/services/`目录，并在该目录下创建以SPI接口全路径名的文件`com.test.spi.Fruit`，文件内容如下：
```
com.test.spi.Apple
com.test.spi.Orange
```

编写SPI测试类

```java
public class FruitTest {

    @Test
    public void testFruit() {
        ServiceLoader<Fruit> serviceLoader = ServiceLoader.load(Fruit.class);
        for (Fruit fruit : serviceLoader) {
            System.out.println(fruit.name());
        }
    }
}
```

项目结构如下：

![avatar](../../../_media/image/java/spi/project-1.jpg) 

## Java SPI源码分析

- **ServiceLoader类SPI配置文件路径**

```java
public final class ServiceLoader<S>
    implements Iterable<S> {
    private static final String PREFIX = "META-INF/services/";
}
```

- **ServiceLoader类SPI核心实现**

``` java
    public static <S> ServiceLoader<S> load(Class<S> service) {
        // 获取当前线程的ClassLoader
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return ServiceLoader.load(service, cl);
    }

    public static <S> ServiceLoader<S> load(Class<S> service,
                                            ClassLoader loader)
    {
        return new ServiceLoader<>(service, loader);
    }

    private ServiceLoader(Class<S> svc, ClassLoader cl) {
        // 初始化ServiceLoader属性
        service = Objects.requireNonNull(svc, "Service interface cannot be null");
        loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
        acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
        // 加载
        reload();
    }
     
    public void reload() {
        providers.clear();
        // 创建懒加载的迭代器
        lookupIterator = new LazyIterator(service, loader);
    }
```

- **LazyIterator类反射创建对象过程分析**

`LazyIterator`实现了`Iterator`，所以是在遍历迭代器时候通过反射创建的对象

```java
    private class LazyIterator
        implements Iterator<S> {
        // 接口全路径
        Class<S> service;
        // 类加载器
        ClassLoader loader;
        // 配置文件对象
        Enumeration<URL> configs = null;
        // 迭代器
        Iterator<String> pending = null;
        // SPI实现类全路径名
        String nextName = null;

        private LazyIterator(Class<S> service, ClassLoader loader) {
            this.service = service;
            this.loader = loader;
        }

        private boolean hasNextService() {
            if (nextName != null) {
                return true;
            }
            if (configs == null) {
                try {
                    // 文件全路径名
                    String fullName = PREFIX + service.getName();
                    if (loader == null)
                        configs = ClassLoader.getSystemResources(fullName);
                    else
                        configs = loader.getResources(fullName);
                } catch (IOException x) {
                    fail(service, "Error locating configuration files", x);
                }
            }
            while ((pending == null) || !pending.hasNext()) {
                if (!configs.hasMoreElements()) {
                    return false;
                }
                // 获取SPI类的全路径名
                pending = parse(service, configs.nextElement());
            }
            nextName = pending.next();
            return true;
        }
        // 反射创建对象
        private S nextService() {
            if (!hasNextService())
                throw new NoSuchElementException();
            String cn = nextName;
            nextName = null;
            Class<?> c = null;
            try {
                // 制定类加载器，反射创建对象
                c = Class.forName(cn, false, loader);
            } catch (ClassNotFoundException x) {
                fail(service,
                     "Provider " + cn + " not found");
            }
            if (!service.isAssignableFrom(c)) {
                fail(service,
                     "Provider " + cn  + " not a subtype");
            }
            try {
                // 初始化对象, 并判断是否与接口符合
                S p = service.cast(c.newInstance());
                // 将初始化的对象放入缓存中
                providers.put(cn, p);
                return p;
            } catch (Throwable x) {
                fail(service,
                     "Provider " + cn + " could not be instantiated",
                     x);
            }
            throw new Error();          // This cannot happen
        }
    }
```

## 总结

Java的SPI实现的主流程如下：
- 使用IO流读取配置资源文件
- 实现迭代器接口和懒加载的模式在遍历阶段创建对象
- 使用反射创建对象并放到缓存中

下节我们参考dubbo来看下dubbo的SPI是怎么实现的