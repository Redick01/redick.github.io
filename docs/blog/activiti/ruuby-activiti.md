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
        <!--activiti modeler start-->
        <dependency>
            <groupId>org.activiti</groupId>
            <artifactId>activiti-json-converter</artifactId>
            <version>${activiti.version}</version>
            <exclusions>
                <exclusion>
                    <groupId>org.activiti</groupId>
                    <artifactId>activiti-bpmn-model</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <!-- xml解析依赖-->
        <dependency>
            <groupId>org.apache.xmlgraphics</groupId>
            <artifactId>batik-codec</artifactId>
            <version>1.7</version>
        </dependency>
        <dependency>
            <groupId>org.apache.xmlgraphics</groupId>
            <artifactId>batik-css</artifactId>
            <version> 1.7</version>
        </dependency>
        <dependency>
            <groupId>org.apache.xmlgraphics</groupId>
            <artifactId>batik-svg-dom</artifactId>
            <version>1.7</version>
        </dependency>
        <dependency>
            <groupId>org.apache.xmlgraphics</groupId>
            <artifactId>batik-svggen</artifactId>
            <version>1.7</version>
        </dependency>
        <!-- xml解析依赖-->
        <!--activiti modeler end-->
    </dependencies>
```

### 排除Spring Security和DataSource的Configuration

&nbsp; &nbsp; 在应用启动类的`@SpringBootApplication`注解排除Spring Security和DataSource的Configuration


```java
@SpringBootApplication(exclude = {
        DataSourceAutoConfiguration.class,
        org.activiti.spring.boot.SecurityAutoConfiguration.class,
})
```

### 流程设计器集成

/activiti-explorer/service/model//json 问题



### 解决跨域问题

### 解决文件上传,临时文件夹被程序自动删除问题

### 解决前端上传文件没有进度条问题

### 待办、已办开发