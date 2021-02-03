---
layout: post
title: "spring cloud alibaba sentinel使用详解"
date: 2021-02-02 16:42:30
categories: Spring-Cloud
keywords: "spring cloud alibaba sentinel"
description: "spring-cloud-alibaba-sentinel使用详解"
---

## 一、Sentinel简介

​	sentinel是面向分布式服务框架的轻量级流量控制框架,主要以流量为切入点,从流量控制,熔断降级,系统负载保护等多个维度来维护系统的稳定性。

<img src="/img/sentinel/sentinel功能.png"  style="zoom:50%;" />

### Sentinel 基本概念

**资源**

资源是 Sentinel 的关键概念。它可以是 Java 应用程序中的任何内容，例如，由应用程序提供的服务，或由应用程序调用的其它应用提供的服务，甚至可以是一段代码。只要通过 Sentinel API 定义的代码，就是资源，能够被 Sentinel 保护起来。大部分情况下，可以使用方法签名，URL，甚至服务名称作为资源名来标示资源。

---

**规则**

围绕资源的实时状态设定的规则，可以包括流量控制规则、熔断降级规则以及系统保护规则。所有规则可以动态实时调整。

## 二、Sentinel 流量控制

**针对来源**

Sentinel可以针对调用者进行限流，填写微服务名，指定对哪个微服务进行限流 ，默认default(不区分来源，全部限制)

**阈值类型**

- QPS：设置每秒能承受的请求数量
- 线程数：设置最多支持的线程数量【并非一个请求对应一个线程】

<div style="clear:both;width:100%;float: left;">
<img src="/img/sentinel/QPS.png"  style="zoom:52%;float:left" />
<img src="/img/sentinel/thread.png" style="zoom:52%;float:right" />
</div>



**流控模式**

- 直接：到达阈值时对当前资源进行限流操作
- 关联：当关联的资源接收到的请求达到了阈值上线，则对<span style="color:red">当前资源进行限流操作</span>【例如支付模块压力大时对订单模块进行限流】
- 链路：以调用链路为单位做限流，整个链路的总体流量只按照入口资源的请求量来计算【feign.sentinel.enabled: true需要打开】

<div style="clear:both;width:100%;float: left;">
<div>
<img src="/img/sentinel/direct.png"  style="zoom:52%;float:left"/>
<img src="/img/sentinel/guanlian.png" style="zoom:52%;float:left"/>
</div>
<div style="margin-top:8px">
<img src="/img/sentinel/簇点.png" style="zoom:52%;" />
</div>
</div>


使用簇点链路时可能需要展开链路关系，基本不用

```java
/**
  *  在spring-cloud-alibaba v2.1.1.RELEASE及前，sentinel1.7.0及后，关闭URL PATH聚合需要通过该方式
  *  在spring-cloud-alibaba v2.1.1.RELEASE后，可以配置关闭：spring.cloud.sentinel.web-context-unify=false
  *  手动注入Sentinel的过滤器，关闭Sentinel注入CommonFilter实例，
  *  修改配置文件中的 spring.cloud.sentinel.filter.enabled=false
  */
@Bean
public FilterRegistrationBean sentinelFilterRegistration() {
  FilterRegistrationBean registration = new FilterRegistrationBean();
  registration.setFilter(new CommonFilter());
  registration.addUrlPatterns("/*");
  // 入口资源关闭聚合
  registration.addInitParameter(CommonFilter.WEB_CONTEXT_UNIFY, "false");
  registration.setName("sentinelFilter");
  registration.setOrder(1);
  return registration;
}
```

> 链路必须使用`@SentinelResource`实现监控

**流控效果**

- 快速失败：直接抛出限流异常
- 预热模式：避免低水位服务器突然接收到大量请求宕机采用逐渐放宽限流策略，例如QPS=x，预热时长=y，冷加载因子默认为3，就是要让该资源在第y秒的时候每秒能够承受x次并发请求数量，第一次进行限流的时间点大概在x/3次请求时发生【可通过`spring.cloud.sentinel.flow.coldFactor`设置冷加载因子】
- 排队等待：匀速器模式，所有请求堆积在入口处等待，以QPS为准每秒放行响应的请求进行处理，请求间隔为（1/阈值s），可设置超时时间来过滤掉部分等待中的请求，超时时间需要小于请求的间隔才能生效

<div style="clear:both;width:100%;float: left;">
<div>
<img src="/img/sentinel/quickerror.png"  style="zoom:52%;float:left"/>
<img src="/img/sentinel/warnup.png" style="zoom:52%;float:left"/>
</div>
<div style="margin-top:8px">
<img src="/img/sentinel/queuewait.png" style="zoom:52%;" />
</div>
</div>

## 三、热点参数降流

​	热点参数限流会统计传入参数中的热点参数，并根据配置的限流阈值与模式，对包含热点参数的资源调用进行限流，热点参数限流可以看做是一种特殊的流量控制，仅对包含热点参数的资源调用生效

<img src="/img/sentinel/paramlimit.png" style="zoom:60%;" />

## 四、Sentinel 熔断降级

**降级策略**

- **慢调用比例**： 平均响应时间，当1s内持续进入N个请求，对应时刻的平均响应时间（秒级）均超过阈值，那么在接下的时间窗口之内，对这个方法的调用都会自动地熔断），注意Sentinel默认统计的RT上限是4900ms，超出此阈值的都会算作4900ms，若需要变更此上限可以通过启动配置项`-Dcsp.sentinel.statistic.max.rt=4900`来配置
- **异常比例**：是指当资源的每秒异常总数占通过量的比值超过阈值之后，资源进入降级状态，即在接下的时间窗口之内，对这个方法的调用都会自动地返回，异常比率的阈值范围是`[0.0, 1.0]`，代表`0% - 100%`
- **异常数**：是指当资源近1分钟的异常数目超过阈值之后会进行熔断，注意由于统计时间窗口是分钟级别的，若时间窗口小于 60s，则结束熔断状态后仍可能再进入熔断状态

<div style="clear:both;width:100%;float:left;">
<div>
<img src="/img/sentinel/RT.png"  style="zoom:50%;float:left"/>
<img src="/img/sentinel/exceptionnum.png" style="zoom:52%;float:left"/>
</div>
<div>
<img src="/img/sentinel/exceptionnum2.png" style="zoom:52%;" />
</div>
</div>

## 五、Sentinel 系统自适应

​	Sentinel系统自适应保护从整体维度对应用入口流量进行控制，结合应用的Load、总体平均RT、入口QPS和线程数等几个维度的监控指标，让系统的入口流量和系统的负载达到一个平衡，让系统尽可能跑在最大吞吐量的同时保证系统整体的稳定性。

<img src="/img/sentinel/systemlimit.png" style="zoom:67%;" />

**Load**：仅对Linux/Unix-like机器生效，系统的load1【`uptime`命令】作为启发指标，进行自适应系统保护，当系统load1超过设定的启发值，且系统当前的并发线程数超过估算的系统容量时才会触发系统保护，系统容量由系统的maxQps * minRt估算得出，设定参考值一般是CPU cores * 2.5

**平均RT**：当单台机器上所有入口流量的平均RT达到阈值即触发系统保护，单位是毫秒

**并发线程数：**当单台机器上所有入口流量的并发线程数达到阈值即触发系统保护

**入口 QPS：**当单台机器上所有入口流量的 QPS 达到阈值即触发系统保护

**CPU使用率**：当系统 CPU 使用率超过阈值即触发系统保护（取值范围 0.0-1.0），比较灵敏

## 六、授权规则

​	当我们需要根据调用来源来判断该次请求是否允许放行，这时候可以就使用Sentinel的来源访问控制的功能，对应的操作就是加上相应的授权规则

<img src="/img/sentinel/授权规则.png" style="zoom:60%;" />

```java
/**
 * 解析流控应用表示
 */
@Component
public class CustomRequestOriginParser implements RequestOriginParser {

  // 获取调用方标识信息并返回
  @Override
  public String parseOrigin(HttpServletRequest request) {
    String serviceName = request.getParameter("serviceName");
    StringBuffer url = request.getRequestURL();
    if (url.toString().endsWith("favicon.ico")) {
      // 浏览器会向后台请求favicon.ico图标
      return serviceName;
    }

    if (StringUtils.isEmpty(serviceName)) {
      throw new IllegalArgumentException("serviceName must not be null");
    }

    return serviceName;
  }
}
```

## 七、配置持久化

### 在Sentinel Dashboard操作并将规则写入文件

```java
/**
 * 拉模式规则持久化	
 */
public class FileDataSourceInit implements InitFunc {
  @Override	
  public void init() throws Exception {	
    // 存放路径
    String ruleDir = System.getProperty("user.home") + "/sentinel/rules";	
    String flowRulePath = ruleDir + "/flow-rule.json";	
    String degradeRulePath = ruleDir + "/degrade-rule.json";	
    String systemRulePath = ruleDir + "/system-rule.json";	
    String authorityRulePath = ruleDir + "/authority-rule.json";	
    String paramFlowRulePath = ruleDir + "/param-flow-rule.json";	

    this.mkdirIfNotExits(ruleDir);	
    this.createFileIfNotExits(flowRulePath);	
    this.createFileIfNotExits(degradeRulePath);	
    this.createFileIfNotExits(systemRulePath);	
    this.createFileIfNotExits(authorityRulePath);	
    this.createFileIfNotExits(paramFlowRulePath);	

    // 流控规则	
    ReadableDataSource<String, List<FlowRule>> flowRuleRDS = new FileRefreshableDataSource<>(
      flowRulePath,	
      flowRuleListParser	
    );	
    // 将可读数据源注册至FlowRuleManager	
    // 这样当规则文件发生变化时，就会更新规则到内存	
    FlowRuleManager.register2Property(flowRuleRDS.getProperty());
    WritableDataSource<List<FlowRule>> flowRuleWDS = new FileWritableDataSource<>(
      flowRulePath,	
      this::encodeJson	
    );	
    // 将可写数据源注册至transport模块的WritableDataSourceRegistry中	
    // 这样收到控制台推送的规则时，Sentinel会先更新到内存，然后将规则写入到文件中	
    WritableDataSourceRegistry.registerFlowDataSource(flowRuleWDS);

    // 降级规则	
    ReadableDataSource<String, List<DegradeRule>> degradeRuleRDS = new FileRefreshableDataSource<>(
      degradeRulePath,	
      degradeRuleListParser	
    );	
    DegradeRuleManager.register2Property(degradeRuleRDS.getProperty());
    WritableDataSource<List<DegradeRule>> degradeRuleWDS = new FileWritableDataSource<>(	
      degradeRulePath,	
      this::encodeJson	
    );	
    WritableDataSourceRegistry.registerDegradeDataSource(degradeRuleWDS);	

    // 系统规则	
    ReadableDataSource<String, List<SystemRule>> systemRuleRDS = new FileRefreshableDataSource<>(
      systemRulePath,	
      systemRuleListParser	
    );	
    SystemRuleManager.register2Property(systemRuleRDS.getProperty());
    WritableDataSource<List<SystemRule>> systemRuleWDS = new FileWritableDataSource<>(	
      systemRulePath,	
      this::encodeJson	
    );	
    WritableDataSourceRegistry.registerSystemDataSource(systemRuleWDS);	

    // 授权规则	
    ReadableDataSource<String, List<AuthorityRule>> authorityRuleRDS = new FileRefreshableDataSource<>(
      flowRulePath,	
      authorityRuleListParser	
    );	
    AuthorityRuleManager.register2Property(authorityRuleRDS.getProperty());
    WritableDataSource<List<AuthorityRule>> authorityRuleWDS = new FileWritableDataSource<>(	
      authorityRulePath,	
      this::encodeJson	
    );	
    WritableDataSourceRegistry.registerAuthorityDataSource(authorityRuleWDS);	

    // 热点参数规则	
    ReadableDataSource<String, List<ParamFlowRule>> paramFlowRuleRDS = new FileRefreshableDataSource<>(
      paramFlowRulePath,	
      paramFlowRuleListParser	
    );	
    ParamFlowRuleManager.register2Property(paramFlowRuleRDS.getProperty());
    WritableDataSource<List<ParamFlowRule>> paramFlowRuleWDS = new FileWritableDataSource<>(	
      paramFlowRulePath,	
      this::encodeJson	
    );	
    ModifyParamFlowRulesCommandHandler.setWritableDataSource(paramFlowRuleWDS);
  }	

  private Converter<String, List<FlowRule>> flowRuleListParser = source -> JSON.parseObject(
    source,	
    new TypeReference<List<FlowRule>>() {
    }	
  );	
  private Converter<String, List<DegradeRule>> degradeRuleListParser = source -> JSON.parseObject(	
    source,	
    new TypeReference<List<DegradeRule>>() {
    }	
  );	
  private Converter<String, List<SystemRule>> systemRuleListParser = source -> JSON.parseObject(	
    source,	
    new TypeReference<List<SystemRule>>() {	
    }	
  );	

  private Converter<String, List<AuthorityRule>> authorityRuleListParser = source -> JSON.parseObject(	
    source,	
    new TypeReference<List<AuthorityRule>>() {	
    }	
  );	

  private Converter<String, List<ParamFlowRule>> paramFlowRuleListParser = source -> JSON.parseObject(	
    source,	
    new TypeReference<List<ParamFlowRule>>() {	
    }	
  );	

  private void mkdirIfNotExits(String filePath) {
    File file = new File(filePath);
    if (!file.exists()) {	
      file.mkdirs();	
    }	
  }	

  private void createFileIfNotExits(String filePath) throws IOException {	
    File file = new File(filePath);	
    if (!file.exists()) {	
      file.createNewFile();
    }	
  }	

  private <T> String encodeJson(T t) {	
    return JSON.toJSONString(t);	
  }	
}
```

在resources下创建配置目录`META-INF/services`，然后添加文件`com.alibaba.csp.sentinel.init.InitFunc`

在文件中添加配置类全类名

### 从Nacos读取配置

引入依赖

```xml
<dependency>
  <groupId>com.alibaba.csp</groupId>
  <artifactId>sentinel-datasource-nacos</artifactId>
</dependency>
```

配置yaml

```yaml
spring:
  cloud:
    sentinel:
      datasource:
        ds:
          nacos:
            serverAddr: localhost:8848
            groupId: WYK_GROUP
            namespace: DEV
            dataId: ${spring.application.name}-sentinel
            dataType: 'json'
            ruleType: FLOW
```

创建配置文件`${spring.application.name}-sentinel`

```json
[
	{
		"clusterConfig": {
			"fallbackToLocalWhenFail": true,
			"sampleCount": 10,
			"strategy": 0,
			"thresholdType": 0,
			"windowIntervalMs": 1000
		},
		"clusterMode": false,
		"controlBehavior": 0,
		"count": 10.0,
		"grade": 1,
		"limitApp": "default",
		"maxQueueingTimeMs": 500,
		"resource": "/payment/{id}",
		"strategy": 0,
		"warmUpPeriodSec": 10
	}
]
```

## 八、自定义错误提示

```java
/**
 * 自定义sentinel报错信息
 */
@Component
public class CustomBlockExceptionHandler implements BlockExceptionHandler {

  private ObjectMapper objectMapper = new ObjectMapper();

  @Override
  public void handle(HttpServletRequest request, HttpServletResponse response, BlockException e) throws Exception {
    response.setContentType(MediaType.APPLICATION_JSON_VALUE);
    response.setCharacterEncoding(CharEncoding.UTF_8);
    PrintWriter writer = response.getWriter();
    // 触发限流
    if(e instanceof FlowException) {
      response.setStatus(2001);
      writer.print(objectMapper.writeValueAsString(CommentResult.failed("触发限流")));
    }
    // 授权规则不通过
    else if(e instanceof AuthorityException) {
      response.setStatus(2002);
      writer.print(objectMapper.writeValueAsString(CommentResult.failed("授权规则不通过")));
    }
    // 服务降级
    else if(e instanceof DegradeException) {
      response.setStatus(2003);
      writer.print(objectMapper.writeValueAsString(CommentResult.failed("服务降级")));
    }
    // 热点参数限流
    else if(e instanceof ParamFlowException) {
      response.setStatus(2004);
      writer.print(objectMapper.writeValueAsString(CommentResult.failed("触发热点限流")));
    }
    // 系统保护
    else if(e instanceof SystemBlockException) {
      response.setStatus(2005);
      writer.print(objectMapper.writeValueAsString(CommentResult.failed("触发系统保护规则")));
    }
    // 兜底
    else {
      response.setStatus(HttpStatus.INTERNAL_SERVER_ERROR.value());
      writer.print(objectMapper.writeValueAsString(CommentResult.failed("发生未知错误")));
    }
    writer.flush();
    writer.close();
  }
}
```

## 九、其它

@SentinelResource注解还有`blockHandler`、`blockHandlerClass`、`fallback`、`fallbackClass`用于限流和服务降级使用方法类似`HystrixCommand`注解

OpenFeign也可以和Sentinel配合使用`@FeignClient`中`fallback`来实现降级