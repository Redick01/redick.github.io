# 通过MySQL驱动拦截器实现执行sql耗时计算 <!-- {docsify-ignore-all} -->


## 背景

&nbsp; &nbsp; 公司的一个需求，公司既有的链路追踪日志组件要支持MySQL的sql执行时间打印，要实现链路追踪常用的手段就是实现第三方框架或工具提供的拦截器接口或者是过滤器接口，对于MySQL也不例外，实际上就是实现了MySQL驱动的拦截器接口而已。


## 具体实现

&nbsp; &nbsp; MySQL的渠道有不同的版本，不同版本的拦截器接口是不同的，所以要针对你所使用的不同版本的MySQL驱动去实现响应的拦截器，接下来分别介绍下MySQL渠道5,6,8版本的实现方式。


### MySQL5

&nbsp; &nbsp; 这里以MySQL渠道5.1.18版本为例实现，实现`StatementInterceptorV2`接口，主要实现逻辑在`preProcess`和`postProcess`方法，这两个方法是sql执行前后要执行的方法，我所使用的框架是logback，这里使用MDC来记录sql执行前的一个时间戳，代码在`postProcess`方法`MDC.put("sql_exec_time", start);`，自己也可以使用ThreadLocal等来实现，然后在`postProcess`方法中使用`MDC.get("sql_exec_time")`将记录的sql执行前的时间取出来，最后再用当前时间戳减去sql执行前的时间，就算出了sql执行的时间。

```java
import static net.logstash.logback.marker.Markers.append;

import com.mysql.jdbc.Connection;
import com.mysql.jdbc.ResultSetInternalMethods;
import com.mysql.jdbc.Statement;
import com.mysql.jdbc.StatementInterceptorV2;
import com.redick.util.LogUtil;
import java.sql.SQLException;
import java.util.Properties;
import lombok.extern.slf4j.Slf4j;
import org.slf4j.MDC;

/**
 * @author Redick01
 */
@Slf4j
public class Mysql5StatementInterceptor implements StatementInterceptorV2 {

    @Override
    public void init(Connection connection, Properties properties) throws SQLException {

    }

    @Override
    public ResultSetInternalMethods preProcess(String s, Statement statement, Connection connection)
            throws SQLException {
        String start = String.valueOf(System.currentTimeMillis());
        MDC.put("sql_exec_time", start);
        log.info(LogUtil.customizeMarker(LogUtil.kLOG_KEY_TRACE_TAG, "sql_exec_before"), "开始执行sql");
        return null;
    }

    @Override
    public boolean executeTopLevelOnly() {
        return false;
    }

    @Override
    public void destroy() {

    }

    @Override
    public ResultSetInternalMethods postProcess(String s, Statement statement,
            ResultSetInternalMethods resultSetInternalMethods, Connection connection, int i,
            boolean b, boolean b1, SQLException e) throws SQLException {
        long start = Long.parseLong(MDC.get("sql_exec_time"));
        long end = System.currentTimeMillis();
        log.info(LogUtil.customizeMarker(LogUtil.kLOG_KEY_TRACE_TAG, "sql_exec_after")
                .and(append(LogUtil.kLOG_KEY_SQL_EXEC_DURATION, end - start)), "结束执行sql");
        return null;
    }
}
```

### MySQL6

&nbsp; &nbsp; MySQL6和MySQL5基本一样，只是接口不是同一个，直接放代码

```java
import static net.logstash.logback.marker.Markers.append;

import com.mysql.cj.api.MysqlConnection;
import com.mysql.cj.api.jdbc.Statement;
import com.mysql.cj.api.jdbc.interceptors.StatementInterceptor;
import com.mysql.cj.api.log.Log;
import com.mysql.cj.api.mysqla.result.Resultset;
import com.redick.util.LogUtil;
import java.sql.SQLException;
import java.util.Properties;
import lombok.extern.slf4j.Slf4j;
import org.slf4j.MDC;

/**
 * @author Redick01
 */
@Slf4j
public class Mysql6StatementInterceptor implements StatementInterceptor {

    @Override
    public StatementInterceptor init(MysqlConnection mysqlConnection, Properties properties,
            Log log) {
        return null;
    }

    @Override
    public <T extends Resultset> T preProcess(String s, Statement statement) throws SQLException {
        String start = String.valueOf(System.currentTimeMillis());
        MDC.put("sql_exec_time", start);
        log.info(LogUtil.customizeMarker(LogUtil.kLOG_KEY_TRACE_TAG, "sql_exec_before"), "开始执行sql");
        return null;
    }

    @Override
    public boolean executeTopLevelOnly() {
        return false;
    }

    @Override
    public void destroy() {

    }

    @Override
    public <T extends Resultset> T postProcess(String s, Statement statement, T t, int i, boolean b,
            boolean b1, Exception e) throws SQLException {
        long start = Long.parseLong(MDC.get("sql_exec_time"));
        long end = System.currentTimeMillis();
        log.info(LogUtil.customizeMarker(LogUtil.kLOG_KEY_TRACE_TAG, "sql_exec_after")
                .and(append(LogUtil.kLOG_KEY_SQL_EXEC_DURATION, end - start)), "结束执行sql");
        return null;
    }
}
```

### MySQL8

&nbsp; &nbsp; MySQL8和MySQL5/6的拦截器接口又不一样了，MySQL8的拦截器接口是`com.mysql.cj.interceptors.QueryInterceptor`，统计sql执行时间的方式还是一样的，代码如下：

```java
import static net.logstash.logback.marker.Markers.append;

import com.mysql.cj.MysqlConnection;
import com.mysql.cj.Query;
import com.mysql.cj.interceptors.QueryInterceptor;
import com.mysql.cj.log.Log;
import com.mysql.cj.protocol.Resultset;
import com.mysql.cj.protocol.ServerSession;
import com.redick.util.LogUtil;
import java.util.Properties;
import java.util.function.Supplier;
import lombok.extern.slf4j.Slf4j;
import org.slf4j.MDC;

/**
 * @author Redick01
 */
@Slf4j
public class Mysql8QueryInterceptor implements QueryInterceptor {

    @Override
    public QueryInterceptor init(MysqlConnection mysqlConnection, Properties properties, Log log) {
        return null;
    }

    @Override
    public <T extends Resultset> T preProcess(Supplier<String> supplier, Query query) {
        String start = String.valueOf(System.currentTimeMillis());
        MDC.put("sql_exec_time", start);
        log.info(LogUtil.customizeMarker(LogUtil.kLOG_KEY_TRACE_TAG, "sql_exec_before"), "开始执行sql");
        return null;
    }

    @Override
    public boolean executeTopLevelOnly() {
        return false;
    }

    @Override
    public void destroy() {

    }

    @Override
    public <T extends Resultset> T postProcess(Supplier<String> supplier, Query query, T t,
            ServerSession serverSession) {
        long start = Long.parseLong(MDC.get("sql_exec_time"));
        long end = System.currentTimeMillis();
        log.info(LogUtil.customizeMarker(LogUtil.kLOG_KEY_TRACE_TAG, "sql_exec_after")
                .and(append(LogUtil.kLOG_KEY_SQL_EXEC_DURATION, end - start)), "结束执行sql");
        return null;
    }
}
```

### 使用方法

&nbsp; &nbsp; MySQL5和6的使用方式一样，在数据库链接的url中增加如下`statementInterceptors`参数，例如：

```yml
    url: jdbc:mysql://127.0.0.1:3316/log-helper?useUnicode=true&characterEncoding=UTF8&statementInterceptors=com.redick.support.mysql.Mysql5StatementInterceptor&serverTimezone=CST
```

&nbsp; &nbsp; MySQL8则是在url中增加`queryInterceptors`参数，例如：

```yml
    url: jdbc:mysql://127.0.0.1:3316/log-helper?useUnicode=true&characterEncoding=UTF8&queryInterceptors=com.redick.support.mysql.Mysql8QueryInterceptor&serverTimezone=CST
```

### 测试结果

&nbsp; &nbsp; sql执行前日志

```json
{"@timestamp":"2023-02-28T17:16:29.234+08:00","@version":"0.0.1","message":"开始执行sql","logger_name":"com.redick.support.mysql.Mysql5StatementInterceptor","thread_name":"http-nio-3321-exec-4","level":"INFO","level_value":20000,"traceId":"9ed930dc-4cc6-4719-bf33-9fcb618fd65b","spanId":"1","request_type":"getName","parentId":"0","trace_tag":"sql_exec_before"}
```

&nbsp; &nbsp; sql执行后日志，sql_duration标识执行sql耗时3ms

```json
{"@timestamp":"2023-02-28T17:16:29.237+08:00","@version":"0.0.1","message":"结束执行sql","logger_name":"com.redick.support.mysql.Mysql5StatementInterceptor","thread_name":"http-nio-3321-exec-4","level":"INFO","level_value":20000,"traceId":"9ed930dc-4cc6-4719-bf33-9fcb618fd65b","spanId":"1","request_type":"getName","parentId":"0","trace_tag":"sql_exec_after","sql_duration":3}
```

具体实现代码参考[基于Spring AOP + logstash-logback-encoder日志链路追踪工具LogHelper](https://github.com/Redick01/log-helper)