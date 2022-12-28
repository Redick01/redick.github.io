# 解析pfx文件文件报java.security.InvalidKeyException: Illegal key size <!-- {docsify-ignore-all} -->


## 为什么会产生这样的错误？

我们做Java开发，或是Android开发，都会先在电脑上安装JDK(Java Development Kit) 并配置环境变量，JDK也就是 Java 语言的软件开发工具包，JDK中包含有JRE（Java Runtime Environment，即：Java运行环境），JRE中包括Java虚拟机（Java Virtual Machine）、Java核心类库和支持文件，而我们今天要说的主角就在Java的核心类库中。在Java的核心类库中有一个JCE（Java Cryptography Extension），JCE是一组包，它们提供用于加密、密钥生成和协商以及 Message Authentication Code（MAC）算法的框架和实现，所以这个是实现加密解密的重要类库。

在我们安装的JRE目录下有这样一个文件夹：%JAVE_HOME%\jre\lib\security（%JAVE_HOME%是自己电脑的Java路径，一版默认是：C:\Program Files\Java，具体看自己当时安装JDK和JRE时选择的路径是什么），其中包含有两个.jar文件：“local_policy.jar ”和“US_export_policy.jar”，也就是我们平时说的jar包，再通俗一点说就是Java中包含的类库（Sun公司的程序大牛封装的类库，供使用Java开发的程序员使用），这两个jar包就是我们JCE中的核心类库了。JRE中自带的“local_policy.jar ”和“US_export_policy.jar”是支持128位密钥的加密算法，而当我们要使用256位密钥算法的时候，已经超出它的范围，无法支持，所以才会报：“java.security.InvalidKeyException: Illegal key size or default parameters”的异常。那么我们怎么解决呢？

## 如何解决？

解决方案：去官方下载JCE无限制权限策略文件。

jdk 5: http://www.oracle.com/technetwork/java/javasebusiness/downloads/java-archive-downloads-java-plat-419418.html#jce_policy-1.5.0-oth-JPR

jdk6: http://www.oracle.com/technetwork/java/javase/downloads/jce-6-download-429243.html

JDK7的下载地址: http://www.oracle.com/technetwork/java/javase/downloads/jce-7-download-432124.html
JDK8的下载地址: http://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html 


下载 local_policy.jar和US_export_policy.jar 替换 jre/lib/security下的jar

