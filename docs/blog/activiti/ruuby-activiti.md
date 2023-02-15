# Spring Boot集成Activiti 6.0流程设计器（一） <!-- {docsify-ignore-all} -->

&nbsp; &nbsp; 最近公司的管理系统有工作流程审批的需求，准备引入工作流技术，因此，经过技术选型，选择使用`Activiti6.0`作为工作流引擎，使用工作流就离不开流程设计器，本篇文章不介绍什么是工作流，也不介绍Activiti技术和BPMN规范等，主要介绍如何去基于`Activiti6.0`集成其Web端的流程设计器，并实现与既有管理系统的用户体系打通。

&nbsp; &nbsp; 本篇文章会介绍Activiti 6.0流程设计器继承中我所遇到的各种问题，以及解决办法。

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

&nbsp; &nbsp; 在应用启动类的`@SpringBootApplication`注解排除Spring Security和DataSource的Configuration，所为排除就是在Spring容器启动过程中不去加载排除的Class，因为我所集成的后台项目已经有数据源的配置了，所以这里要排除掉Activiti的数据源配置的Bean，因为Activiti和SpringSecurity有集成，然后我的项目不需要SpringSecurity，所以，这里也排除掉SpringSecurity要自动装配的配置，如果不排除掉服务缺少SpringSecurity相关的dependency就会报错。


```java
@SpringBootApplication(exclude = {
        DataSourceAutoConfiguration.class,
        org.activiti.spring.boot.SecurityAutoConfiguration.class,
        org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration.class,
})
```

### 流程设计器接口定义

&nbsp; &nbsp; 这里就是Activiti的流程设计器的接口了，这些都是可以从Activiti Modeler那里获取到，这里简单的列一下所有的接口

```java
  /**
   * 角色列表
   * @param role
   * @return
   */
  @GetMapping("/modeler/role/list")
  public TableDataInfo list(RoleRequestDTO role) throws IOException;

  /**
   * 某角色下的用户列表
   * @param roleKey
   * @return
   */
  @GetMapping("/modeler/user/listByRoleKey")
  @LogMarker(businessDescription = "某角色下的用户列表")
  public TableDataInfo list(String roleKey) throws IOException;

  /**
  * 保存流程定义
  */
  @RequestMapping(value="/modeler/model/{modelId}/save", method = RequestMethod.POST)
  @ResponseStatus(value = HttpStatus.OK)
  public void saveModel(@PathVariable String modelId, @RequestBody MultiValueMap<String, String> values);

   // 获取stencilset.json数据
   @RequestMapping(value="/modeler/editor/stencilset", method = RequestMethod.GET, produces = "application/json;charset=utf-8")
  public @ResponseBody String getStencilset();

  // 获取模型JSON
  @RequestMapping(value="/modeler/model/{modelId}/json", method = RequestMethod.GET, produces = "application/json")
  public ObjectNode getEditorJson(@PathVariable String modelId);
```

## 前端集成

### 流程设计器集成

&nbsp; &nbsp; 将流程设计器的前端资源拷贝到resource/static文件夹下，将stencilset.json文件放到resource文件夹下


## 总结

&nbsp; &nbsp; 上面只是简单的介绍了下Activiti Modeler的简单集成，其中的一些细节并为体现，因为代码量比较多，就不写出来了，后面参考我github上的代码。

&nbsp; &nbsp; 做了以上的集成后实际上就可以通过使用流程设计器了，但是也是打开，编辑，保存已经存在的流程模型，无法完成新建流程模型，如果想完成流程模型创建，编辑，挂起，激活，转换等等操作的话，就需要我们自己去开发业务，开发接口，然后去实现相关的功能了。





### 解决跨域问题

### 解决文件上传,临时文件夹被程序自动删除问题

### 解决前端上传文件没有进度条问题

### 待办、已办开发