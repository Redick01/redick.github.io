# SpringBoot自动装配 <!-- {docsify-ignore-all} -->

- 什么是SpringBoot自动装配
- SpringBoot怎么实现的自动装配


## 什么是SpringBoot自动装配

&nbsp; &nbsp; 在使用SpringBoot开发应用程序的时候我们能够极其快速的就搭建出一个可以丝滑运行的工程代码，这得益于SpringBoot的自动装配能力，但是对于自动装配，Spring Framework早就已经实现了这个功能，SpringBoot是在这个基础上做了优化，其机制就像是SPI。但是大多数的开发人员可能仅仅听到过自动装配，并且一提到自动装配就想到SpringBoot，但究竟自动装配是怎么实现的呢？按需装配又是怎么一回事却没有深究其理。

> SpringBoot在启动的时候会扫描外部引用的jar包中/META-INF/spring.factories文件，然后根据文件中的配置加载到Spring容器中，对于外部的jar来说，要支持SpringBoot就按照SpringBoot自动装配的规范封装对应的starter即可。

&nbsp; &nbsp; 在使用Spring Framework搭建项目需要依赖第三方库时，除了要引入第三方的jar之外，还需要进行相应的配置，也许是xml配置，也许是java配置，总之这很麻烦，但如果使用SpringBoot，我们就只需要引入相应的starter并且在配置文件`application.properties`或`application.yml`文件中添加少量的配置即可，例如，项目中使用redis，那就直接在maven中引入`spring-boot-starter-data-redis`的starter然后再通过少量的注解或者配置就可以使用第三方组件了。



## SpringBoot怎么实现的自动装配
