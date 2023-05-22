# 依赖版本管理及项目结构 <!-- {docsify-ignore-all} -->



## 依赖版本管理



### Spring Cloud基础依赖版本

```xml
  <properties>
     	 	<!-- Spring Cloud Alibaba-->
        <spring-cloud-alibaba.version>2021.0.4.0</spring-cloud-alibaba.version>
        <!-- Spring Cloud -->
        <spring.cloud.version>2021.0.5</spring.cloud.version>
        <!-- Spring Boot -->
        <spring-boot.version>2.7.7</spring-boot.version>
    		<lomnok.version>1.18.24</lomnok.version>
    		<log-helper.version>1.0.5-RELEASE</log-helper.version>
  </properties>
	<dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>${spring-cloud-alibaba.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring.cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
  </dependencyManagement>
	<dependencies>
        <!-- bootstrap 启动器 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-bootstrap</artifactId>
        </dependency>
        
    		<!-- spring cloud alibaba nacos服务注册发现 -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
        
    		<!-- spring cloud alibaba nacos配置中心 -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>${lomnok.version}</version>
        </dependency>
        
        <!-- spring cloud alibaba nacos配置中心 -->
        <dependency>
            <groupId>io.github.redick01</groupId>
            <artifactId>log-helper-spring-boot-starter-common</artifactId>
            <version>${log-helper.version}</version>
        </dependency>
   </dependencies>
```



### 项目目录结构

```shell
# 安全认证模块
├── ruuby-auth
# 业务模块
├── ruuby-biz
# 通用代码模块
├── ruuby-common
# 微服务网关模块
├── ruuby-gateway
# 组件模块
└── ruuby-modules
		# 动态数据源，读写分离组件
    ├── ruuby-module-datasource
    # Mybatis-Plus代码生成模块
    ├── ruuby-module-mybatisplus-generator
    # Spring Redis封装模块
    ├── ruuby-module-redis
    # RocketMQ版本控制
    ├── ruuby-module-rocketmq
    # 分布式事务Seata模块
    ├── ruuby-module-seata
    # 系统保护Sentinel模块
    ├── ruuby-module-sentinel
    # API文档Swagger集成模块
    ├── ruuby-module-swagger
    # 定时任务XXL-JOB模块
    └── ruuby-module-xxljob
```

