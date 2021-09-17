# Spring Bean生命周期 <!-- {docsify-ignore-all} -->

https://www.cnblogs.com/zrtqsk/p/3735273.html

https://www.jianshu.com/p/1dec08d290c1

https://blog.csdn.net/iechenyb/article/details/83788338

## Bean声明周期的四个阶段

- **实例化**
- **属性赋值**
- **初始化**
- **销毁**

&nbsp; &nbsp; 实例化和属性赋值可以对应构造方法和setter方法注入，初始化和销毁是用户自定义扩展的两个阶段。顺序是实例化-》属性赋值-》初始化-》销毁

