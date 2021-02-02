---
layout: post
title: "spring cloud ribbon使用详解"
date: 2021-02-02 16:42:30
categories: Spring-Cloud
keywords: "spring cloud ribbon"
description: "spring-cloud-ribbon使用详解"
---

## 一、Ribbon概述

### Ribbon简介

​	Spring Cloud Ribbon 是基于Netflix Ribbon实现的一套客户端负载均衡的工具。主要功能是提供客户端的软件负载均衡算法和服务调用。Ribbon客户端组件提供一系列完善的配置项如连接超时、重试等。简单的说就是在配置文件中列出Load Balancer后面所有的机器，Ribbon会自动的帮助你基于某种规则（如简单轮询，随机连接等）去连接这些机器。现在项目处于维护状态中，但依旧还在大范围使用

### LB负载均衡(Load Balance)

- 将用户的请求平摊到多个服务上，从而达到系统的高可用（HA）。常见的负载均衡软件有Nginx、LVS等。
- Ribbon本地负载均衡与Nginx服务端负载均衡的区别
    - Nginx是服务器负载均衡，客户端所有请求都会交给Nginx，然后由Nginx实现转发请求。即负载均衡由服务端实现。
    - Ribbon本地负载均衡在调用服务接口的时候，会在注册中心上获取注册信息服务列表之后缓存到JVM本地，从而在本地实现RPC远程服务调用技术。Ribbon属于进程内负载均衡（将LB逻辑集成到消费方，消费方从服务注册中心获知有那些地址可用，然后自己再从这些地址中选出一个合适的服务器）

## 二、集成Ribbon

<span style="color:red">如果项目中使用了eureka client、zookeeper、consul、nacos则已经自动依赖了Ribbon</span>， 然后结合RestTemplate使用即可【@LoadBalanced】

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>
```

### 使用Ribbon提供的负载均衡算法

![algorithm](/img/spring-cloud-ribbon/algorithm.png)

| 算法                      | 解释                                                         |
| ------------------------- | ------------------------------------------------------------ |
| RoundRobinRule            | 轮询，默认算法                                               |
| RandomRule                | 随机                                                         |
| RetryRule                 | 先按照RoudRobinRule的策略获取服务，如果获取服务失败则在指定时间内会进行重试，获取可用的服务 |
| WeightedResponseTimeRule  | 对RoudRobinRule的扩展，响应速度越快的实例选择权重越大，越容易被选择 |
| BestAvailableRule         | 会优过滤掉由于多次访问故障而处于断路器跳闸状态的服务，然后选择一个并发量最小的服务 |
| AvailabilityFilteringRule | 先过滤掉故障实例，再选择并发较小的实例                       |
| ZoneAvoidanceRule         | 复合判断server所在区域的性能和server的可用性选择服务器       |

主启动类加入注解@RibbonClients

```java
@RibbonClients(
    @RibbonClient(name = "CLOUD-PAYMENT-SERVICE", configuration = RandomRule.class)
)
```

## 三、自定义负载规则

**自定义某服务的规则不能被spring boot扫描到，否则定义的配置类就会被所有的Ribbon客户端使用【全局使用】，起不到特殊化定制的效果。**

自定义类实现`AbstractLoadBalancerRule`或者`IRue`即可

```java
public class MyRule extends AbstractLoadBalancerRule {

  public Server choose(ILoadBalancer lb, Object key) {
		//...自定义规则...
  }

  protected int chooseRandomInt(int serverCount) {
    return ThreadLocalRandom.current().nextInt(serverCount);
  }

  @Override
  public Server choose(Object key) {
    return choose(getLoadBalancer(), key);
  }

  @Override
  public void initWithNiwsConfig(IClientConfig clientConfig) {
  }
}
```

主启动类加入注解@RibbonClients

```java
@RibbonClients(
    @RibbonClient(name = "CLOUD-PAYMENT-SERVICE", configuration = MyRule.class)
)
```

