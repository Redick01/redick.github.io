# Dubbo SPI设计实现源码分析

## Dubbo SPI核心实现

- Dubbo SPI实现介绍
- ExtensionFactory接口
- ExtensionLoader<T> SPI核心实现
- 总结

## Dubbo SPI实现

&nbsp; &nbsp; 由于java提供的SPI实现不足，所以Dubbo自己实现了一套SPI，Dubbo的SPI的实现应用启动时候将实现了SPI接口的扩展类加载到缓存中，下面就简单的看一下Dubbo SPI实现的核心源码

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

### getExtensionLoader

&nbsp; &nbsp; 开头提到了Dubbo的SPI是在程序启动，服务注册过程中将扩展接口实现加载进来，首先通过私有构造器会调到`getExtensionLoader`方法，该方法主要逻辑如下：

1. `type`是否为空
2. `type` 是否是接口类型
3. `type`类是否带有`@SPI`注解
4. 验证通过后会从`EXTENSION_LOADERS`扩展实现的缓存中获取扩展实例
5. 代码中一开始是获取不到的，所以会走到`EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));`去初始化并缓存扩展实例。

```
    // 扩展实现缓存容器
    private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<>(64);

    // 获取SPI扩展实例
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

### ExtensionLoader私有构造器

&nbsp; &nbsp; 在`getExtensionLoader`会调用私有构造器这时候会初始化`ExtensionLoader`类，代码逻辑如下：
1. 实例化一些属性，一开始这些属性都实例化类但是是空的集合；
2. 当第一次执行私有构造器`ExtensionLoader`时，`type`会是其他的`SPI`接口，这时候就会走到`ExtensionLoader.getExtensionLoader(ExtensionFactory.class)`，此时会讲`ExtensionFactory`加到缓存中；
3. 调用`getAdaptiveExtension`方法，获得自适应扩展点的实例，将获取到的实例复制给objectFactory，通过debug返回的集合包含`SpiExtensionFactory`和`SpringExtensionFactory`。

```
    private final ConcurrentMap<Class<?>, String> cachedNames = new ConcurrentHashMap<Class<?>, String>();

    private final Holder<Map<String, Class<?>>> cachedClasses = new Holder<Map<String, Class<?>>>();

    private final Map<String, Activate> cachedActivates = new ConcurrentHashMap<String, Activate>();
    private final ConcurrentMap<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<String, Holder<Object>>();
    private final Holder<Object> cachedAdaptiveInstance = new Holder<Object>();

    private Map<String, IllegalStateException> exceptions = new ConcurrentHashMap<String, IllegalStateException>();

    // 私有化构造器
    private ExtensionLoader(Class<?> type) {
        this.type = type;
        objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
    }
```

### getAdaptiveExtension方法

&nbsp; &nbsp; 该方法获取自适应扩展点，代码如下：

```
    public T getAdaptiveExtension() {
        // 从缓存中获取自适应扩展点对象实例
        Object instance = cachedAdaptiveInstance.get();
        // 获取不到，采用双重检查
        if (instance == null) {
            if (createAdaptiveInstanceError == null) {
                // 对缓存使用对象监视器（锁）
                synchronized (cachedAdaptiveInstance) {
                    instance = cachedAdaptiveInstance.get();
                    if (instance == null) {
                        try {
                            // 创建自适应扩展点对象
                            instance = createAdaptiveExtension();
                            // 缓存该对象
                            cachedAdaptiveInstance.set(instance);
                        } catch (Throwable t) {
                            createAdaptiveInstanceError = t;
                            throw new IllegalStateException("fail to create adaptive instance: " + t.toString(), t);
                        }
                    }
                }
            } else {
                throw new IllegalStateException("fail to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
            }
        }

        return (T) instance;
    }
```

### createAdaptiveExtension方法

&nbsp; &nbsp; 创建自适应扩展点对象，代码如下，这里先不看`injectExtension`方法后续会详细看一下这个方法的源码并分析其作用，以下代码通过`getAdaptiveExtensionClass().newInstance()`创建自适应扩展点的对象
```
    private T createAdaptiveExtension() {
        try {
            return injectExtension((T) getAdaptiveExtensionClass().newInstance());
        } catch (Exception e) {
            throw new IllegalStateException("Can not create adaptive extension " + type + ", cause: " + e.getMessage(), e);
        }
    }
```

### getAdaptiveExtensionClass方法

&nbsp; &nbsp; 首先会调用`getExtensionClasses()`加载扩展的类（详细流程后续分析），如果自适应扩展类缓存不为空返回自适应扩展类，否则调用`createAdaptiveExtensionClass()`方法创建自适应扩展类。

```
    private Class<?> getAdaptiveExtensionClass() {
        getExtensionClasses();
        if (cachedAdaptiveClass != null) {
            return cachedAdaptiveClass;
        }
        return cachedAdaptiveClass = createAdaptiveExtensionClass();
    }
```

### createAdaptiveExtensionClass方法

&nbsp; &nbsp; 该方法的代码如下，该方法作用就是创建自适应扩展的类，代码分析可以参考代码中的注释

```
    private Class<?> createAdaptiveExtensionClass() {
        // 创建自适应类的代码，这里不详细看，这个方法其实是通过动态代理生成代理类，使用反射进行了字节码增强操作
        String code = createAdaptiveExtensionClassCode();
        // 获取类加载器
        ClassLoader classLoader = findClassLoader();
        // 获取字节码增强编译器，这里字节码增强的编译器也是可扩展的，这里默认使用的是javassist
        com.alibaba.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
        // 使用字节码增强编译器根据字节码，类加载器创建类
        return compiler.compile(code, classLoader);
    }
```

### 从配置文件中加载扩展实现源码分析

&nbsp; &nbsp; 该方法一开始会从`cachedClasses`缓存中获取支持的扩展，如果获取不到那么就从固定路径下的扩展配置文件中加载扩展点，代码如下，这里代码比较多也比较复杂，大致流程还是比较容易理解，每个方法逻辑如下：
1. getExtensionClasses：尝试从缓存中获取扩展点，扩展点的类信息存到了自己定义的一个java bean中，从缓存中获取不到时调用`loadExtensionClasses`方法加载扩展点类，并缓存
2. loadExtensionClasses：首先会判断type类是否有`SPI`注解，如果有会检查注解的值如果有值则会缓存默认扩展类名，然后会从固定的路径加载扩展点，这里的路径有`META-INF/dubbo/internal/`，`META-INF/dubbo/`，`META-INF/services/`
3. loadDirectory：获取类加载器，根据`URL`加载扩展点
4. loadResource：加载可扩展的接口到`cachedNames`中
5. loadClass：判断加载的类是否为Wapper类，是则将其放到Wapper类的缓存中；否则直接进行加载处理，加载可扩展类到`cachedNames`中

```
private Map<String, Class<?>> getExtensionClasses() {
        Map<String, Class<?>> classes = cachedClasses.get();
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
                    classes = loadExtensionClasses();
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
    }

    // synchronized in getExtensionClasses
    private Map<String, Class<?>> loadExtensionClasses() {
        final SPI defaultAnnotation = type.getAnnotation(SPI.class);
        if (defaultAnnotation != null) {
            String value = defaultAnnotation.value();
            if ((value = value.trim()).length() > 0) {
                String[] names = NAME_SEPARATOR.split(value);
                if (names.length > 1) {
                    throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                            + ": " + Arrays.toString(names));
                }
                if (names.length == 1) cachedDefaultName = names[0];
            }
        }

        Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
        loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
        loadDirectory(extensionClasses, DUBBO_DIRECTORY);
        loadDirectory(extensionClasses, SERVICES_DIRECTORY);
        return extensionClasses;
    }

    private void loadDirectory(Map<String, Class<?>> extensionClasses, String dir) {
        String fileName = dir + type.getName();
        try {
            Enumeration<java.net.URL> urls;
            ClassLoader classLoader = findClassLoader();
            if (classLoader != null) {
                urls = classLoader.getResources(fileName);
            } else {
                urls = ClassLoader.getSystemResources(fileName);
            }
            if (urls != null) {
                while (urls.hasMoreElements()) {
                    java.net.URL resourceURL = urls.nextElement();
                    loadResource(extensionClasses, classLoader, resourceURL);
                }
            }
        } catch (Throwable t) {
            logger.error("Exception when load extension class(interface: " +
                    type + ", description file: " + fileName + ").", t);
        }
    }
    private void loadResource(Map<String, Class<?>> extensionClasses, ClassLoader classLoader, java.net.URL resourceURL) {
        try {
            BufferedReader reader = new BufferedReader(new InputStreamReader(resourceURL.openStream(), "utf-8"));
            try {
                String line;
                while ((line = reader.readLine()) != null) {
                    final int ci = line.indexOf('#');
                    if (ci >= 0) line = line.substring(0, ci);
                    line = line.trim();
                    if (line.length() > 0) {
                        try {
                            String name = null;
                            int i = line.indexOf('=');
                            if (i > 0) {
                                name = line.substring(0, i).trim();
                                line = line.substring(i + 1).trim();
                            }
                            if (line.length() > 0) {
                                loadClass(extensionClasses, resourceURL, Class.forName(line, true, classLoader), name);
                            }
                        } catch (Throwable t) {
                            IllegalStateException e = new IllegalStateException("Failed to load extension class(interface: " + type + ", class line: " + line + ") in " + resourceURL + ", cause: " + t.getMessage(), t);
                            exceptions.put(line, e);
                        }
                    }
                }
            } finally {
                reader.close();
            }
        } catch (Throwable t) {
            logger.error("Exception when load extension class(interface: " +
                    type + ", class file: " + resourceURL + ") in " + resourceURL, t);
        }
    }

    private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
        if (!type.isAssignableFrom(clazz)) {
            throw new IllegalStateException("Error when load extension class(interface: " +
                    type + ", class line: " + clazz.getName() + "), class "
                    + clazz.getName() + "is not subtype of interface.");
        }
        if (clazz.isAnnotationPresent(Adaptive.class)) {
            if (cachedAdaptiveClass == null) {
                cachedAdaptiveClass = clazz;
            } else if (!cachedAdaptiveClass.equals(clazz)) {
                throw new IllegalStateException("More than 1 adaptive class found: "
                        + cachedAdaptiveClass.getClass().getName()
                        + ", " + clazz.getClass().getName());
            }
        } else if (isWrapperClass(clazz)) {
            Set<Class<?>> wrappers = cachedWrapperClasses;
            if (wrappers == null) {
                cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
                wrappers = cachedWrapperClasses;
            }
            wrappers.add(clazz);
        } else {
            clazz.getConstructor();
            if (name == null || name.length() == 0) {
                name = findAnnotationName(clazz);
                if (name.length() == 0) {
                    throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + resourceURL);
                }
            }
            String[] names = NAME_SEPARATOR.split(name);
            if (names != null && names.length > 0) {
                Activate activate = clazz.getAnnotation(Activate.class);
                if (activate != null) {
                    cachedActivates.put(names[0], activate);
                }
                for (String n : names) {
                    if (!cachedNames.containsKey(clazz)) {
                        cachedNames.put(clazz, n);
                    }
                    Class<?> c = extensionClasses.get(n);
                    if (c == null) {
                        extensionClasses.put(n, clazz);
                    } else if (c != clazz) {
                        throw new IllegalStateException("Duplicate extension " + type.getName() + " name " + n + " on " + c.getName() + " and " + clazz.getName());
                    }
                }
            }
        }
    }
    private boolean isWrapperClass(Class<?> clazz) {
        try {
            clazz.getConstructor(type);
            return true;
        } catch (NoSuchMethodException e) {
            return false;
        }
    }
``` 

### injectExtension方法

&nbsp; &nbsp; 通过该方法的主要目的是进行依赖注入，代码及流程如下：
1. 通过反射过滤出已`set`开头的方法
2. 判断时候被`DisableInject`修饰，如果被修饰那就不进行依赖注入
3. 获取到类信息，根据类信息获取到属性信息`property`
4. 实例化扩展类
5. 调用`invoke`方法进行依赖注入，其实就是调用实例对象的`setter`方法，这里是通过反射实现

```
    private T injectExtension(T instance) {
        try {
            if (objectFactory != null) {
                for (Method method : instance.getClass().getMethods()) {
                    if (method.getName().startsWith("set")
                            && method.getParameterTypes().length == 1
                            && Modifier.isPublic(method.getModifiers())) {
                        /**
                         * Check {@link DisableInject} to see if we need auto injection for this property
                         */
                        if (method.getAnnotation(DisableInject.class) != null) {
                            continue;
                        }
                        Class<?> pt = method.getParameterTypes()[0];
                        try {
                            String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                            Object object = objectFactory.getExtension(pt, property);
                            if (object != null) {
                                method.invoke(instance, object);
                            }
                        } catch (Exception e) {
                            logger.error("fail to inject via method " + method.getName()
                                    + " of interface " + type.getName() + ": " + e.getMessage(), e);
                        }
                    }
                }
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return instance;
    }
```

### AdaptiveExtensionFactory类getExtension方法

&nbsp; &nbsp; 该方法是`SPI`的实现类动态加载的方法，实际通过`createExtension()`方法加载对象实例，代码及处理流程如下：

1. 判断扩展名如`(@SPI("dubbo")中的dubbo)`不为空
2. 扩展名如果微`true`返回默认扩展，默认扩展代码不展示了，看代码逻辑会返回`null`
3. 从扩展实现实例缓存中获取扩展对象`Holder`，如果是空的，将扩展名和`Holder`对象放到缓存中，并返回扩展名的holder
4. `get()`扩展对象，使用双重检查，如果对象是空的那么通过`createExtension(name)`创建这个对象并 `set`到`holder`中
```
    public T getExtension(String name) {
        if (name == null || name.length() == 0)
            throw new IllegalArgumentException("Extension name == null");
        if ("true".equals(name)) {
            return getDefaultExtension();
        }
        Holder<Object> holder = cachedInstances.get(name);
        if (holder == null) {
            cachedInstances.putIfAbsent(name, new Holder<Object>());
            holder = cachedInstances.get(name);
        }
        Object instance = holder.get();
        if (instance == null) {
            synchronized (holder) {
                instance = holder.get();
                if (instance == null) {
                    instance = createExtension(name);
                    holder.set(instance);
                }
            }
        }
        return (T) instance;
    }
```

### createExtension方法

&nbsp; &nbsp; 代码及流程如下：
1. 根据扩展名，获取扩展类的`Class`
2. 从缓存中获取扩展类的对象实例
3. 如果实例为空通过反射创建实例并放到`EXTENSION_INSTANCES`缓存中
4. `injectExtension`方法前面已经有过说明
5. 如果包装器类缓存不为空就遍历包装器缓存，创建包装器类对象，这里的包装器主要有`org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper`,`listener=org.apache.dubbo.rpc.protocol.ProtocolListenerWrapper`等

```
    private T createExtension(String name) {
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw findException(name);
        }
        try {
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
                EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
            injectExtension(instance);
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
                for (Class<?> wrapperClass : wrapperClasses) {
                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                }
            }
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                    type + ")  could not be instantiated: " + t.getMessage(), t);
        }
    }
```

## 总结

&nbsp; &nbsp; 断断续续花了两天的时间将dubbo spi的部分实现整理了一下，实际上还有很多源码没拆解，感兴趣的朋友可以自己私下debug一下代码来理解dubbo spi的流程，我是基于dubbo 2.6.5版本分析的，如果分析有无也请欢迎指正，如果想仿照dubbo实现一套spi机制我建议阅读另一款开源中间件`soul`网关的spi实现，`soul`网关的spi实现其实就是仿照的dubbo spi实现，但是与dubbo相比，soul网关的实现会简单许多，对于初学者和快速理解spi的朋友比dubbo更加友好。