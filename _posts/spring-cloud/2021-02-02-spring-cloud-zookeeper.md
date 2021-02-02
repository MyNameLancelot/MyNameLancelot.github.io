---
layout: post
title: "spring cloud zookeeper使用详解"
date: 2021-02-02 16:42:30
categories: Spring-Cloud
keywords: "spring cloud zookeeper"
description: "spring-cloud-zookeeper使用详解"
---

## 一、简介

​	Zookeeper作为知名的分布式调度系统，我们也可以利用其作为配置中心。其wacth 主动通知机制， 可以将node节点数据变更信息及时通知到client端。<span style="color:red">【注册的服务节点为zookeeper的临时节点，即客户端退出节点删除】</span>

## 二、服务注册进Zookeeper

### 服务注册

**配置依赖**

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
</dependency>
```

**第一步：配置主启动类**

```java
@SpringBootApplication
@EnableDiscoveryClient
public class PaymentApplication8003 {

  public static void main(String[] args) {
    SpringApplication.run(PaymentApplication8003.class);
  }
}
```

**第二步：配置YAML**

```yaml
spring:
  application:
    name: cloud-payment-service
  cloud:
    zookeeper:
      connect-string: 192.168.22.160:2181
```

### 服务消费

**第一步：配置主启动类**

```java
@SpringBootApplication
@EnableDiscoveryClient
public class OrderApplication80 {

  public static void main(String[] args) {
    SpringApplication.run(OrderApplication80.class);
  }
}
```

**第二步：配置YAML**

```yaml
spring:
  application:
    name: cloud-order-service
  cloud:
    zookeeper:
      connect-string: 192.168.22.160:2181
```

**第三步：配置RestTemplate**

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

**第四步：调用服务**

```java
@RestController
@RequiredArgsConstructor
@Slf4j
public class OrderController {

  // 注意Eureka是大写，这个是小写
  private static final String PROVIDER_URL = "http://cloud-payment-service";

  private final RestTemplate restTemplate;

  @GetMapping("/order/{id}")
  public CommentResult getOrder(@PathVariable("id") Long id) {
    log.info("查询订单{}", id);
    CommentResult commentResult = restTemplate.getForObject(PROVIDER_URL + "/payment/" + id, CommentResult.class);
    return commentResult;
  }
}

```

## 三、主动获取Eureka注册的服务

```java
@GetMapping("/discoveryClientInfo")
public DiscoveryClient getDiscoveryClientInfo() {
  List<String> services = discoveryClient.getServices();
  for (String service : services) {
    log.info("service: {}", service);
    List<ServiceInstance> instances = discoveryClient.getInstances(service);
    for (ServiceInstance instance : instances) {
      log.info("instanceID={} instanceHost={} instancePort={}",instance.getInstanceId(),instance.getHost(), instance.getPort());
    }
  }
  return discoveryClient;
}
```

