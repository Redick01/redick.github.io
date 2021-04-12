# Arthas Idea 插件

一、背景

&emsp;&emsp; 目前Arthas 官方的工具还不够足够的简单，需要记住一些命令，特别是一些扩展性特别强的高级语法，比如ognl获取spring context 为所欲为，watch、trace 不够简单，需要构造一些命令工具的信息，因此只需要一个能够简单处理字符串信息的插件即可使用。当在处理线上问题的时候需要最快速、最便捷的命令，因此Idea arthas 插件还是有存在的意义和价值的。使用文档的例子应该是比较老，随着版本的迭代功能融合命令的名称有过修改，但是功能未变化,，体验demo 是根据2.10 版本编写的，将整个arthas idea plugin 插件的创作初衷以及一些使用的最好例子展现了出来，为啥arthas idea plugin 会存在
变更记录: https://www.yuque.com/wangji-yunque/ikhsmq/pa45y2  后续收集的资料全部分享到这个 语雀文档维护，当前这个是一个分享链接。

**将光标放置在具体的类、字段、方法上面 右键选择需要执行的命令，部分会有窗口弹出、根据界面操作获取命令；部分直接获取命令复制到了剪切板 ，自己启动arthas 后粘贴即可执行。**

to 新手：
     对于使用arthas的新手同学 个人建议先去了解arthas的基本用法 arthas 官方的线上实战教程 、官方的基础命令，然后在使用arthas idea plugin 体验一下带来的优势，刚刚学习对于环境安装、命令使用还不熟悉，先熟悉了在使用插件，这样才能更好利用好插件去解决一些问题，方便使用高级功能、简化记忆和流程操作，这些基本功还需要自己先掌握。本插件也是在不断理解和学习的基础上创造的，需要有一定的arthas 使用的基础。

历史参加征文

[阿里开源的那个牛X的问题排查工具——Arthas，推出IDEA插件了！ ](https://developer.aliyun.com/article/756842)

[阿里开源那个牛哄哄问题排查工具竟然不会用？最佳实践来了！](https://juejin.im/post/6850418120198619144)



### 1.1  体验demo 地址 

[爱上Java诊断利器Arthas之Arthas idea plugin 的前世今生 体验demo](https://github.com/WangJi92/arthas-plugin-demo)  参加arthas 征文活动，

### 1.2 idea plugin github issue 

https://github.com/WangJi92/arthas-idea-plugin/issues

这里会增加一些使用的功能的说明，小编有时候会记录一下问题，方便查看，相当于笔记吧。

### 1.3 最新版本

2.10 支持 watch、tt 获取spring context.getBean 功能通用化

static spring context 一次直接调用

[watch get spring context 要触发一次调用http请求后才能触发](https://github.com/WangJi92/arthas-idea-plugin/issues/5)

[tt get spring context ](https://github.com/alibaba/arthas/issues/482)需要记录一次tt(缓存调用的方法的参数 method 后续反射调用即可) 后续都可以直接使用

将这种获取直接调用方法、字段 标准化非常便于使用。之前没有写在文档上面 这里记录一下// 2020-06-22

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1592927201723-28a5d6ed-5722-4de1-85a2-b0b2c361f5eb.png?x-oss-process=image%2Fresize%2Cw_1500)

2.11 支持功能  [Logger 日志级别动态更新](https://github.com/WangJi92/arthas-idea-plugin/issues/7)

2.12 watch 功能增强支持watch放置在字段上观察值 // 2020-06-22

2.13 [火焰图功能支持 ](https://wangji.blog.csdn.net/article/details/106934179) // 2020-06-23

功能总结 [核心变量+异步任务](https://blog.csdn.net/u012881904/article/details/105603047)

[arthas 入门最佳实践](https://wangji.blog.csdn.net/article/details/106964278) // 2020-07-05

2.14 优化高版本右键展示问题2020版本，会导致右键命令在最下面去了 //2020-08-01

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1596292749279-78e06cc7-bf70-499d-b378-d97f47d3480d.png?x-oss-process=image%2Fresize%2Cw_1500)



增加 getstatic 这个命令，相对ognl 获取静态变量更加轻量级,用起来简单

```
getstatic com.wangji92.arthas.plugin.demo.controller.StaticTest INVOKE_STATIC_LONG -x 3
```

ognl 的这种更加复杂，需要先获取一次classloader的值，但是如果有两个classloader 加载这个字段，上面需要添加-c 参数才可以。

```
ognl  -x  3 '@com.wangji92.arthas.plugin.demo.controller.StaticTest@INVOKE_STATIC_LONG' -c 316bc132
```

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1596292904663-2a60b649-ba82-49d6-8a72-0fbb008636bb.png?x-oss-process=image%2Fresize%2Cw_1500)

trace 命令默认增加  --skipJDKMethod false 不熟悉的同学查到时间耗死很多但是没有展示，其实被默认的jdk方法执行导致的。

```
trace com.wangji92.arthas.plugin.demo.controller.CommonController watchField -n 5 '1==1' --skipJDKMethod false
```

# 二、支持的功能

支持的功能都是平时处理最常用的一些功能，一些快捷的链接，在处理紧急问题时候不需要到处查找，都是一些基本的功能,自动复制到剪切板中去，方便快捷。

![arthas command.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1585491342968-f2b3e2fe-2f3c-4ac6-bd3c-16de2230ef9d.png?x-oss-process=image%2Fresize%2Cw_1500)

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1592927432866-9cbe98c0-ff53-4b5e-a0d0-8dd795cc8956.png)





## 2.1 watch

![image.png](https://cdn.nlark.com/yuque/0/2019/png/171220/1577111133387-260aa3ff-1a8d-441f-ab5a-3786c585e618.png?x-oss-process=image%2Fresize%2Cw_1500)

```
watch com.command.idea.plugin.utils.StringUtils toLowerFristChar '{params,returnObj,throwExp}' -n 5 -x 3
```



## 2.2 trace 

### 2.2.1 基本trace 

```
trace com.command.idea.plugin.utils.StringUtils toLowerFristChar -n 5
```



### 2.2.2 trace -E

> arthas 2.1 版本新增功能

trace命令只会trace匹配到的函数里的子调用，并不会向下trace多层。因为trace是代价比较贵的，多层trace可能会导致最终要trace的类和函数非常多。因此Arthas 官方支持 trace -E 特殊获取多个，该插件支持一下trace -E



![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1578319600258-72f7e2a4-f7b1-4d48-a17a-b18e11070578.png)



```
trace -E com.github.wangji92.arthas.plugin.utils.ClipboardUtils|com.github.wangji92.arthas.plugin.utils.OgnlPsUtils getClassBeanName|setClipboardString -n 5
```



![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1578319724852-bf26109a-f9a4-441b-9593-ecf02e74f8f4.png)

## 2.3 static ognl (字段或者方法)

![image.png](https://cdn.nlark.com/yuque/0/2019/png/171220/1577111229892-122cacbb-1f25-47cc-bab2-df5cc10cebc0.png)



### 2.3.1 右键static ognl

### 2.3.2 获取classload命令

必须要获取，不然会找不到classload，arthas 官方获取问题系统的classload，spring 项目应该无法获取到这个class的信息，因此首先执行一下这个命令



```
sc -d com.command.idea.plugin.utils.StringUtils
```



![image.png](https://cdn.nlark.com/yuque/0/2019/png/171220/1577111475711-5ffa5492-9099-4b50-9ac0-18b03631f03c.png)



### 2.3.2 复制到界面，获取命令，执行即可

![image.png](https://cdn.nlark.com/yuque/0/2019/png/171220/1577111519442-aee9d985-5d0c-4f67-a8fc-bd4f2d9c3ff5.png)



```
ognl  -x  3  '@com.command.idea.plugin.utils.StringUtils@toLowerFristChar(" ")' -c 8bed358
```



## 2.4 Static Spring Context (需要配置静态spring context)

 实际上就是根据当前的spring项目中的获取静态的spring context这样可以直接根据这个context直接获取任何的Bean方法，一般在Java后端服务中都有这样的Utils类，因此这个可以看为一个常量! 可以参考:http://www.dcalabresi.com/blog/java/spring-context-static-class/ 有了这个，我们可以跟进一步的进行数据简化，由于在idea这个环节中，可以获取方法参数，spring bean的名称等等，非常的方便。



```
public class ApplicationContextProvider implements ApplicationContextAware {
    
    private static ApplicationContext context;
 
    public ApplicationContext getApplicationContext() {
        return context;
    }
 
    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        context = ctx;
    }
}
```

### 2.4.1 设置获取spring context的上下文

> ps 这里可以使用@applicationContextProvider@context

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1598149583905-0010df25-4357-4040-a76b-c01db11d0f90.png?x-oss-process=image%2Fresize%2Cw_1500)

**arthas idea plugin 配置** https://www.yuque.com/wangji-yunque/ikhsmq/ugrc8n  //2020-08-23

### 2.4.2 右键点击需要调用的方法

这里的策略和static ognl 一样的，本质还是ognl的调用。

![image.png](https://cdn.nlark.com/yuque/0/2019/png/171220/1577112169623-5535dd8a-47d2-4fbb-8631-f785406c6057.png?x-oss-process=image%2Fresize%2Cw_1500)



```
ognl  -x  3  '#springContext=@applicationContextProvider@context,#springContext.getBean("arthasInstallCommandAction").actionPerformed(new com.intellij.openapi.actionSystem.AnActionEvent())' -c desw22
```



## 2.5  install(linux)

安装脚本，可以一键的通过as.sh 进行执行

实际上就是 java -jar ~/.arthas-boot.jar

```
curl -sk https://arthas.aliyun.com/arthas-boot.jar -o ~/.arthas-boot.jar  && echo "alias as.sh='java -jar ~/.arthas-boot.jar --repo-mirror aliyun --use-http 2>&1'" >> ~/.bashrc && source ~/.bashrc && echo "source ~/.bashrc" >> ~/.bash_profile && source ~/.bash_profile
```



![image.png](https://cdn.nlark.com/yuque/0/2019/png/171220/1577112830008-8b7d0714-f37d-4425-b343-c7318eaf51ef.png?x-oss-process=image%2Fresize%2Cw_1500)



## 2.6 常用特殊用法链接

- [special ognl](https://github.com/alibaba/arthas/issues/71)
- [tt get spring context](https://github.com/alibaba/arthas/issues/482)
- [get error filter](https://github.com/alibaba/arthas/issues/429)
- [dubbo 遇上arthas](http://hengyunabc.github.io/dubbo-meet-arthas/)
- [Arthas实践--jad/mc/redefine线上热更新一条龙](http://hengyunabc.github.io/arthas-online-hotswap/)
- [ognl 使用姿势](https://blog.csdn.net/u010634066/article/details/101013479)



> arthas 2.3 版本新增功能

## 2.7 monitor

方法执行监控(性能问题排查，一段时间内的性能指标)  --cycle 10 10 秒统计一次

```
monitor com.wj.study.demo.generator.ControllerTest wangji -n 10 --cycle 10
```



![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1578757262270-5d83a980-a950-4761-9e2b-a27326cf6482.png?x-oss-process=image%2Fresize%2Cw_1500)

## 2.8 stack

获取方法从哪里执行的调用栈(用途：源码学习调用堆栈，了解调用流程)



```
[arthas@12293]$ stack com.wj.study.demo.generator.ControllerTest wangji -n 5
Press Q or Ctrl+C to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 33 ms.
ts=2020-01-11 23:38:35;thread_name=http-nio-8080-exec-3;id=1e;is_daemon=true;priority=5;TCCL=org.springframework.boot.web.embedded.tomcat.TomcatEmbeddedWebappClassLoader@50a691d3
    @com.wj.study.demo.generator.ControllerTest.wangji()
        at sun.reflect.NativeMethodAccessorImpl.invoke0(NativeMethodAccessorImpl.java:-2)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:498)
        at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:190)
        at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:138)
        at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:106)
        at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:888)
        at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:793)
        at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87)
        at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1040)
        at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:943)
        at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:1006)
        at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:898)
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:634)
        at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:883)
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:741)
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:231)
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
        at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:53)
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
```



> arthas 2.4 版本新增功能

## 2.9 Time Tunnel 

方法执行数据的时空隧道，记录下指定方法每次调用的入参和返回信息，并能对这些不同的时间下调用进行观测(可以重新触发，周期触发，唯一缺点对于ThreadLocal 信息丢失[隐含参数]、引用对象数据变更无效)

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1578756609330-f134d8f3-5a2c-46d8-b8a3-23ad890aa376.png)

### 2.9.1 跟踪记录

```
tt -t com.wj.study.demo.generator.ControllerTest wangji -n 5
```

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1578756720497-ad243a3f-67ce-4dff-83ed-bb34791daad4.png?x-oss-process=image%2Fresize%2Cw_1500)



### 2.9.2 q 退出跟踪

访问接口一次  q 退出，可以通过 tt -l 获取时光记录列表

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1578756769226-8ef63dec-4f67-40f9-9889-dc319596d348.png?x-oss-process=image%2Fresize%2Cw_1500)



### 2.9.3 重新触发一次



```
tt -p -i 1000
```



![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1578756827465-ca279fce-88f2-431a-b1d7-a13ffc5cb7eb.png?x-oss-process=image%2Fresize%2Cw_1500)



### 2.9.4 获取入参、返回值



```
tt  -w '{method.name,params,returnObj,throwExp}' -x 3 -i 1000
```

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1578756966803-a262d73d-6885-4cbd-a0b6-21ae96938983.png?x-oss-process=image%2Fresize%2Cw_1500)



### 2.9.5 周期性触发 3次



```
tt -p --replay-times 3 --replay-interval 2000 -i 1000
```

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1578757062376-480cc08b-3c17-4006-b49c-47beb855f7f4.png)

### 2.9.6 全部命令

```
执行一次TT列表中的1000 tt -p -i 1000  
查看第一个参数 tt  -w params[0] -i 1000 
查看方法执行参数 tt  -w '{method.name,params,returnObj,throwExp}' -x 3 -i 1000
周期性执行 tt -p --replay-times 3 --replay-interval 2000 -i 1000
时光隧道列表 tt -l
删除时光隧道列表 tt --delete-all
```



> arthas 2.5 版本新增功能

## 2.10 Ognl Set Static Field

 参考： https://github.com/alibaba/arthas/issues/641  设置Static Field，这里没有考虑复杂对象，支持简单的参数值得设置，通过反射处理。

```
ognl -x 3  '#field=@com.wj.study.demo.generator.ControllerTest@class.getDeclaredField("aLong"),#field.setAccessible(true),#field.set(null,0L)' -c desw22
```



![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1579270806872-919e185d-b15c-4a01-934f-89d613c8c4ae.png)



> arthas 2.6  版本新增功能

## 2.11 **Ognl get selected spring property**

**更加方便线上查询某个配置项是否生效，是否已经更新生效，特别是不确定的情况**

#### 脚本

```
ognl -x 3 '#springContext=@com.wj.study.demo.generator.ApplicationContextProvider@context,#springContext.getEnvironment().getProperty("wangji.text")' -c 18b4aac2
```

#### 操作图

**![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1579621877425-9fc26cce-5de6-4614-9b1f-ba82f64e5fc4.png?x-oss-process=image%2Fresize%2Cw_1500)**

**
**

**![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1579621904551-cff12552-b4ad-4155-afec-3e0d61d5ce70.png)**

#### 执行效果

**![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1579621937836-c9570b3a-2f42-4efd-8c5c-a8dc39d87ed9.png?x-oss-process=image%2Fresize%2Cw_1500)**

**
**

## 2.12 **Ognl get all spring property**

**获取spring 中全部的配置项信息，方便分析问题。**

#### 脚本

```
ognl -x 3 '#springContext=@com.wj.study.demo.generator.ApplicationContextProvider@context,#allProperties={},#standardServletEnvironment=#propertySourceIterator=#springContext.getEnvironment(),#propertySourceIterator=#standardServletEnvironment.getPropertySources().iterator(),#propertySourceIterator.{#key=#this.getName(),#allProperties.add("                "),#allProperties.add("------------------------- name:"+#key),#this.getSource() instanceof java.util.Map ?#this.getSource().entrySet().iterator.{#key=#this.key,#allProperties.add(#key+"="+#standardServletEnvironment.getProperty(#key))}:#{}},#allProperties' -c 18b4aac2
```

#### 操作图



![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1579621961026-678f1f86-ec93-42d0-8177-178072d8d4b6.png?x-oss-process=image%2Fresize%2Cw_1500)

#### 效果图



![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1579622004554-5c4122b1-f5c0-4e3b-b8fe-1f8224cbfaea.png?x-oss-process=image%2Fresize%2Cw_1500)



> 2.7 支持功能 

## 2.13 支持设置 static final变量的值

#### 脚本



```
ognl -x 3  '#field=@com.wj.study.demo.generator.ControllerTest@class.getDeclaredField("final_text"),#modifiers=#field.getClass().getDeclaredField("modifiers"),#modifiers.setAccessible(true),#modifiers.setInt(#field,#field.getModifiers() & ~@java.lang.reflect.Modifier@FINAL),#field.setAccessible(true),#field.set(null,"2222")' -c 18b4aac2
```



#### 操作图

##### 选中需要设置的值

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1583509622738-8c595020-31ef-4175-9c0f-35b91cf2fba7.png?x-oss-process=image%2Fresize%2Cw_1500)



##### 在ognl 表达式中#filed.set(null,""） 这里输入修改的值

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1583509682789-69f6c3ac-43e0-43a3-84db-db534c9ccb9a.png)

##### 获取classloader的值,先启动arthas 

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1583509942871-a43b0e46-6625-4d8b-9f71-d9a255718bb0.png)

sc -d com.wj.study.demo.generator.ControllerTest



![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1583509993082-74400ffb-edec-4979-b9c4-79b16ad260fd.png?x-oss-process=image%2Fresize%2Cw_1500)

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1583510021486-37075b3e-664a-4956-978c-e0aa6199f098.png)

##### 设置值

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1583510090108-2fb5ca88-ce2f-4cdb-b91a-c9350448ad24.png?x-oss-process=image%2Fresize%2Cw_1500)

##### 查看结果

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1583510202265-31033334-b782-42c6-aa31-13b62eddc7b9.png)

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1583510163642-a2338161-0cba-4652-99d9-ceae28f65114.png?x-oss-process=image%2Fresize%2Cw_1500)

github 说明： https://github.com/WangJi92/arthas-idea-plugin/issues/1



> 2.8 代码是否生效？反编译代码比较

## 2.14 代码是否生效？反编译代码比较

#### 脚本



```
jad --source-only com.wangji92.github.study.controller.FileOperationController downLoadByType
```

#### 操作



![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1583510370968-d38dc50d-faf5-4932-8120-1565cd575f5d.png?x-oss-process=image%2Fresize%2Cw_1500)

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1583510427048-c04a3776-3160-4661-950f-273af4758e5f.png?x-oss-process=image%2Fresize%2Cw_1500)



#### github 说明：https://github.com/WangJi92/arthas-idea-plugin/issues/2

> 2.12 watch 功能增强，方法watch 获取成员变量，将光标放置在watch 类的字段上即可使用

## 2.15 watch 功能增强 放置在字段上可以在watch的时候获取值

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1592926313750-4445fa89-9a8b-4c5d-9a9b-19e2f5c8226e.png?x-oss-process=image%2Fresize%2Cw_1500)

// 当获取某个非静态的字段的时候，只能是调用构造、或者非静态方法才可以，这里增加一个表达式判断逻辑

// 默认为 1== 1

// 如果调用静态方法 获取 target.xxxField 这样会报错的~ 这里要添加一个条件限制一下

// 具体可以参考 src/main/java/com/taobao/arthas/core/advisor/ArthasMethod.java

```
watch com.wangji92.arthas.plugin.demo.controller.StaticTest * '{params,returnObj,throwExp,target.filedValue}' -n 5 -x 3 'method.initMethod(),method.constructor!=null || !@java.lang.reflect.Modifier@isStatic(method.method.getModifiers())'
```

静态字段就比较简单

想在watch的同时获取值的信息，不过也可以直接使用get static 通过ognl 获取哦~

```
watch com.wangji92.arthas.plugin.demo.controller.StaticTest * '{params,returnObj,throwExp,@com.wangji92.arthas.plugin.demo.controller.StaticTest@INVOKE_STATIC_LONG}' -n 5 -x 3 '1==1'
```

> 2.13 版本 支持火焰图 async-profiler

## 2.16 火焰图命令支持 async-profiler

可以查看一下我收集一天的资料 https://wangji.blog.csdn.net/article/details/106934179

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1592926631469-baff9033-a88d-4399-a7ee-ded412557cab.png?x-oss-process=image%2Fresize%2Cw_1500)

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1592926653815-b86a1781-1495-444b-b3fd-0da86ad4d7eb.png?x-oss-process=image%2Fresize%2Cw_1500)

## 2.17 classload load class

用指定的classloader 加载class ,可能某个时候 search all the classes loaded by jvm（sc）不存在需要手动的去添加。整个流程化处理 不用记忆 //2020-08-23

https://www.yuque.com/wangji-yunque/ikhsmq/mq5enc

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1598149760558-315214bc-d082-451e-9568-8f6051f9971c.png)

## 2.18 Hot Swap Redefine

支持多选、单选、内部类、匿名类，直接到服务器质任何目录执行脚本和执行的脚本不一样，这个采用自动执行逻辑。  --select  -c 语法糖提供支持。更多详情见文档 https://www.yuque.com/wangji-yunque/ikhsmq/pwxhb4    //2020-08-23

```
$HOME/opt/arthas/as.sh --select com.wangji92.arthas.plugin.demo.ArthasPluginDemoApplication -c "redefine $HOME/opt/arthas/redefine/classes/com/wangji92/arthas/plugin/demo/controller/LoggerDemo.class" >$HOME/opt/arthas/redefine/redefineArthas.out
```

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1598149868467-3bab9804-1433-47c8-a66b-c2b6cb0a1923.png)

可以在version control 选择、project tree 选择、也可以在类文件编辑器中选择 方法、字段、匿名类、内部类都支持，选择范围越大，更新越多。

![image.png](https://cdn.nlark.com/yuque/0/2020/png/171220/1598150662130-b7f464d3-b334-4c98-9c01-baf3bc261389.png?x-oss-process=image%2Fresize%2Cw_1500)

## 2.19 Local File Upload To Oss

由于上面配置了oss 随带增加了一个上传本地文件到oss的功能，方便上传特殊的脚本到指定的服务器处理。  //2020-08-23

## 2.20 匿名类和内部类的支持

https://www.yuque.com/wangji-yunque/ikhsmq/yire5x   //2020-08-23

## 2.21 arthas idea plugin支持被代理的目标对象

ognl 表达式学习起来很奇妙, sc -d  xxxClass 获取所有的当前类相关的类 ，然后通过jad 反编译查看 源码，这样理解一些开源框架非常方便。

https://www.yuque.com/wangji-yunque/ikhsmq/uldktc

https://www.yuque.com/wangji-yunque/ikhsmq/ffk3pw

# 三、其他

## 代码地址

https://github.com/WangJi92/arthas-idea-plugin

# 四、下载地址

 https://plugins.jetbrains.com/plugin/13581-arthas-idea，直接搜索arthas 即可下载



# 五、视频介绍

https://space.bilibili.com/60408393



# 六、了解更多

 csdn[ 汪小哥](https://wangji.blog.csdn.net/)