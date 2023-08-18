# spring aop无法拦截接口（interface）上的方法 <!-- {docsify-ignore-all} -->



**注解继承特性：**

1. 被@Inherited元注解标注的注解标注在类上的时候，子类可以继承父类上的注解。
2. 注解未被@Inherited元注解标注的，该注解标注在类上时，子类不会继承父类上标注的注解。
3. 注解标注在接口上，其子类及子接口都不会继承该注解
4. 注解标注在类或接口方法上，其子类重写该方法不会继承父类或接口中方法上标记的注解



**原因：**

1. 接口没有实现方法，spring依赖注入管理的是对象，接口spring可管理不到，既然不是spirng管理的对象，你spirng+aop的配置肯定是失效的
2. 注解是可以继承，但只有类的注解是可以继承的,还需要@Inherited，方法的注解继承不了，spring管理的是**实际对象**,你加不上注解，更不可能拦截到

因为我们使用了 AOP 特性，与之相关联的便是 Spring 动态代理 了。Spring 的动态代理主要分为两种，一种是JDK 动态代理 ；一种是CGLIB 动态代理

- 使用 JDK 动态代理

JDK 动态代理主要是针对实现了某个接口的类。该方式基于反射的机制实现，会生成一个实现相同接口的代理类，然后通过对方法的重写，实现对代码的增强。

在该方式中接口中的注解无法被实现类继承，AOP 中的切点无法匹配上实现类，所以也就不会为实现类创建代理，所以我们使用的类其实是未被代理的原始类，自然也就不会被增强了。



- 使用 CGLIB 动态代理

在不存在切点注解继承的情况，AOP 可进行有效拦截（CGLIB动态代理）。但是还要考虑以下存在注解继承的情况：

有父类 Parent，和子类 Sub ，切点注解在父类方法。则根据上边提到的只有方法的重写问题，可知，被重写的方法将不会被拦截，而未重写的方法则走 Parent 路线，可以被 AOP 感知拦截。



**AOP创建代理和执行的大致流程：**

1. 在bean初始化后 postProcessAfterInitialization
2. 是否配置切点wrapIfNecessary，就是配置切点了没
3. 创建代理，根据bean(对象)创建代理，创建代理的过程中和class实现的interface没关系，将切点组装成一个切点链
4. 执行过程中走到动态代理的intercept方法中，获取到切点的链，执行链，proceed



**怎么实现接口方法代理**

1. 自定义Advisor
2. 自定义切面方法拦截



#### 参考：

https://www.coder.work/article/15303
https://github.com/spring-projects/spring-framework/issues/22311
https://github.com/alibaba/Sentinel/issues/284
https://github.com/spring-projects/spring-framework/issues/12320

https://stackoverflow.com/questions/4745798/why-java-classes-do-not-inherit-annotations-from-implemented-interfaces