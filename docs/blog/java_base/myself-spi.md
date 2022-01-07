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
        // 根据扩展名字获取扩展的Class
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
