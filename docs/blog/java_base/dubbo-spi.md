# Dubbo SPI设计详解

## Dubbo SPI核心实现

- 介绍
- ExtensionFactory接口
- ExtensionLoader<T> SPI核心实现
- Dubbo版本

## Dubbo SPI实现

&nbsp; &nbsp; 由于java提供的SPI实现不足，所以Dubbo自己实现了一套SPI，Dubbo的SPI的实现是调用的时候将实现了SPI接口的扩展类加载到缓存中，下面就简单的看一下Dubbo SPI实现的核心源码

### ExtensionFactory接口

&nbsp; &nbsp; `ExtensionFactory`接口，提供接入到SPI工厂的核心接口
```
@SPI
public interface ExtensionFactory {

    /**
     * Get extension.
     *
     * @param type object type.
     * @param name object name.
     * @return object instance.
     */
    <T> T getExtension(Class<T> type, String name);

}
```

### ExtensionLoader Dubbo SPI核心实现类

&nbsp; &nbsp; 开头提到了Dubbo的SPI是在程序调用过程中才将扩展接口实现加载进来，这其实是`懒加载`的一种实现方式，实现代码如下：

```
    // 扩展实现缓存容器
    private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<>(64);


    // 私有化构造器
    private ExtensionLoader(Class<?> type) {
        this.type = type;
        objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
    }
    // 核心SPI机制
    @SuppressWarnings("unchecked")
    public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        if (type == null) {
            throw new IllegalArgumentException("Extension type == null");
        }
        if (!type.isInterface()) {
            throw new IllegalArgumentException("Extension type (" + type + ") is not an interface!");
        }
        if (!withExtensionAnnotation(type)) {
            throw new IllegalArgumentException("Extension type (" + type +
                    ") is not an extension, because it is NOT annotated with @" + SPI.class.getSimpleName() + "!");
        }
        // 从缓存中获取扩展实现
        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
            // 将扩展实现加到缓存
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }
```