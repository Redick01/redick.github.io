# Spring Cloud Alibaba-生产安全问题 <!-- {docsify-ignore-all} -->

## Spring Cloud Gateway集成Actuator的安全漏洞和解决方案

  Spring Cloud Gateway 远程代码执行漏洞（CVE-2022-22947）。使用 Spring Cloud Gateway 的应用如果对外暴露了 Gateway Actuator 接口，
则可能存在被 CVE-2022-22947 漏洞利用的风险，攻击者可通过利用此漏洞执行 SpEL 表达式，从而在目标服务器上执行任意恶意代码，获取系统权限。

### 影响范围

- Spring Cloud Gateway 3.1.x < 3.1.1
- Spring Cloud Gateway 3.0.x < 3.0.7

### 解决办法（任选其一）
1. 通过management.endpoint.gateway.enable：false配置将其禁用Actuator
2. 如果需要Actuator端点，则应使用Spring Security对其进行保护
3. 引入Spring-Security模块，配置访问权限验证，访问Actuator接口时需要登录
4. Actuator访问接口使用独立端口，并配置不对外网开放

## 生产环境关闭Swagger

```yaml
springfox:
  documentation:
    enabled: false
    auto-startup: false
    swagger-ui:
      enabled: false
```
