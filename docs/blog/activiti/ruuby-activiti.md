# Spring Boot集成Activiti工作流 <!-- {docsify-ignore-all} -->


## 后端集成


### maven依赖


```xml
    <dependencies>
        <dependency>
            <groupId>org.activiti</groupId>
            <artifactId>activiti-spring-boot-starter-rest-api</artifactId>
            <version>${activiti.version}</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lombok.version}</version>
        </dependency>
    </dependencies>
```

### 解決工作流生成图片