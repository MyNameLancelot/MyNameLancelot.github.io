---
layout: post
title: "spring cloud sleuth使用详解"
date: 2021-02-02 16:42:30
categories: Spring-Cloud
keywords: "spring cloud sleuth"
description: "spring-cloud-sleuth使用详解"
---

## 一、sleuth简介

​	微服务跟踪(sleuth)其实是一个工具,它在整个分布式系统中能跟踪一个用户请求的过程(包括数据采集，数据传输，数据存储，数据分析，数据可视化)，捕获这些跟踪数据，就能构建微服务的整个调用链的视图，这是调试和监控微服务的关键工具。

| 特点              | 说明                                                         |
| ----------------- | :----------------------------------------------------------- |
| 提供链路追踪      | 通过sleuth可以很清楚的看出一个请求经过了哪些服务，可以方便的理清服务局的调用关系 |
| 性能分析          | 通过sleuth可以很方便的看出每个采集请求的耗时，分析出哪些服务调用比较耗时，当服务调用的耗时随着请求量的增大而增大时，也可以对服务的扩容提供一定的提醒作用 |
| 数据分析 优化链路 | 对于频繁地调用一个服务，或者并行地调用等，可以针对业务做一些优化措施 |
| 可视化            | 对于程序未捕获的异常，可以在zipkpin界面上看到                |

### 术语

**trace**

从客户发起请求(request)抵达被追踪系统的边界开始，到被追踪系统向客户返回响应(response)为止的整个过程

---

**span**

每个trace中会调用若干个服务，为了记录调用了哪些服务，以及每次调用的消耗时间等信息，在每次调用服务时，埋入一个调用记录

## 二、sleuth配置

[zipkin下载地址](https://zipkin.io/)，可以使用源文件编译，也可以使用脚本启动，默认访问端口9411

**第一步：引入依赖**

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-sleuth-zipkin</artifactId>
</dependency>
```

**第二步：配置yaml**

```yaml
spring:
  zipkin:
    # zipkin地址
    base-url: http://localhost:9411/
  sleuth:
    # 采样率0-1
    sampler: 1
```
