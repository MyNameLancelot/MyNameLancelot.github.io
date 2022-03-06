---
layout: post
title: "spring cloud eureka使用详解"
date: 2021-02-02 16:42:30
categories: Spring-Cloud
keywords: "eureka"
description: "eureka使用详解"
---

## 一、简介

​	Spring Cloud封装了Netflix公司开发的Eureka模块来实现服务注册和发现，Eureka采用了C-S的设计架构。Eureka Server 作为服务注册功能的服务器，它是服务注册中心。而系统中的其他微服务，作为Eureka 的客户端连接到Eureka Server并定时发送心跳。这样就可以通过Eureka Server来监控系统中各个微服务是否正常运行。Spring Cloud的一些其他模块就可以通过Eureka Server 来发现系统中的其他微服务。

<img src="/img/eureka/eureka架构图.png" alt="eureka架构图" style="zoom:67%;" />

## 二、Eureka Server搭建

### 单机版搭建

**一、导入依赖**

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
  </dependency>
</dependencies>
```

**二、配置YAML**

```yaml
server:
  port: 7001

spring:
  application:
    name: cloud-eureka-server

eureka:
  instance:
    # eureka服务端的名称
    hostname: localhost
  client:
    # 服务端不需要向注册中心注册自己
    register-with-eureka: false
    # 服务端不需要检索服务
    fetch-registry: false
    service-url:
      # 设置与Eureka Server交互的查询服务和注册服务的地址
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

**三、主启动类**

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaApplication7001 {

  public static void main(String[] args) {
    SpringApplication.run(EurekaApplication7001.class);
  }
}
```

### 集群版搭建

Eureka Server在启动时默认会注册自己，成为一个服务。所以Eureka Server也是一个客户端。也就是说们可以配置多个Eureka Server让他们之间相互注册。当服务提供者向其中一个Eureka注册服务时，这个服务就会被共享到其他Eureka上，这样所有的Eureka都会有相同的服务

**相对与单机版，集群版只需修改单机版的YAML**

```yaml
server:
  port: 7002

spring:
  application:
    name: cloud-eureka-server

eureka:
  instance:
    # eureka服务端的名称
    hostname: eureka7002.com
  client:
    # 服务端不需要向注册中心注册自己
    register-with-eureka: false
    # 服务端不需要检索服务
    fetch-registry: false
    service-url:
      # 与其它Eureka Server交互的查询服务和注册服务的地址，不用包含自己
      defaultZone: http://eureka7003.com:7003/eureka/,http://eureka7004.com:7004/eureka/
```

## 三、服务注册进Eureka

**第一步：主启动类**

```java
@SpringBootApplication
@EnableEurekaClient
public class PaymentApplication8001 {

  public static void main(String[] args) {
    SpringApplication.run(PaymentApplication8001.class);
  }
}
```

**第二步：配置YAML**

```yaml
spring:
  application:
    name: cloud-payment-service
eureka:
  client:
    # 需要向注册中心注册自己，默认true
    register-with-eureka: true
    # 需要检索服务，默认true
    fetch-registry: true
    service-url:
      # 设置与Eureka Server交互的查询服务和注册服务的地址
      defaultZone: http://eureka7002:7002/eureka/,http://eureka7003.com:7003/eureka/
  instance:
    # eureka面板上的实例名称
    instance-id: payment8001
    # 访问路径显示IP地址
    prefer-ip-address: true
```

![eureka注册显示](/img/eureka/eureka注册显示.png)

**四、消费者调用服务**

```java
@RestController
@RequiredArgsConstructor
@Slf4j
public class OrderController {

  // 调用服务，直接使用服务名称即可
  private static final String PROVIDER_URL = "http://CLOUD-PAYMENT-SERVICE";

  private final RestTemplate restTemplate;

  @GetMapping("/order/{id}")
  public CommentResult getOrder(@PathVariable("id") Long id) {
    log.info("查询订单{}", id);
    CommentResult commentResult = restTemplate.getForObject(PROVIDER_URL + "/payment/" + id, CommentResult.class);
    return commentResult;
  }
}

@Configuration
public class RestTemplateConfig {

  @Bean
  @LoadBalanced //必须加此注解开启负载均衡才能成功调用，
  public RestTemplate restTemplate() {
    return new RestTemplate();
  }
}
```

## 四、主动获取Eureka注册的服务

第一步：主启动类

```java
@SpringBootApplication
@EnableDiscoveryClient
public class PaymentApplication8001 {

  public static void main(String[] args) {
    SpringApplication.run(PaymentApplication8001.class);
  }
}
```

第二步：主动读取信息

```java
public class PaymentController {

  private final DiscoveryClient discoveryClient;
  
  @GetMapping("/discoveryClientInfo")
  public DiscoveryClient getDiscoveryClientInfo() {
    List<String> services = discoveryClient.getServices();
    for (String service : services) {
      log.info("service: {}", service);
      List<ServiceInstance> instances = discoveryClient.getInstances(service);
      for (ServiceInstance instance : instances) {
        log.info("instanceID={} instanceHost={} instancePort={}",
                 instance.getInstanceId(),instance.getHost(), instance.getPort());
      }
    }
    return discoveryClient;
  }
}
```

## 五、Eureka的自我保护机制

​	Eureka Server在运行期间会去统计心跳失败比例在15分钟之内是否低于85%，如果低于85%，Eureka Server会将这些实例保护起来，让这些实例不会过期，但是在保护期内如果服务刚好这个服务提供者非正常下线了，此时服务消费者就会拿到一个无效的服务实例，此时会调用失败，对于这个问题需要服务消费者端要有一些容错机制，如重试，断路器等。

<span style="color:red">自我保护模式被激活的条件是：在 1 分钟后，`Renews (last min) < Renews threshold`。</span>

- `Renews threshold`：**Eureka Server 期望每分钟收到客户端实例续约的总数**。

- `Renews (last min)`：**Eureka Server最后1分钟收到客户端实例续约的总数**。

解决方式有三种：

- 关闭自我保护模式（`eureka.server.enable-self-preservation`设为`false`），**不推荐**。
- 降低`renewalPercentThreshold`的比例（`eureka.server.renewal-percent-threshold`设置为`0.5`以下），**不推荐**。
- 部署多个Eureka Server并开启其客户端行为（`eureka.client.register-with-eureka`不要设为`false`，默认为`true`），**推荐**。