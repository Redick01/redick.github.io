# soul网关源码分析之发布接口到网关

## 目标

- Http API发布到soul网关源码分析
- Dubbo API发布到soul网关源码分析

## 概述

&nbsp; &nbsp; 业务系统在接口方法上加类似于`@SoulDubboClient`注解，就能够将需要被soul网关代理的接口发布到soul-admin和soul网关，本篇文章主要分析soul网关是如何做到的自动发布API。

##  Http API发布到soul网关源码分析


