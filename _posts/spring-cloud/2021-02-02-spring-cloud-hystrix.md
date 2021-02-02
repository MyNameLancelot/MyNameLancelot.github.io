---
layout: post
title: "spring cloud hystrix使用详解"
date: 2021-02-02 16:42:30
categories: Spring-Cloud
keywords: "spring cloud hystrix"
description: "spring-cloud-hystrix使用详解"
---

## 一、Hystrix简介

​	Hystrix是Netflix开源的一款容错框架，包含常用的容错方法：线程池隔离、信号量隔离、熔断、降级回退。在高并发访问下，系统所依赖的服务的稳定性对系统的影响非常大，依赖有很多不可控的因素，比如网络连接变慢，资源突然繁忙，暂时不可用，服务脱机等。我们要构建稳定、可靠的分布式系统，就必须要有这样一套容错方法。

### Hystrix的主要功能

**服务降级**

服务降级是指当服务器压力剧增的情况下，根据实际业务情况及流量对一些服务和页面有策略的不处理或换种简单的方式处理，从而释放服务器资源以保证核心业务正常运作或高效运作。

触发场景：程序运行异常、超时、服务熔断触发服务降级、线程池/信号量满

---

**服务熔断**

依赖的下游服务多次在一定时间类故障达到了熔断阈值，为避免引发系统崩溃系统进行熔断（不再调用下游故障服务），熔断一定时间会自动尝试恢复

触发场景：n分钟内下游出现了m次故障

<img src="/img/spring-cloud-hystrix/断路器状态扭转.png" alt="断路器状态扭转" style="zoom:80%;" />

---

**服务限流**

当集群处于高并发场景下为保证服务的可靠而进行限流操作

## 二、Hystrix使用

**第一步：引入依赖**

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

**第二步：主启动类**

```java
@SpringBootApplication
@EnableHystrix
public class PaymentApplication8007 {

  public static void main(String[] args) {
    SpringApplication.run(PaymentApplication8007.class);
  }
}
```

**第三步：如果服务消费端使用需要配置YAML**

```yaml
feign:
  hystrix:
    enabled: true
```

### Hystrix服务降级

**单个方法定制服务降价**

```java
/**
  * fallbackMethod 发生服务降级的处理方法
  */
@Override
@HystrixCommand(
  fallbackMethod = "getPaymentTimeOutById_handler", 
  commandProperties = {
    @HystrixProperty(
      name = HystrixPropertiesManager.EXECUTION_ISOLATION_THREAD_TIMEOUT_IN_MILLISECONDS,
      value = "1000")
  })
public Payment getPaymentTimeOutById(Long id) {
  int i = 10 / 0;
  ThreadUtil.sleep(id, TimeUnit.SECONDS);
  Payment payment = new Payment();
  payment.setId(id);
  payment.setSerial(UUID.randomUUID().toString(true));
  return payment;
}

public Payment getPaymentTimeOutById_handler(Long id) {
  log.info("发生服务降级 id = {}", id);
  return null;
}
```

如果是服务提供端可以将服务降级的方法放在Service实现类上，如果是消费端需要将注解标签放在Controller类上【service是interface的open feign】

**定制某个类级别全局默认降级服务**

```java
@RestController
@RequiredArgsConstructor
@Slf4j
// 配置这个类的默认降级服务方法
@DefaultProperties(defaultFallback = "globDefaultFallback")
public class OrderController {

  private final PaymentService paymentService;

  @GetMapping("/order/timeOut/{id}")
  @HystrixCommand	//这个注解不可省略
  public CommentResult getOrderTimeOut(@PathVariable("id") Long id) {
    log.info("查询订单{}", id);
    return paymentService.getPaymentByIdTimeOut(id);
  }

  // 降级服务方法，方法不需要参数，但是方法的返回值包括泛型类型要和被调用方法一致，否则保存
  private CommentResult globDefaultFallback() {
    log.error("======paymentDefaultFallback======");
    return CommentResult.failed();
  }
}
```

**OpenFeign为服务每个方法配置降级服务**

```java
/**
 * 处理服务降级的具体类
 */
@Slf4j
@Component
public class PaymentServiceFallbackImpl implements PaymentService {

  @Override
  public CommentResult<Payment> getPaymentById(Long id) {
    log.error("发生服务降级");
    return CommentResult.failed();
  }

  @Override
  public CommentResult<Payment> getPaymentByIdTimeOut(Long id) {
    log.error("发生服务降级");
    return CommentResult.failed();
  }
}

/**
  * 如果需要定制方法级别限制，需配置YAML[超时时间等也要考虑Feign设置的因素]
  * hystrix:
  *   command:
  *     PaymentService#getPaymentByIdTimeOut(Long):
  *       execution:
  *         isolation:
  *           thread:
  *             timeoutInMilliseconds: 2000
  */
// 指定服务降级的处理类
@FeignClient(name = "CLOUD-PAYMENT-SERVICE", fallback = PaymentServiceFallbackImpl.class)
public interface PaymentService {

  @GetMapping("/payment/{id}")
  CommentResult<Payment> getPaymentById(@PathVariable("id") Long id);

  @GetMapping("/payment/timeOut/{id}")
  CommentResult<Payment> getPaymentByIdTimeOut(@PathVariable("id") Long id);
}
```

### Hystrix服务熔断

熔断打开：请求不再调用当前服务，内部设置时钟一般为MTR（平均故障处理时间），当打开时长达到所设时钟则进入半熔断状态

熔断关闭：熔断关闭不会对服务进行熔断

熔断半开：部分请求根据规则调用当前服务，如果请求成功且符合规则则认为当前服务恢复正常，关闭熔断

![img](/img/spring-cloud-hystrix/a427e3ce747a4ba0b12e352ad058da78.jpeg)

```java
@Override
@HystrixCommand(fallbackMethod = "getPaymentTimeOutById_handler", commandProperties = {
  // 开启服务熔断
  @HystrixProperty(name = HystrixPropertiesManager.CIRCUIT_BREAKER_ENABLED, value = "true"),
  // 触发熔断的最小请求量，默认20
  @HystrixProperty(name = HystrixPropertiesManager.CIRCUIT_BREAKER_REQUEST_VOLUME_THRESHOLD, value = "20"),
  // 时间窗口大小，默认5s
  @HystrixProperty(name = HystrixPropertiesManager.CIRCUIT_BREAKER_SLEEP_WINDOW_IN_MILLISECONDS, value = "10000"),
  // 触发熔断的错误百分比，默认百分之50
  @HystrixProperty(name = HystrixPropertiesManager.CIRCUIT_BREAKER_ERROR_THRESHOLD_PERCENTAGE, value = "50")
})
public Payment getPaymentTimeOutById(Long id) {
  Long i = 10 / id;
  Payment payment = new Payment();
  payment.setId(id);
  payment.setSerial(UUID.randomUUID().toString(true));
  return payment;
}
```

### Hystrix参数参考

<p><span style="display:block;text-align:center;text-decoration:underline">服务降级</span></p>

**HystrixPropertiesManager.EXECUTION_ISOLATION_STRATEGY**

可选值 THREAD, SEMAPHORE,默认THREAD

**HystrixPropertiesManager.EXECUTION_ISOLATION_SEMAPHORE_MAX_CONCURRENT_REQUESTS**

信号池策略下信号池的大小，默认10

**HystrixPropertiesManager.EXECUTION_ISOLATION_THREAD_TIMEOUT_IN_MILLISECONDS**     

线程池策略下线程的超时时间，默认1s

**HystrixPropertiesManager.EXECUTION_TIMEOUT_ENABLED**

是否启动超时检查，默认true

**HystrixPropertiesManager.EXECUTION_ISOLATION_THREAD_INTERRUPT_ON_TIMEOUT**

执行超时时是否中断，默认true

**HystrixPropertiesManager.EXECUTION_ISOLATION_SEMAPHORE_MAX_CONCURRENT_REQUESTS** 

允许回调方法执行的最大并发数，默认10

**HystrixPropertiesManager.FALLBACK_ENABLED**

服务降级是否启用，默认true

---

<p><span style="display:block;text-align:center;text-decoration:underline">服务熔断</span></p>

**HystrixPropertiesManager.CIRCUIT_BREAKER_REQUEST_VOLUME_THRESHOLD**

触发熔断的最小请求量，默认20

**HystrixPropertiesManager.CIRCUIT_BREAKER_ERROR_THRESHOLD_PERCENTAGE**

触发熔断的错误百分比，默认百分之50

**HystrixPropertiesManager.CIRCUIT_BREAKER_SLEEP_WINDOW_IN_MILLISECONDS**

时间窗口大小，默认5s

**HystrixPropertiesManager.CIRCUIT_BREAKER_FORCE_OPEN**

断路器强制打开

**HystrixPropertiesManager.CIRCUIT_BREAKER_FORCE_CLOSED**

断路器强制关闭

---

<p><span style="display:block;text-align:center;text-decoration:underline">收集信息</span></p>

**HystrixPropertiesManager.METRICS_ROLLING_STATS_TIME_IN_MILLISECONDS**

滚动窗口的时间，即判断健康度持续收集的时间，默认10s

**HystrixPropertiesManager.METRICS_ROLLING_STATS_NUM_BUCKETS**

滚动时间窗桶的数量，桶内累计各指标，必须能被滚动窗口整除，默认10

**HystrixPropertiesManager.METRICS_ROLLING_PERCENTILE_ENABLED**

对命令执行的延迟是否使用百分位跟踪计算，如果为false，则返回-1

**HystrixPropertiesManager.METRICS_ROLLING_PERCENTILE_TIME_IN_MILLISECONDS**

百分位滚动窗口持续时间，默认60s

**HystrixPropertiesManager.METRICS_ROLLING_PERCENTILE_NUM_BUCKETS**

百分位统计的桶数量，默认6

**HystrixPropertiesManager.METRICS_ROLLING_PERCENTILE_BUCKET_SIZE**

百分位统计的每个桶大小，默认100

**HystrixPropertiesManager.METRICS_HEALTH_SNAPSHOT_INTERVAL_IN_MILLISECONDS**

默认0.5s

---

<p><span style="display:block;text-align:center;text-decoration:underline">其它</span></p>

**HystrixPropertiesManager.REQUEST_CACHE_ENABLED**

是否开启请求缓存，默认true

**HystrixPropertiesManager.REQUEST_LOG_ENABLED**

请求日志是否打印，默认true

---

<p><span style="display:block;text-align:center;text-decoration:underline">线程相关threadPoolProperties属性</span></p>

**HystrixPropertiesManager.CORE_SIZE**

核心线程数，默认10

**HystrixPropertiesManager.MAX_QUEUE_SIZE**

线程池阻塞队列大小，默认-1，采用SynchronousQueue，否则使用LinkedBlockingQueue

**HystrixPropertiesManager.QUEUE_SIZE_REJECTION_THRESHOLD**

设置拒绝队列阀值，通过此参数即使队列没达到最大请求也能拒绝，默认5

## 三、Hystrix 监控搭建

### 单节点监控

**第一步：引入依赖**

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
</dependency>
```

**第二步：主启动类添加`@EnableHystrixDashboard`**

**第三步：配置YAML**

```yaml
hystrix:
  dashboard:
    proxy-stream-allow-list: "localhost"
```

**第四步：访问Dashboard端的地址`ip:port/hystrix`**，填入`http://服务端ip/port/actuator/hystrix.stream`[服务端actuator中的hystrix.stream需要开放]

### Terbine集群监控

**第一步：引入依赖**

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-netflix-turbine</artifactId>
</dependency>
```

**第二步：主启动类添加`@EnableTurbine`**

**第三步：配置YAML**

```yaml
turbine:
  # 需要收集信息的服务名，即注册中心服务名称
  appConfig: CLOUD-PAYMENT-SERVICE
  aggregator:
    cluster-config: default
  # 指定集群名称
  cluster-name-expression: new String("default")
  # 同一主机上的服务通过主机名和端口号的组合来进行区分，默认以host来区分
  combine-host-port: true
```

**第四步：访问Dashboard端的地址`ip:port/hystrix`**，填入`http://[terbine-IP]:[port]/turbine.stream`
