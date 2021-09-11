# SpringBoot自动装配 <!-- {docsify-ignore-all} -->

- 什么是SpringBoot自动装配
- SpringBoot怎么实现的自动装配


注：源码解析部分基于Spring5.0.4 SpringBoot版本为2.0.0.RELEASE


## 什么是SpringBoot自动装配

&nbsp; &nbsp; 在使用SpringBoot开发应用程序的时候我们能够极其快速的就搭建出一个可以丝滑运行的工程代码，这得益于SpringBoot的自动装配能力，但是对于自动装配，Spring Framework早就已经实现了这个功能，SpringBoot是在这个基础上做了优化，其机制就像是SPI。但是大多数的开发人员可能仅仅听到过自动装配，并且一提到自动装配就想到SpringBoot，但究竟自动装配是怎么实现的呢？按需装配又是怎么一回事却没有深究其理。

> SpringBoot在启动的时候会扫描外部引用的jar包中/META-INF/spring.factories文件，然后根据文件中的配置加载到Spring容器中，对于外部的jar来说，要支持SpringBoot就按照SpringBoot自动装配的规范封装对应的starter即可。

&nbsp; &nbsp; 在使用Spring Framework搭建项目需要依赖第三方库时，除了要引入第三方的jar之外，还需要进行相应的配置，也许是xml配置，也许是java配置，总之这很麻烦，但如果使用SpringBoot，我们就只需要引入相应的starter并且在配置文件`application.properties`或`application.yml`文件中添加少量的配置即可，例如，项目中使用redis，那就直接在maven中引入`spring-boot-starter-data-redis`的starter然后再通过少量的注解或者配置就可以使用第三方组件了。



## SpringBoot怎么实现的自动装配


### 注解

&nbsp; &nbsp; 首先对于一个开发人员最直观的就是SpringBoot提供的自动装配注解`@EnableAutoConfiguration`，这个注解就是声明自动装配的，但是通常情况下我们都不使用这个注解，而是使用`@SpringBootApplication`这个注解，该注解使一个复合注解它是`@EnableAutoConfiguration`、`@ComponentScan`、`@SpringBootConfiguration`注解的集合，这三个注解的作用分别是，开启SpringBoot自动装配、扫描启动类所在包下所有被`@Service`,`@Component`,`@Controller`等注解修饰的Bean、在上下文注册bean一般修饰配置类。

&nbsp; &nbsp; `@EnableAutoConfiguration`注解是声明自动装配的，但是其实自动装配怎么实现的我们还是一头雾水，但是仔细看`@EnableAutoConfiguration`这个注解的代码其实能够发现，这个注解`@Import`了`AutoConfigurationImportSelector`这个类，`@Import`注解作用就是将`AutoConfigurationImportSelector`初始化到Spring上下文，所以`AutoConfigurationImportSelector`其实就是SpringBoot自动装配的一个实现了。下面我们通过分析下这个类。

### AutoConfigurationImportSelector.java

#### 该类的继承关系如下：

红框中的继承关系是实现自动装配的核心，其核心的接口是`ImportSelector`，`AutoConfigurationImportSelector`实现了该接口并且重写了`selectImports`方法，方法返回需要装配的类的全路径名的字符串数组。

![avatar](../../_media/image/spring/autoconfigurationimportselector.png)


#### **下面是AutoConfigurationImportSelector#selectImports具体实现的代码**

```java
    /**
    AnnotationMetadata 注解元数据
    */
    @Override
	public String[] selectImports(AnnotationMetadata annotationMetadata) {
        // 1. 是否开启自动装配开关
		if (!isEnabled(annotationMetadata)) {
			return NO_IMPORTS;
		}
		try {
            // 2.读取所有自动装配的bean
			AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
					.loadMetadata(this.beanClassLoader);
            // 3.注解属性exclude和excludeName属性
			AnnotationAttributes attributes = getAttributes(annotationMetadata);
            // 4.读取spring-boot-autoconfigure包META-INF/spring.factories中所有自动装配的bean
			List<String> configurations = getCandidateConfigurations(annotationMetadata,
					attributes);
            // 5.去重
			configurations = removeDuplicates(configurations);
            // 6.根据@Order注解排序
			configurations = sort(configurations, autoConfigurationMetadata);
            // 7.排除指定不要需要自动装配的class
			Set<String> exclusions = getExclusions(annotationMetadata, attributes);
			checkExcludedClasses(configurations, exclusions);
			configurations.removeAll(exclusions);
            // 8.过滤自动装配的bean，做到按需装配
			configurations = filter(configurations, autoConfigurationMetadata);
			fireAutoConfigurationImportEvents(configurations, exclusions);
			return StringUtils.toStringArray(configurations);
		}
		catch (IOException ex) {
			throw new IllegalStateException(ex);
		}
	}
```

##### **第一步，自动装配开关是否开启**

1. 判读`spring.boot.enableautoconfiguration`配置，如果没有配置默认是true代表默认开启自动装配，否则如果显示配置了false就不进行自动装配返回一个空的字符串数组。

```java
    protected boolean isEnabled(AnnotationMetadata metadata) {
		if (getClass() == AutoConfigurationImportSelector.class) {
			// 1
			return getEnvironment().getProperty(
					EnableAutoConfiguration.ENABLED_OVERRIDE_PROPERTY, Boolean.class,
					true);
		}
		return true;
	}
```

##### **第二步，读取所有自动装配的bean**

1. 读取META-INF/spring-autoconfigure-metadata.properties中所有自动装配bean的配置
2. 将读取到的所有自动装配的bean初始化到Properties中
3. 将所有自动装配的bean加载到AutoConfigurationMetadata

```java
	public static AutoConfigurationMetadata loadMetadata(ClassLoader classLoader) {
		return loadMetadata(classLoader, PATH);
	}

	static AutoConfigurationMetadata loadMetadata(ClassLoader classLoader, String path) {
		try {
			// 1. META-INF/spring-autoconfigure-metadata.properties
			Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(path)
					: ClassLoader.getSystemResources(path));
			Properties properties = new Properties();
			while (urls.hasMoreElements()) {
				// 2
				properties.putAll(PropertiesLoaderUtils
						.loadProperties(new UrlResource(urls.nextElement())));
			}
			// 3
			return loadMetadata(properties);
		}
		catch (IOException ex) {
			throw new IllegalArgumentException(
					"Unable to load @ConditionalOnClass location [" + path + "]", ex);
		}
	}

	static AutoConfigurationMetadata loadMetadata(Properties properties) {
		return new PropertiesAutoConfigurationMetadata(properties);
	}

	...
```

##### **第三步，获取@EnableAutoConfiguration注解exclude和excludeName属性**

1. 获取@EnableAutoConfiguration注解name
2. 初始化exclude和excludeName属性，该属性是排除不需要自动装配的bean的配置

```java
    protected AnnotationAttributes getAttributes(AnnotationMetadata metadata) {
        // 1
		String name = getAnnotationClass().getName();
        // 2
		AnnotationAttributes attributes = AnnotationAttributes
				.fromMap(metadata.getAnnotationAttributes(name, true));
		Assert.notNull(attributes,
				"No auto-configuration attributes found. Is " + metadata.getClassName()
						+ " annotated with " + ClassUtils.getShortName(name) + "?");
		return attributes;
	}
```

##### **第四步，读取spring-boot-autoconfigure包META-INF/spring.factories中所有自动装配的bean**

![avatar](../../_media/image/spring/factories-config.jpeg)

简单的看一下下面的代码，下面的代码是从spring-boot-autoconfigure包META-INF/spring.factories获取自动装配bean的列表的过程。

```java
    protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
			AnnotationAttributes attributes) {
        // 1
		List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
				getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
		Assert.notEmpty(configurations,
				"No auto configuration classes found in META-INF/spring.factories. If you "
						+ "are using a custom packaging, make sure that file is correct.");
		return configurations;
	}

    public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
		String factoryClassName = factoryClass.getName();
        // 2
		return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
	}

	private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
		MultiValueMap<String, String> result = cache.get(classLoader);
		if (result != null)
			return result;
		try {
            // 4
			Enumeration<URL> urls = (classLoader != null ?
					classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
			result = new LinkedMultiValueMap<>();
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				UrlResource resource = new UrlResource(url);
				Properties properties = PropertiesLoaderUtils.loadProperties(resource);
				for (Map.Entry<?, ?> entry : properties.entrySet()) {
					List<String> factoryClassNames = Arrays.asList(
							StringUtils.commaDelimitedListToStringArray((String) entry.getValue()));
					result.addAll((String) entry.getKey(), factoryClassNames);
				}
			}
			cache.put(classLoader, result);
			return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load factories from location [" +
					FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
	}
```

#### **第五步，自动装配bean去重**

这一步没什么好说的使用Set做去重。

```java
    protected final <T> List<T> removeDuplicates(List<T> list) {
		return new ArrayList<>(new LinkedHashSet<>(list));
	}
```

#### **第六步，根据@Order注解排序**

这一步也没什么好说的，根绝@Order注解将自动装配的Bean先排序

```java
    private List<String> sort(List<String> configurations,
			AutoConfigurationMetadata autoConfigurationMetadata) throws IOException {
		configurations = new AutoConfigurationSorter(getMetadataReaderFactory(),
				autoConfigurationMetadata).getInPriorityOrder(configurations);
		return configurations;
	}
```

#### **第七步，根据第三步exclude和excludeName属性排除指定不要需要自动装配的class**

这里也没什么好说的，根据第三步的exclude和excludeName得到一个要排除自动装配的bean的集合，然后removeAll移除

```java

    @Override
	public String[] selectImports(AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return NO_IMPORTS;
		}
		try {
			...
			Set<String> exclusions = getExclusions(annotationMetadata, attributes);
			checkExcludedClasses(configurations, exclusions);
			configurations.removeAll(exclusions);
			...
		}
		catch (IOException ex) {
			throw new IllegalStateException(ex);
		}
	}

    protected Set<String> getExclusions(AnnotationMetadata metadata,
			AnnotationAttributes attributes) {
		Set<String> excluded = new LinkedHashSet<>();
		excluded.addAll(asList(attributes, "exclude"));
		excluded.addAll(Arrays.asList(attributes.getStringArray("excludeName")));
		excluded.addAll(getExcludeAutoConfigurationsProperty());
		return excluded;
	}
```

#### **第八步，过滤不需要自动装配的bean，做到按需装配**


