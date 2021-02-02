---
layout: post
title: "spring cloud open feign使用详解"
date: 2021-02-02 16:42:30
categories: Spring-Cloud
keywords: "spring cloud open feign"
description: "spring-cloud-open-feign使用详解"
---

## 一、简介

​	Open Feign为微服务架构下服务之间的调用提供了解决方案。首先利用了OpenFeign的声明式方式定义Web服务客户端，其次还更进一步通过集成Ribbon实现负载均衡的HTTP客户端，而且还可以和服务降级、熔断、限流框架集成提供了发生熔断，错误情况下的处理。

## 二、Open Feign的使用

**第一步：引入依赖**

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

**第二步：主启动类配置**

```java
@SpringBootApplication
@EnableFeignClients
public class FeignOrderApplication80 {

  public static void main(String[] args) {
    SpringApplication.run(FeignOrderApplication80.class);
  }
}
```

**第三步：创建Service**

```java
// 指定服务的名称
@FeignClient(name = "CLOUD-PAYMENT-SERVICE")
public interface PaymentService {

  // 书写HTTP请求
  @GetMapping("/payment/{id}")
  CommentResult<Payment> getPaymentById(@PathVariable("id") Long id);
}
```

**Open Feign的YAML配置**

```yaml
feign:
  compression:
    request:
      # 开启请求压缩
      enabled: true
      # 希望返回的类型，默认也是以下列表
      mime-types: text/xml,application/xml,application/json
      # 请求超过此大小才进行压缩
      min-request-size: 2048
    response:
      # 是否可解压响应
      enabled: true
      # 可使用Gzip解压缩
      useGzipDecoder: true
  client:
    config:
      # 全局设置
      default:
        # 发生请求读取结果的超时时间
        read-timeout: 10000
        # 连接超时时间
        connection-timeout: 2000
      # 不同服务的超时时间设置
      CLOUD-PAYMENT-SERVICE:
        read-timeout: 10000
```

## 三、Open Feign的额外配置

### Open Feign的日志配置

YAML配置需要开启日志的包和日志等级

```yaml
feign:
  client:
    config:
      default:
        # NONE: 不开启日志(默认)
        # BASIC:记录请求方法、URL、响应状态、执行时间
        # HEADERS: 在BASIC基础上 加载请求/响应头
        # FULL: 在HEADERS基础上 增加body和请求元数据
        logger-level: FULL
        
logging:
  level:
    com.kun.springcloud.service: debug
```

### Open Feign重试机制

<span style="color:red">默认Open Feign不进行任何重试，使用`feign.Retryer.NEVER_RETRY`</span>

**第一种，直接注入相当于修改了全局配置**

```java
@Bean
public Retryer feignRetryer() {
  // fegin提供的默认实现，最大请求次数为5，初始间隔时间为100ms，下次间隔时间1.5倍递增，重试间最大间隔时间为1s，
  return new Retryer.Default(); // =>this(100, SECONDS.toMillis(1), 5);
}
```

**第二种，创建类可精确配置在不同地方**

```yaml
feign:
  client:
    config:
      default:
        retryer: com.kun.springcloud.config.SimpleRetryer
      CLOUD-PAYMENT-SERVICE:
        read-timeout: 1000
        retryer: com.kun.springcloud.config.SimpleRetryer
```



