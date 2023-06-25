# 依赖版本管理及项目结构 <!-- {docsify-ignore-all} -->



## 依赖版本管理



### Spring Cloud基础依赖版本

```xml
  <properties>
     	 	<maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
        <!-- Project revision -->
        <revision>1.0.0</revision>
        <spring-cloud-alibaba.version>2021.0.4.0</spring-cloud-alibaba.version>
        <!-- Spring Cloud -->
        <spring.cloud.version>2021.0.5</spring.cloud.version>
        <!-- Spring Boot -->
        <spring-boot.version>2.7.7</spring-boot.version>
        <swagger.fox.version>3.0.0</swagger.fox.version>
        <fastjson.version>2.0.22</fastjson.version>
        <log-helper.version>1.0.5-RELEASE</log-helper.version>
        <jjwt.version>0.9.1</jjwt.version>
        <mysql-connector.version>5.1.18</mysql-connector.version>
        <mybatis.plus.version>3.5.2</mybatis.plus.version>
        <dynamic.datasource.version>3.5.1</dynamic.datasource.version>
        <druid.version>1.2.15</druid.version>
        <xxl-job.version>2.4.0</xxl-job.version>
        <lomnok.version>1.18.24</lomnok.version>
        <checkstyle-site.version>3.7.1</checkstyle-site.version>
        <checkstyle.version>3.1.0</checkstyle.version>
        <flatten.version>1.1.0</flatten.version>
        <owasp.version>8.1.0</owasp.version>
        <elastic-job.version>3.0.3</elastic-job.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
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
    # 分布式任务Elastic Job模块
    ├── ruuby-module-elasticjob
    # 全链路灰度发布组件
    ├── ruuby-module-graysacle
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

