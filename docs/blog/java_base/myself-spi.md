# 自定义SPI实现 <!-- {docsify-ignore-all} -->


&nbsp; &nbsp; 参考`dubbo`和`shenyu网关`实现自定义的SPI


## SPI标注注解

&nbsp; &nbsp; 标注提供`SPI`能力接口的注解

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface SPI {

    /**
     * value
     * @return value
     */
    String value() default "";
}
```

&nbsp; &nbsp; 标准`SPI`实现的注解`@Join`

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Join {
}
```

## SPI核心实现

#### SPI的一些Class和扩展对象缓存

&nbsp; &nbsp; `SPI`实现是一个懒加载的过程，只有当通过`get`方法获取扩展的实例时才会加载扩展，并创建扩展实例，这里我们定义一个集合用于缓存扩展类，扩展对象等，代码如下：


```java
@Slf4j
@SuppressWarnings("all")
public class ExtensionLoader<T> {

    /**
     * SPI配置扩展的文件位置
     * 扩展文件命名格式为 SPI接口的全路径名，如：com.redick.spi.test.TestSPI
     */
    private static final String DEFAULT_DIRECTORY = "META-INF/log-helper/";

    /**
     * 扩展接口 {@link Class}
     */
    private final Class<T> tClass;

    /**
     * 扩展接口 和 扩展加载器 {@link ExtensionLoader} 的缓存
     */
    private static final Map<Class<?>, ExtensionLoader<?>> MAP = new ConcurrentHashMap<>();

    /**
     * 保存 "扩展" 实现的 {@link Class}
     */
    private final Holder<Map<String, Class<?>>> cachedClasses = new Holder<>();

    /**
     * "扩展名" 对应的 保存扩展对象的Holder的缓存
     */
    private final Map<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<>();

    /**
     * 扩展class 和 扩展点的实现对象的缓存
     */
    private final Map<Class<?>, Object> joinInstances = new ConcurrentHashMap<>();

    /**
     * 扩展点默认的 "名称" 缓存
     */
    private String cacheDefaultName;

    // 省略代码后面介绍
}
```

#### 获取扩展器ExtensionLoader

```java
    public static<T> ExtensionLoader<T> getExtensionLoader(final Class<T> tClass) {
        // 参数非空校验
        if (null == tClass) {
            throw new NullPointerException("tClass is null !");
        }
        // 参数应该是接口
        if (!tClass.isInterface()) {
            throw new IllegalArgumentException("tClass :" + tClass + " is not interface !");
        }
        // 参数要包含@SPI注解
        if (!tClass.isAnnotationPresent(SPI.class)) {
            throw new IllegalArgumentException("tClass " + tClass + "without @" + SPI.class + " Annotation !");
        }
        // 从缓存中获取扩展加载器，如果存在直接返回，如果不存在就创建一个扩展加载器并放到缓存中
        ExtensionLoader<T> extensionLoader = (ExtensionLoader<T>) MAP.get(tClass);
        if (null != extensionLoader) {
            return extensionLoader;
        }
        MAP.putIfAbsent(tClass, new ExtensionLoader<>(tClass));
        return (ExtensionLoader<T>) MAP.get(tClass);
    }
```

#### 扩展加载器构造方法

```java
    public ExtensionLoader(final Class<T> tClass) {
        this.tClass = tClass;
    }
```

#### 获取SPI扩展对象

&nbsp; &nbsp; 获取SPI扩展对象是懒加载过程，第一次去获取的时候是没有的，会触发从问家中加载资源，通过反射创建对象，并缓存起来。

```java
    public T getJoin(String cacheDefaultName) {
        // 扩展名 文件中的key
        if (StringUtils.isBlank(cacheDefaultName)) {
            throw new IllegalArgumentException("join name is null");
        }
        // 扩展对象存储缓存
        Holder<Object> objectHolder = cachedInstances.get(cacheDefaultName);
        // 如果扩展对象的存储是空的，创建一个扩展对象存储并缓存
        if (null == objectHolder) {
            cachedInstances.putIfAbsent(cacheDefaultName, new Holder<>());
            objectHolder = cachedInstances.get(cacheDefaultName);
        }
        // 从扩展对象的存储中获取扩展对象
        Object value = objectHolder.getT();
        // 如果对象是空的，就触发创建扩展，否则直接返回扩展对象
        if (null == value) {
            synchronized (cacheDefaultName) {
                value = objectHolder.getT();
                if (null == value) {
                    // 创建扩展对象
                    value = createExtension(cacheDefaultName);
                    objectHolder.setT(value);
                }
            }
        }
        return (T) value;
    }
```

#### 创建扩展对象

&nbsp; &nbsp; 反射方式创建扩展对象的实例

```java
    private Object createExtension(String cacheDefaultName) {
        // 根据扩展名字获取扩展的Class，从Holder中获取 key-value缓存，然后根据名字从Map中获取扩展实现Class
        Class<?> aClass = getExtensionClasses().get(cacheDefaultName);
        if (null == aClass) {
            throw new IllegalArgumentException("extension class is null");
        }
        Object o = joinInstances.get(aClass);
        if (null == o) {
            try {
                // 创建扩展对象并放到缓存中
                joinInstances.putIfAbsent(aClass, aClass.newInstance());
                o = joinInstances.get(aClass);
            } catch (InstantiationException e) {
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            }
        }
        return o;
    }
```

#### 从Holder中获取获取扩展实现的Class集合

```java
    public Map<String, Class<?>> getExtensionClasses() {
        // 扩区SPI扩展实现的缓存，对应的就是扩展文件中的 key - value
        Map<String, Class<?>> classes = cachedClasses.getT();
        if (null == classes) {
            synchronized (cachedClasses) {
                classes = cachedClasses.getT();
                if (null == classes) {
                    // 加载扩展
                    classes = loadExtensionClass();
                    // 缓存扩展实现集合
                    cachedClasses.setT(classes);
                }
            }
        }
        return classes;
    }
```

#### 加载扩展实现Class

&nbsp; &nbsp; 加载扩展实现Class，就是从文件中获取扩展实现的Class，然后缓存起来

```java
    public Map<String, Class<?>> loadExtensionClass() {
        // 扩展接口tClass，必须包含SPI注解
        SPI annotation = tClass.getAnnotation(SPI.class);
        if (null != annotation) {
            String v = annotation.value();
            if (StringUtils.isNotBlank(v)) {
                // 如果有默认的扩展实现名，用默认的
                cacheDefaultName = v;
            }
        }
        Map<String, Class<?>> classes = new HashMap<>(16);
        // 从文件加载
        loadDirectory(classes);
        return classes;
    }

    private void loadDirectory(final Map<String, Class<?>> classes) {
        // 文件名
        String fileName = DEFAULT_DIRECTORY + tClass.getName();
        try {
            ClassLoader classLoader = ExtensionLoader.class.getClassLoader();
            // 读取配置文件
            Enumeration<URL> urls = classLoader != null ? classLoader.getResources(fileName)
                    : ClassLoader.getSystemResources(fileName);
            if (urls != null) {
                // 获取所有的配置文件
                while (urls.hasMoreElements()) {
                    URL url = urls.nextElement();
                    // 加载资源
                    loadResources(classes, url);
                }
            }
        } catch (IOException e) {
            log.error("load directory error {}", fileName, e);
        }
    }

    private void loadResources(Map<String, Class<?>> classes, URL url) {
        // 读取文件到Properties，遍历Properties，得到配置文件key（名字）和value（扩展实现的Class）
        try (InputStream inputStream = url.openStream()) {
            Properties properties = new Properties();
            properties.load(inputStream);
            properties.forEach((k, v) -> {
                // 扩展实现的名字
                String name = (String) k;
                // 扩展实现的Class的全路径
                String classPath = (String) v;
                if (StringUtils.isNotBlank(name) && StringUtils.isNotBlank(classPath)) {
                    try {
                        // 加载扩展实现Class，就是想其缓存起来，缓存到集合中
                        loadClass(classes, name, classPath);
                    } catch (ClassNotFoundException e) {
                        log.error("load class not found", e);
                    }
                }
            });
        } catch (IOException e) {
            log.error("load resouces error", e);
        }
    }

    private void loadClass(Map<String, Class<?>> classes, String name, String classPath) throws ClassNotFoundException {
        // 反射创建扩展实现的Class
        Class<?> subClass = Class.forName(classPath);
        // 扩展实现的Class要是扩展接口的实现类
        if (!tClass.isAssignableFrom(subClass)) {
            throw new IllegalArgumentException("load extension class error " + subClass + " not sub type of " + tClass);
        }
        // 扩展实现要有Join注解
        Join annotation = subClass.getAnnotation(Join.class);
        if (null == annotation) {
            throw new IllegalArgumentException("load extension class error " + subClass + " without @Join" +
                    "Annotation");
        }
        // 缓存扩展实现Class
        Class<?> oldClass = classes.get(name);
        if (oldClass == null) {
            classes.put(name, subClass);
        } else if (oldClass != subClass) {
            log.error("load extension class error, Duplicate class oldClass is " + oldClass + "subClass is" + subClass);
        }
    }
```

#### 存储Holder

```java
    public static class Holder<T> {

        private volatile T t;

        public T getT() {
            return t;
        }

        public void setT(T t) {
            this.t = t;
        }
    }
```

#### 测试SPI

- **定义SPI接口**

```java
@SPI
public interface TestSPI {

    void test();
}
```

- **扩展实现1和2**

```java
@Join
public class TestSPI1Impl implements TestSPI {

    @Override
    public void test() {
        System.out.println("test1");
    }
}

@Join
public class TestSPI2Impl implements TestSPI {

    @Override
    public void test() {
        System.out.println("test2");
    }
}
```

- **在resources文件夹下创建META-INF/log-helper文件夹，并创建扩展文件**

文件名称（接口全路径名）：com.redick.spi.test.TestSPI

文件内容

```properties
testSPI1=com.redick.spi.test.TestSPI1Impl
testSPI2=com.redick.spi.test.TestSPI2Impl
```

- **动态使用测试**

```java
public class SpiExtensionFactoryTest {

    @Test
    public void getExtensionTest() {
        TestSPI testSPI = ExtensionLoader.getExtensionLoader(TestSPI.class).getJoin("testSPI1");
        testSPI.test();
    }
}

测试结果：

test1
```

```java
public class SpiExtensionFactoryTest {

    @Test
    public void getExtensionTest() {
        TestSPI testSPI = ExtensionLoader.getExtensionLoader(TestSPI.class).getJoin("testSPI2");
        testSPI.test();
    }
}

测试结果：

test2
```

## 总结

&nbsp; &nbsp; 实现一个自定义的SPI机制其核心的逻辑就是扩展的加载，本篇是参考Dubbo等开源项目简单实现了一个SPI机制的核心代码，核心逻辑就是从SPI扩展的配置文件中加载扩展实现的流程，通常情况下，SPI的应用场景出现在高度可扩展组件，并且在使用过程中有需求能够灵活切换不同的实现的时候。比如程序使用限流组件，使用“令牌桶算法”和“漏桶算法”分别实现了限流逻辑，在业务使用限流算法的过程中，就可以通过SPI机制在程序启动过程中将两种算法实现的组件加载好，然后通过参数指定具体使用的限流算法。此外，SPI机制能够对扩展开放，常用于开源软件，用户可以实现自己的扩展。