---
layout: post
title: "spring cloud alibaba naocs使用详解"
date: 2021-02-02 16:42:30
categories: Spring-Cloud
keywords: "spring cloud alibaba naocs"
description: "spring-cloud-alibaba-naocs使用详解"
---

## 一、Naocs简介

​	Nacos是阿里巴巴开源的一款支持服务注册与发现，配置管理以及微服务管理的组件。用来取代以前常用的注册中心（zookeeper , eureka等等），以及配置中心（spring cloud config等等）。Nacos是集成了注册中心和配置中心的功能，做到了二合一。

**Nacos 的关键特性包括**

- 服务发现和服务健康监测
- 动态配置服务
- 动态DNS服务
- 服务及其元数据管理

## 二、Naocs安装部署

### 单机部署

第一步：运行`conf/nacos-mysql.sql`文件【默认nacos用嵌入式数据库derby，需切换为MySQL】

第二步：`修改conf/application.properties文件`

```properties
spring.datasource.platform=mysql

db.num=1
db.url.0=jdbc:mysql://11.162.196.16:3306/nacos_devtest?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&serverTimezone=Asia/Shanghai
db.user=nacos_devtest
db.password=youdontknow
```

第三步：修改`conf/cluster.conf`，配置成`ip:port`格式

```conf
# ip:port
200.8.9.16:8848
200.8.9.17:8848
200.8.9.18:8848
```

### 集群部署

因为每个nacos都是读取的MySQL的信息，所有数据是一样的，只需要前面挂上Nginx实现负载均衡即可实现集群模式

<img src="/img/nacos/nacos-集群.png" alt="nacos-集群" style="zoom:50%;" />

## 三、Naocs作为服务注册中心

**第一步：添加依赖**

```xml
<dependency>
  <groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

**第二步：配置YAML**

```yaml
spring:
  application:
    name: cloud-order-service
  cloud:
    nacos:
      # nacos地址
      server-addr: localhost:8848
      discovery:
        # namespace的区分
        namespace: DEV
        # 所属组
        group: WYK_GROUP
        # 注册的服务名称，默认即为${spring.application.name}
        service: ${spring.application.name}
        # 服务集群名称，默认DEFAULT
        cluster-name: WYKCLUSTER
```

**第三步：主启动类**

```java
@SpringBootApplication
@EnableDiscoveryClient
public class PaymentApplication8005 {

  public static void main(String[] args) {
    SpringApplication.run(PaymentApplication8005.class);
  }
}
```

**第四步：配置路由，nacos使用的是ribbon<span style="color:blue">【如果使用Open Feign可以跳过】</span>**

```java
@Configuration
public class RestTemplateConfig {

  @Bean
  @LoadBalanced
  public RestTemplate restTemplate() {
    return new RestTemplate();
  }
}
```

## 四、Naocs作为配置中心

**第一步：添加依赖**

```xml
<dependency>
  <groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

**第二步：配置YAML<span style="color:red">【Bootstrap文件】</span>**

```yaml
spring:
  cloud:
    nacos:
      # nacos地址
      server-addr: localhost:8848
      config:
        # 文件后缀
        file-extension: yaml
        # namespace的区分 
        namespace: DEV
        # 所属组
        group: WYK_GROUP
        # 服务集群名称，默认DEFAULT
        cluster-name: WYKCLUSTER
        # Data ID
        name: payment-dev
```

还可以使用<span style="color:blue">`spring.cloud.config.prefix`</span>+<span style="color:blue">`-`</span>+<span style="color:blue">`spring.profiles.active`</span>+<span style="color:blue">`.`</span>+<span style="color:blue">`spring.cloud.config.file-extension`</span>的组合

`spring.cloud.config.prefix`默认值为`applicationName`

## 五、几种注册中心的比较

|                  | nacos                      | eureka      | consul            | zookeeper  |
| ---------------- | -------------------------- | ----------- | ----------------- | ---------- |
| **一致性协议**   | AP / CP                    | AP          | CP                | CP         |
| **健康检查**     | TCP/HTTP/MYSQL/Client Beat | Client Beat | TCP/HTTP/gRPC/Cmd | Keep Alive |
| **雪崩保护**     | 支持                       | 支持        | 不支持            | 不支持     |
| **自动注销实例** | 支持                       | 支持        | 不支持            | 支持       |
| **访问协议**     | HTTP/DNS/UDP               | HTTP        | HTTP/DNS          | TCP        |
| **监听支持**     | 支持                       | 支持        | 支持              | 支持       |
| **多数据中心**   | 支持                       | 支持        | 支持              | 不支持     |
| **跨注册中心**   | 支持                       | 不支持      | 支持              | 不支持     |
| **k8s集成**      | 支持                       | 不支持      | 不支持            | 支持       |

一般来说，如果不需要存储服务级别的信息且服务实例是通过nacos-client注册，并能够保持心跳上报，那么就可以选择AP模式。当前主流的服务如SpringCloud和Dubbo服务，都适用与AP模式，AP模式为了服务的可能性而减弱了一致性，因此AP模式下只支持注册临时实例。如果需要在服务级别编辑或存储配置信息，那么CP是必须，K8S服务和DNS服务则适用于CP模式。CP模式下则支持注册持久化实例，此时则是以Raft协议为集群运行模式，该模式下注册实例之前必须先注册服务，如果服务不出存在，则会返回错误。