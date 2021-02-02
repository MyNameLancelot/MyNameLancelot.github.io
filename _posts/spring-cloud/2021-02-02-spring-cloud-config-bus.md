---
layout: post
title: "spring cloud config使用详解"
date: 2021-02-02 16:42:30
categories: Spring Cloud
keywords: "Spring Cloud Config,Spring Cloud Config使用详解 "
description: "Spring Cloud Config使用详解"
---

## 一、Spring Cloud Config简介

​	Spring Cloud Config项目是一个解决分布式系统的配置管理方案。它包含了Client和Server两个部分，Server提供配置文件的存储以接口的形式将配置文件的内容提供出去，Client通过接口获取数据并依据此数据初始化自己的应用。目前SpringCloud Config的Server主要是通过Git方式做一个配置中心，然后每个服务从Server获取自身配置所需的参数。

<img src="/img/spring-cloud-config/Spring-Cloud-Config.png" alt="Spring Cloud Config" style="zoom:67%;" />

## 二、Spring Cloud Config Server配置

**第一步：引入依赖**

```xml
<!-- Server 配置 -->
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

**第二步：配置YAML**

```yaml
server:
  port: 3000

spring:
  application:
    name: cloud-config-center
  cloud:
    config:
      server:
        git:
          # Git工程下的https地址
          uri: https://github.com/MyNameLancelot/spring-cloud-config.git
          # 工程名
          search-paths: spring-cloud-config
          # 默认拉取的分支，客户端可以手动设置
          default-label: master
          # 本地文件被污染时强制拉取
          force-pull: true
          # 需要在本地配置ssh的公钥,如果不配置需要忽略公钥检查
          strict-host-key-checking: false

management:
  endpoints:
    web:
      exposure:
        # 暴露消息总线刷新地址，用于bus刷新
        include: bus-refresh
```

**第三步：主启动类**

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigCenter3000 {

  public static void main(String[] args) {
    SpringApplication.run(ConfigCenter3000.class);
  }
}
```

## 三、Spring Cloud Config Client配置

**第一步：引入依赖**

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

**第二步：配置YAML<span style="color:red">[bootstrap.yaml]</span>**

```yaml
spring:
  cloud:
    config:
      # 使用的分支，覆盖server的默认分支
      label: master
      # 配置文件名称
      name: app
      # 配置文件环境
      profile: dev
      # server地址
      uri: http://localhost:3000
```

访问的Git文件为{label}分支下的{name}-{profile}.yaml或{name}-{profile}.properties

注意：使用`@RefreshScope`的Spring管理下的类才能实现自动刷新

## 四、Spring Cloud Bus简介

​	如果只使用Spring Cloud Config则需要自己手动一个个刷新，所以需要Bus进行总线消息通知，用于全局刷新和单节点刷新。SpringCloud Bus目前支持RabbitMQ和Kafka。

## 五、Spring Cloud Bus配置

**第一步：引入依赖**

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

**第二步：配置YAML【配置RabbitMQ或者Kafaka均可】**

```yaml
spring:
  cloud:
    bus:
      trace:
        # 总线事件跟踪，访问/trace可获得更新信息
        enabled: true
  rabbitmq:
    host: 192.168.2.189
    port: 5672
    password: guest
    username: guest
    virtual-host: /

management:
  endpoints:
    web:
      exposure:
        include: bus-refresh
```

## 六、Spring Cloud Bus说明

<span style="color:red">注意配置时必须要暴露`management.endpoints.web.exposure.include=bus-refresh`</span>

全局刷新即刷新所有客户端

```shell
curl -X POST http:/[config-server-ip]:[port]/actuator/bus-refresh
```

指定刷新

```shell
curl -X POST http:/[config-client-ip]:[port]/actuator/bus-refresh
```

