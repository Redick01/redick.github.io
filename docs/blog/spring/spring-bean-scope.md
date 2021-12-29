# Spring Bean的作用域 <!-- {docsify-ignore-all} -->


## 单例

&nbsp; &nbsp; Bean的Scope默认就是单例的

#### 测试单例Scope


## 原型

&nbsp; &nbsp; 通过`@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)`标注的bean是原型的

## 自定义

&nbsp; &nbsp; 实现自定义Scope代码如下：

```java
public class CustomizeScopeBean implements Scope {

    public static final String SCOPE_NAME = "thread-local";

    public static final NamedThreadLocal<Map<String, Object>> namedThreadLocal = new NamedThreadLocal("thread-local-obj") {

        @Override
        public Map<String, Object> initialValue() {
            return new HashMap<>();
        }
    };

    @Override
    public Object get(String name, ObjectFactory<?> objectFactory) {
        Map<String, Object> context = getContext();
        Object bean = context.get(name);
        if (null == bean) {
            bean = objectFactory.getObject();
            context.put(name, bean);
        }
        return bean;
    }

    @Override
    public Object remove(String name) {
        Map<String, Object> context = getContext();
        return context.remove(name);
    }

    @Override
    public void registerDestructionCallback(String name, Runnable callback) {

    }

    @Override
    public Object resolveContextualObject(String key) {
        Map<String, Object> context = getContext();
        return context.get(key);
    }

    @Override
    public String getConversationId() {
        Thread thread = Thread.currentThread();
        return String.valueOf(thread.getId());
    }

    private Map<String, Object> getContext() {
        return namedThreadLocal.get();
    }
}
```

&nbsp; &nbsp; 测试自定义bean代码：

```java
public class CustomizeScopeBeanDemo {

    @Bean
    @Scope(CustomizeScopeBean.SCOPE_NAME)
    public User user() {
        return createUser();
    }

    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
        context.register(CustomizeScopeBeanDemo.class);
        // 注册自定义scope
        context.addBeanFactoryPostProcessor(beanFactory -> {
            beanFactory.registerScope(CustomizeScopeBean.SCOPE_NAME, new CustomizeScopeBean());
        });
        context.refresh();
        beanDependencyLookUp(context);
        lookup(context);
        context.stop();
    }

    private User createUser() {
        User user = new User();
        user.setId(System.nanoTime());
        user.setUsername("mer");
        return user;
    }

    private static void beanDependencyLookUp(AnnotationConfigApplicationContext context) {
        for (int i = 0; i < 4; i++) {

            Thread thread = new Thread(() -> {
                User user = context.getBean("user", User.class);
                System.out.println(user);
            });
            thread.start();
            try {
                thread.join();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    private static void lookup(AnnotationConfigApplicationContext context) {
        for (int i = 0; i < 4; i++) {
            User user = context.getBean("user", User.class);
            System.out.println(user);
        }
    }
}
```